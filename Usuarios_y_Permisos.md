# Usuarios y permisos

En esta práctica vamos a ver cómo podemos crear diferentes usuarios y permitirles el acceso únicamente a algunos servicios y/o pools

# Gestión de usuarios

  * Listar los usuarios y permisos definidos en el cluster

```
ceph auth list
```

  * Crear pools test y test2

```
ceph osd pool create test 128
ceph osd pool create test2 128
```

  * Crear un usuario que solo podrá acceder al pool "test"

```
ceph auth get-or-create client.test mon 'allow r' osd 'allow rw pool=test' > ceph.client.test.keyring
```

  * El comando anterior volcará el usuario y su clave en el fichero "ceph.client.test.keyring"

  * Copiar el fichero "ceph.client.test.keyring" al directorio /etc/ceph (o cualquier otro) de la maquina remota desde donde vayamos a acceder

## Acceso a pools con rados

Esta sección debe realizarse desde la maquina remota donde hemos copiado el fichero keyring de la sección anterior

  * Listar los pools

```
rados lspools
```

  * El comando anterior fallará porque tenemos que decirle qué usuario y clave queremos usar para validarnos al cluster Ceph. Por defecto de usa "admin" y se buscan varios ficheros keyring en /etc/ceph
  * Probamos otra vez especificando el usuario y el fichero keyring

```
rados -k ceph.client.test.keyring --id test lspools
```

  * Puesto que especificar el usuario y keyring en cada comando es un engorro, podemos especificarlos en una variable de entorno

```
export CEPH_ARGS="-k ceph.client.test.keyring --id test"
rados lspools
```

  * Listamos los objetos en el pool

```
rados ls -p test
```

  * Escribimos algunos objetos en el pool test

```
rados put services /etc/services -p test
rados put kk1 /etc/services -p test
rados put kk2 /etc/services -p test
rados put kk3 /etc/services -p test
```

  * Verificamos que los objetos se han creado

```
rados ls -p test
```

  * Listamos los objetos del pool "test2"

```
rados ls -p test22
```

  * Debería darnos error porque el usuario que estamos usando "test" solo tiene permisos sobre el pool "test"

### Control de acceso por nombre del objeto (object_prefix)

Puesto que tener muchos pools implica una gran carga al cluster Ceph, no pueden ser el único mecanismo de control de acceso. Podemos establecer control de acceso basándonos en un prefijo de nombre del objeto (por ejemplo, dar acceso a todos los objetos cuyo nombre empiece por "test_")

  * Creamos un usuario test2 con acceso a pool test, pero solo a los objetos cuyo nombre empiecen por "kk"

```
ceph auth get-or-create client.test2 mon "allow *" osd "allow rw pool=test object_prefix kk"
```

  * Desde el host cliente, intentamos descargar objetos desde el pool test

```
rados -k ceph.client.test2.keyring --id test2 -p test get kk1 kk1
rados -k ceph.client.test2.keyring --id test2 -p test get kk2 kk2
rados -k ceph.client.test2.keyring --id test2 -p test get services services
```

  * Solo debería permitirnos descargar los objetos que empiezan por "kk" (kk1 y kk2)

### Control de acceso por namespace

Puesto que la granularidad de acceso por pool no es escalable y la basada en object_prefix es poco flexible, en las versiones recientes de Ceph se ha introducido el concepto de "namespace" que son espacios de nombres separados dentro de un mismo pool. Cada namespace es independiente de los demas, puede contener objetos que tengan el mismo nombre que los de otros namespaces y, sobre todo, permiten establecer control de acceso a nivel de namespace

  * Modificamos el usuario test2 para que solo tenga acceso al namespace "nm1" en el pool test

```
ceph auth get-or-create client.test2 mon "allow *" osd "allow rw pool=test namespace=nm1"
```

  * Con el usuario administrador, escribimos varios objetos en el namespace nm1 del pool test

```
rados put kk1 /etc/services -p test -N nm1
rados put kk2 /etc/services -p test -N nm1
```

  * Desde la maquina cliente Ceph, intentamos listar con el usuario test2 los diferentes namespaces del pool test

```
rados -k ceph.client.test2.keyring --id test2 -p test -N nm1 ls
rados -k ceph.client.test2.keyring --id test2 -p test ls
```

  * Solo debería permitirnos listar los objetos del namespace "nm1"

## Control de acceso a RBD (Rados Block Device)

Cuando accedemos a almacenamiento de bloques (RBD) necesitamos poder definir permisos de acceso a nivel de dispositivo RBD
Vamos a ver cómo podemos hacerlo

  * Como usuario admin, creamos una imagen rbd

```
rbd create testvm01-disk01 -s 2G
```

  * Puesto que ya hemos visto que todos los objetos de bajo nivel de una imagen RBD tienen el mismo prefijo, podemos aplicar control de acceso por "object_prefix". Consultamos el object_prefix de la imagen que acabamos de crear

```
# rbd info testvm01-disk01
rbd image 'testvm01-disk01':
	size 2048 MB in 512 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.14a30643c9869
	format: 2
	features: layering
	flags: 
	create_timestamp: Sat Jan 27 20:26:32 2018

```

  * En este ejemplo, los objetos de esta imagen tendrán nombres que empiezan por "rbd_data.149ef74b0dc51". Necesitamos dar acceso a los datos y headers de esa imagen

```
ceph auth caps client.test mon "allow r" osd 'allow rx pool=rbd, allow rwx object_prefix rbd_header.14a30643c9869, allow rwx object_prefix rbd_data.14a30643c9869'
```

  * Con esos permisos, el usuario "test" podrá listar las imagenes, pero solo podrá montar la imagen "testvm01-disk01"

  * Por supuesto, podríamos definir multiples pools rbd y gestionar el control de accesos a nivel de pool, pero como ya se ha comentado, el tener muchos pools no escala bien

## Control de acceso en CephFS

En CephFS también podemos definir permisos de acceso de forma que los usuarios solo tengan acceso a un sub-árbol determinado del CepFS 

  * Montamos el cepfs como usuario admin para crear los subdirectorios (obtener el secret del keyring de admin)

```
mount -t ceph ceph-mon1:/ /mnt/ -oname=admin,secret=AQDBOGZai+YkAxAAEeUbkgyqCR1a4X9dZmNseQ==
```

  * Creamos varios subdirectorios

```
mkdir /mnt/test-home /mnt/private1 /mnt/private2
```

  * Permitimos al usuario test acceso al pool cephfs_data, pero limitando el subdirectorio en los permisos del servicio mds. **NOTA:** En kernels anteriores al 4.x hay que dar tambien acceso de lectura al directorio "/" para que se pueda montar, lo que hace casi inviable el tener subdirectorios privados

```
ceph auth caps client.test mon "allow r" osd 'allow * pool=cephfs_data' mds "allow * path=/test-home"
```

  * En versiones recientes de ceph se puede usar el comando "ceph fs authorize"

```
ceph fs authorize cephfs client.test /test-home rw
```

  * Desde la maquina remota, montamos el CephFS como usuario test

```
mount -t ceph ceph-mon1:/ /mnt/ -oname=test,secret=AQD0wmxawfKqFBAAhbRnIACauIOgYZ4nNAHAWg==
```

  * Deberíamos tener acceso únicamente a /mnt/test-home

  * También puede montarse únicamente el directorio al que tenemos acceso

```
mount -t ceph ceph-mon1:/test-home /mnt/ -oname=test,secret=AQD0wmxawfKqFBAAhbRnIACauIOgYZ4nNAHAWg==
```

