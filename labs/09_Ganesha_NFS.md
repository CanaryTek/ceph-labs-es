# Configurar un gateway NFS con GaneshaNFS

En este lab configuraremos una pasarela NFS y exportaremos el volumen CephFS mediante NFS

También configuraremos el servicio en alta disponibilidad, de forma que si una pasarela falla, no se pierda el servicio

## Preparar el sistema de ficheros a exportar

Crearemos un directorio "test" bajo la raíz de CephFS para utilizar en estas pruebas

  * Montamos CephFS en la maquina de despliegue (leer ADMIN_KEY de /etc/ceph/ceph.client.admin.keyring)

```shell
mount -t ceph ceph-mon1,ceph-mon2.ceph-mon3:/ /mnt -oname=admin,secret=ADMIN_KEY
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
mount -t nfs ceph-mon1:/cephfs /mnt/
```

  * Crear un fichero de prueba

```shell
echo 1 > /mnt/test/_uno.txt
```

  * Comprobar que podemos ver el fichero también en el CephFS montado en la maquina ceph-admin

Vale, esto ha sido fácil, pero hemos montado el sistema desde una de las 3 pasarelas, por lo que si esa pasarela falla, perderemos el servicio.
Tenemos un Punto Único de FallO (PUFO), que es lo que se supone que los sistemas como Ceph pretenden eliminar.

