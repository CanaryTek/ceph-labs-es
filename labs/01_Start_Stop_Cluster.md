# Inicio y parada completa del cluster

Operativa para realizar una parada controlada completa del cluster y posterior arranque del mismo

**NOTA:** Aunque el procedimiento descrito es extensible a cualquier cluster Ceph, se incluyen algunos comandos que utilizan "salt", que es el sistema de despliegue de SuSE Enterprise Storage. En caso de usar otro sistema de despliegue, deberan utilizarse comandos equivalentes

## Parada controlada del cluster

  * Antes de iniciar la parada, verificar que el cluster esta en estado HEALTH_OK. Si no lo esta, resolver el problema y esperar a que este en estado HEALTH_OK

```
ceph status
```

  * Si no lo esta, investigar la causa, resolver el problema y esperar a que este en estado HEALTH_OK

```
ceph health detail
```

  * Verificar que todos los nodos del cluster estan activos

```
salt "*" test.ping
```

  * Si no responden todos, investigar y resolver la causa

  * Pues que vamos a parar los OSD, debemos indicar al cluster que no debe responder a estos eventos de fallo. Lo hacemos activando las siguientes flags

```
ceph osd set noout
ceph osd set nobackfill
ceph osd set norecover
```

  * Verificar que los flags estan activos

```
ceph status
```

  * Asegurarse de que los clientes de almacenamiento estan desconectados
  * Parar las pasarelas de servicio (radosgw, iscsi, nfs-ganesha)
  * Parar los nodos OSD

```
salt "ceph-osd*" cmd.run "halt -p"
```

  * Verificar que los osd no responden

```
salt "*" test.ping
```

  * Parar los nodos MON

```
salt "ceph-mon*" cmd.run "halt -p"
```

  * Parar el nodo "admin" (si es necesario)

## Arranque tras parada controlada del cluster

  * Arrancar nodo admin
  * Arrancar nodos MON y OSD. El órden no es critico, pero si arrancamos los MON antes del damos tiempo a establecer quorum mientras arrancan los OSD
  * Desde el admin, esperar a que respondan todos los nodos

```
salt "*" test.ping
```

  * Verificar el estado del cluster, logicamente no estara en estado HEALTH_OK

```
ceph status
```

  * Desactivamos los flags

```
ceph osd unset noout
ceph osd unset nobackfill
ceph osd unset norecover
```

  * Al poco tiempo, el cluster deberia recuperarse y recuperar el estado HEALTH_OK

```
ceph status
```

### Incidencias habituales

  * La incidencia más habitual es la falta de sincronización de relojes entre los MON. Podemos forzar a que se sincronicen con (asumiendo que el servidor NTP es "ceph-admin"): 

```
salt "ceph-mon*" cmd.run "systemctl stop ntpd ; ntpdate -dv ceph-admin; systemctl start ntpd"
```

