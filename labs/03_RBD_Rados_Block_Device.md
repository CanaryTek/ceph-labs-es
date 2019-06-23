# RBD: Rados Block Device

RBD es el interfaz de servicio de almacenamiento en modo bloques de Ceph

## Preparación

  * En las versiones más recientes de Ceph, el pool para el servicio RBD no se crea automáticamente, así que lo primero que hay que hacer es crear el pool

        ceph osd pool create rbd 128 128

  * Podemos darle un nombre diferente, o incluso usar diferentes pools para RBD (p.ej. para tener unos volúmenes alojados en discos de rotación y otros en discos SSD). En caso de tener varios pools, podemos indicar el pool sobre el que queremos trabajar con el parámetro "-p NOMBRE_DEL_POOL"

  * Creamos un volumen

        rbd create vm01-disk0 --size=1G

  * Si listamos los objetos del pool a bajo nivel (con RADOS), veremos que se nos crean varios objetos. Algunos son de control y otros de datos

        rados ls -p rbd

  * Listamos los volúmenes RBD creados

        rbd list

  * Podemos obtener información detallada del volumen

        rbd info vm01-disk0

## Uso de volúmenes RBD desde una máquina Linux

  * Mapeamos el volumen RBD a un dispositivo de bloques

        rbd map vm01-disk0

  * Dependiendo de las versiones de Ceph y del Kernel, nos puede dar un error diciendo que algunas "features" no están soportadas por el kernel. Esto pasa porque el driver RBD esta incluido en la base de código del kernel, y puede ser más antiguo que la versión de Ceph instalada. Si nos da ese error, tendremos que desactivar esas "features" en el pool RBD

        rbd feature disable POOL_RBD FEATURE

  * Listar todos los dispositivos mapeados

        rbd showmapped

  * Crear un sistema de ficheros y montarlo

        mkfs -t ext4 /dev/rbd0
        mount /dev/rbd0 /mnt/
        df

  * Listar el uso de espacio

        rbd du

  * Si copiamos datos, podemos ver como el uso de almacenamiento aumenta (Ceph es thin-provisioning)

        dd if=/dev/zero of=/mnt/TEST.dd bs=1M count=200
        df
        rbd du

  * Si listamos los objetos del pool a bajo nivel, veremos que RBD nos esta troceando la información en muchos objetos, para hacerlos mas manejables y para aprovechar mejor las características de Ceph de distribuir la carga entre todos los nodos

        rados ls -p rbd

  * Como puede verse, hay muchos objetos con nombre rbd_data.ID.SEGMENT, donde ID identifica el volumen RBD (que podemos ver en el atributo "block_name_prefix" en la salida del comando "rbd info VOLUMEN_RBD"

## Redimensionado de volúmenes RBD

  * Podemos extender en caliente un volumen RBD

        rbd resize vm01-disk0 --size=2G

  * En la maquina que lo tiene montado, podemos verificar que el tamaño ha cambiado

        lsblk

  * Una vez aumentado, podemos redimensionar el sistema de ficheros en el cliente

        resize2fs /dev/rbd0
        df

## Utilización de snapshots RBD

  * Creamos un snapshot

        rbd snap create rbd/vm01-disk0@snap1

  * Borramos el fichero de pruebas creado en el apartado anterior

        rm /mnt/TEST.dd

  * Listamos los snapshots

        rbd snap ls rbd/vm01-disk0

  * Podemos hacer rollback del volumen RBD. OJO! para hacerlo hay que desmontar antes el volumen en el cliente

        umount /mnt
        rbd rollback rbd/vm01-disk0@snap1 

  * Volvemos a montarlo para comprobar que el fichero de test vuelve a estar ahí

        mount /dev/rbd0 /mnt
        ls /mnt

  * Borramos el snapshot

        rbd snap rm rbd/vm01-disk0@snap1

  * Si tenemos muchos snapshots, podemos borrar todos los snapshots de un volumen de golpe

        rbd snap purge rbd/vm01-disk0

## Clones de volúmenes RBD 

Podemos crear clones de lectura/escritura desde un snapshot. Los clones pueden tener sus propios snapshots e incluso clones

  * Creamos un nuevo volumen y un snapshot

```
rbd create os-base --size 2G
rbd map os-base
mkfs -t ext4 /dev/rbd0
mount /dev/rbd0 /mnt/
echo "THIS IS THE BASE OS" > /mnt/BASE_OS.txt
umount /mnt
rbd unmap os-base
rbd snap create rbd/os-base@snap1
```

  * Para poder usar un snapshot como base para crear clones, antes hay que protegerlo contra borrados accidentales

        rbd snap protect rbd/os-base@snap1

  * Ya podemos crear el clon

        rbd clone rbd/os-base@snap1 vm02-disk0

  * Si ejecutamos "rbd info" en un clon, se nos mostrara su snapshot "padre"

```
ceph-test:~ # rbd info vm02-disk0
rbd image 'vm02-disk0':
	size 2048 MB in 512 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.621c238e1f29
	format: 2
	features: layering
	flags: 
	create_timestamp: Sat Jan 20 13:29:30 2018
	parent: rbd/os-base@snap1
	overlap: 2048 MB
```

  * Podemos "aplanar" (flatten) un clon, lo que significa que Ceph copiara todos los bloques desde su snapshot padre, para que exista como un volumen independiente y ya no dependa de su "padre"

        rbd flatten vm02-disk0
        rbd info vm02-disk0 

  * Ahora "rbd info" no nos muestra ningún parent

  * Puesto que el clon ya es un volumen totalmente independiente, ahora podemos borrar el snapshot original (desprotegiéndolo antes)

        rbd snap unprotect rbd/os-base@snap1
        rbd snap rm rbd/os-base@snap1


