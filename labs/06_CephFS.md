# CephFS

En esta práctica vamos a trabajar con el almacenamiento en sistema de ficheros de CephFS

## Montar el CephFS desde otra máquina

  * El montaje de CephFS es parecido al de NFS, hay que indicar el MON (o la lista de MONs separados por comas), y el filesystem. Ademas, hay que especificar el usuario y el secreto para validar la conexión (puede leerse del keyring)

```
mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==
```

  * Si tenemos varios CephFS, podemos indicar cual queremos montar con la opción "mds_namespace". **OJO** el soporte de varios CephFS en el mismo aun no se considera estable

```
mount -t ceph 192.168.122.21,192.168.122.22,192.168.122.23:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==,mds_namespace=fs2
```

## Snapshots

CephFS soporta snapshots a nivel de sistema de ficheros. Los snapshots se gestionan con los comandos básicos de gestion de directorios (ls, mkdir, rmdir, etc) sobre un directorio oculto llamado ".snap"

  * En la version actual de Ceph (luminous), los snapshots aún se consideran una funcionalidad en beta, así que no vienen activados por defecto. Los activamos:

```
ceph fs set cephfs allow_new_snaps true --yes-i-really-mean-it
```

  * Crear un snapshot llamado 20180409

```
mkdir /mnt/.snap/20180409
```

  * Listamos los snapshots

```
ls /mnt/.snap
```

  * Listamos el contenido del snapshot que hemos creado

```
ls /mnt/.snap/20180409
```

  * Borramos el snapshot

```
rmdir /mnt/.snap/20180409
```

