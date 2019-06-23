# CephFS

En esta práctica vamos a trabajar con el almacenamiento en sistema de ficheros de CephFS

## Montar el CephFS desde otra máquina

  * El montaje de CephFS es parecido al de NFS, hay que indicar el MON (o la lista de MONs separados por comas), y el filesystem. Ademas, hay que especificar el usuario y el secreto para validar la conexión (puede leerse del keyring)

```
mount -t ceph ceph-mon1,ceph-mon2,ceph-mon3:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==
```

  * Si tenemos varios CephFS, podemos indicar cual queremos montar con la opción "mds_namespace". **OJO** el soporte de varios CephFS en el mismo aun no se considera estable

```
mount -t ceph ceph-mon1,ceph-mon2,ceph-mon3:/ /mnt -oname=admin,secret=AQAaeFxaAAAAABAAFQK/pP6+QHQqc2ATztlx7A==,mds_namespace=fs2
```

## Snapshots

CephFS soporta snapshots a nivel de sistema de ficheros. Los snapshots se gestionan con los comandos básicos de gestion de directorios (ls, mkdir, rmdir, etc) sobre un directorio oculto llamado ".snap"

  * Creamos datos para las pruebas

```
mkdir -p /mnt/dir1
mkdir -p /mnt/dir2
echo before_snap > /mnt/dir1/file1
echo before_snap > /mnt/dir2/file2
```

  * Creamos un snapshot

```
mkdir /mnt/.snap/snap_root
```

  * Nos da error "Operation not permitted"?
  * En la version actual de Ceph (luminous), los snapshots aún se consideran una funcionalidad en beta, así que no vienen activados por defecto. Los activamos:

```
ceph fs set cephfs allow_new_snaps true --yes-i-really-mean-it
```

  * Volvemos a intentarlo

```
mkdir /mnt/.snap/snap_root
```

  * Creamos otro snapshot de dir1

```
mkdir /mnt/dir1/.snap/snap_dir1
```

  * Listamos los snapshots

```
ls /mnt/.snap
ls /mnt/dir1/.snap
```

  * Listamos el contenido del snapshot que hemos creado

```
ls /mnt/.snap/snap_root
ls /mnt/dir1/.snap/snap_dir1/
```

  * En /mnt/dir1/.snap veremos una referencia al snapshot snap_root, pero no podremos listar su contenido

  * Modificamos datos

```
echo after_snap1 > /mnt/dir1/file1
echo after_snap1 > /mnt/dir2/file2
```

  * Creamos otro snapshot de dir2 y moduficamos datos

```
mkdir /mnt/dir1/.snap/snap2_dir1
echo after_snap2 > /mnt/dir1/file1
echo after_snap2 > /mnt/dir2/file2
```

  * El snapshot solo afecta a los datos "por debajo" de ese snapshot

  * Podemos acceder a los datos antiguos a traves de la ruta del snapshot

```
cat /mnt/dir1/file1
cat /mnt/dir2/file2
cat /mnt/.snap/snap_root/dir1/file1
cat /mnt/.snap/snap_root/dir2/file2
cat /mnt/dir1/.snap/snap_dir1/file1
cat /mnt/dir1/.snap/snap2_dir1/file1
```

  * Borramos los snapshots

```
rmdir /mnt/.snap/snap_root
rmdir /mnt/dir1/.snap/snap_dir1
rmdir /mnt/dir1/.snap/snap2_dir1
```

