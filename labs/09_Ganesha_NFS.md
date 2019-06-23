# Configurar un gateway NFS con GaneshaNFS

En este lab configuraremos una pasarela NFS y exportaremos el volumen CephFS mediante NFS

También configuraremos el servicio en alta disponibilidad, de forma que si una pasarela falla, no se pierda el servicio

## Activar el servicio

En el despliegue por defecto con ansible, no se activa la exportación de CephFS mediante NFS. Asegúrate de haber realizado el despliegue con la siguiente opción en el fichero de variables:

```shell
nfs_file_gw: true
```

Si es necesario, añade esa opción y repite el despliegue del playbook

## Preparar el sistema de ficheros a exportar

Crearemos un directorio "test" bajo la raíz de CephFS para utilizar en estas pruebas

  * Montamos CephFS en la maquina de despliegue (leer ADMIN_KEY de /etc/ceph/ceph.client.admin.keyring)

```shell
mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=ADMIN_KEY
```

  * Ya tenemos CephFS montado en /mnt
  * Creamos el directorio test

```shell
mkdir /mnt/test
```

  * Damos permiso de escritura a todo el mundo (porque por defecto tenemos root_squash activado). OJO! En producción utilizar permisos mas restrictivos

```shell
chmod 777 /mnt/test
```

## Montar en el cliente

  * Montar CephFS vía NFS

```shell
mount -t nfs 192.168.122.21:/cephfile /mnt/
```

  * Crear un fichero de prueba

```shell
echo 1 > /mnt/test/_uno.txt
```

  * Comprobar que podemos ver el fichero también en el CephFS montado en la maquina de deploy

Vale, esto ha sido fácil, pero hemos montado el sistema desde una de las 3 pasarelas, por lo que si esa pasarela falla, perderemos el servicio.
Tenemos un Punto Único de FallO (PUFO), que es lo que se supone que los sistemas como Ceph pretenden eliminar.

En la siguiente sección vamos a resolver este problema...

## NFS HA con Keepalived

Puesto que estamos exportando el mismo CephFS desde las 3 pasarelas GaneshaNFS, lo único que necesitamos es tener algún software que nos gestione la IP de servicio NFS entre las 3 pasarelas disponibles, de forma que si la pasarela activa falla, otra pasarela tome control de la IP de servicio. Para esta tarea, vamos a utilizar el software keepalived
Si queremos repartir la carga entre las 3 pasarelas, podemos tener 3 IP de servicio diferentes y repartir los clientes entre dichas IP. De esta forma, si una de las pasarelas falla, otra de las pasarelas activas tomara control de su IP de servicio y, por tanto, de sus clientes.

Vamos a crear un cluster keepalived con una "IP flotante" para el servicio NFS. Los clientes NFS utilizaran esta IP para montar el NFS. De esta forma, si la pasarela activa falla, keepalived asociará la IP flotante a otra de las pasarelas y el cliente recuperara el servicio NFS.

  * Instalar keepalived en los 3 nodos pasarela

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h 'yum -y install keepalived'; done
```

  * Crear la configuración básica de keepalived (cambiar la IP virtual si es necesario)

```shell
cat > keepalived.conf <<_EOF_
global_defs {
   router_id GANESHA-NFS-HA
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.122.20
    }
}
_EOF_
```

  * Copiar la configuración keepalived a todos los nodos NFS

```shell
for h in ceph-mon{1,2,3}; do echo $h; scp keepalived.conf $h:/etc/keepalived ; done
```

  * Arrancar y activar el servicio keepalived en los nodos NFS

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h systemctl start keepalived; ssh $h systemctl enable keepalived; done
```

  * Comprobar que la IP virtual esta activa en uno de los nodos

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h ip a; done
```

### Prueba de failover

Ahora vamos a montar el NFS usando la IP flotante de servicio y forzar un fallo de la pasarela activa
La IP flotante debería activarse en otra pasarela y, tras algunos segundos, el cliente debería recuperar la conexión NFS

  * Montar el CephFS vía NFS (usando la IP flotante)

```shell
mount -t nfs 192.168.122.20:/cephfile /mnt/
```

  * Ejecutamos este comando, que escribe en el fichero de test en un bucle (para detectar el fallo y la recuperación)

```shell
while true; do sleep 1; date | tee -a /mnt/test/_test.txt ; done
```

  * Matamos la pasarela activa (parada "a lo bestia")

  * La salida del comando anterior debería pararse mientras el servicio NFS deja de responder, y recuperarse tras un tiempo

