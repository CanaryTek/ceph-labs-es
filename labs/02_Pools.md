# Trabajar con Pools

Las "Pools" son la base del almacenamiento en Ceph. Todos los diferentes servicios de Ceph se implementan con Pools

## Operaciones básicas

  * Listar los pools (la opcion "detail" añade los detalles de los pools)

```
ceph osd pool ls [detail]
```

  * Crear un pool con 128 Placement Groups (PG)

```
ceph osd pool create test 128
```

  * Crear un objeto en el pool

```
echo TEST > mydata
rados -p test put test_object mydata
```

  * Tambien se puede hacer a traves de la entrada estandar

```
echo TEST | rados -p test put test_object -
```

  * Ver contenido de un pool

```
rados -p test ls
```

  * Leer (descargar) un objeto

```
rados -p test get test_object mydata
cat mydata
```

  * Tambien se puede volcar a la salida estandar

```
rados -p test get test_object -
```

  * Agregar datos a un objeto (append)

```
echo uno | rados -p test append test_append -
echo dos | rados -p test append test_append -
echo tres | rados -p test append test_append -
rados -p test get test_append -
```

  * Eliminar objetos

```
rados -p test rm test_object
```

## Snapshots

Ceph permite crear snapshots de los pools

  * Creamos objetos de prueba

```
echo before snap | rados -p test put test_snap1 -
echo before snap | rados -p test put test_snap2 -
```

  * Listamos el contenido del pool

```
rados -p test ls
```

  * Creamos un snapshot llamado "test-snap01"

```
rados -p test mksnap test-snap01
```

  * Listamos los snapshots para asegurarnos de que se ha creado

```
rados -p test lssnap
```

  * Borramos un objeto de prueba y modificamos otro

```
rados -p test rm test_snap1
echo after snap | rados -p test put test_snap2 -
```

  * Listamos el contenido del pool

```
rados -p test ls
```

  * SORPRESA! Aun vemos el objeto borrado, porque esta referenciado por un snapshot. Pero si intentamos leer el objeto nos dara error

```
rados -p test get test_snap1 -
```

  * Ver los snapshots en los que aparece un objeto

```
rados -p test listsnaps test_snap1
rados -p test listsnaps test_snap2
```

  * Leer un objeto en un snapshot determinado

```
rados -p test -s test-snap01 get test_snap1 -
rados -p test -s test-snap01 get test_snap2 -
```

  * Hacer rollback de un snapshot (volver atras)

```
rados -p test rollback test_snap2 test-snap01
```

  * Eliminar snapshot

```
rados -p test rmsnap test-snap01
```

## Metadata

RADOS permite asignar metadatos en formato clave/valor a los objetos

  * Creamos un objeto

```
echo "test matadata" | rados -p test put test_metadata -
```

  * Asignamos metadata

```
rados -p test setomapval test_metadata key1 valor1
rados -p test setomapval test_metadata key2 valor2
```

  * Listar metadatos

```
rados -p test listomapvals test_metadata
```

  * Obtener un valor determinado

```
rados -p test getomapval test_metadata key1
```

  * Eliminar una clave

```
rados -p test rmomapkey test_metadata key2
```

  * Cuando leemos un objeto a fichero perdemos sus metadatos

```
rados -p test listomapvals test_metadata
rados -p test get test_metadata - | rados -p test put test_metadata2 -
rados -p test listomapvals test_metadata2
```

  * Copiar el objeto con rados conserva los metadatos

```
rados -p test listomapvals test_metadata3
rados -p test cp test_metadata test_metadata3
rados -p test listomapvals test_metadata3
```

## Exportar e importar

  * Podemos "serializar" el contenido de un pool completo a un fichero
 
```
rados -p test export test.rados_export
```

  * Esto nos creara un fichero test.rados_export con todos los datos del pool

  * Podemos importar ese contenido a otro pool con

```
rados mkpool test_import
rados -p test_import import test.rados_export
```

  * Podemos hacer el export a la salida estandar y comprimirlo

```
rados export -p test - | gzip > test.rados_export.gz
```

  * Podemos importar el export comprimido con

```
gunzip -c test.export.gz | rados import -p fast-pool -
```

  * Incluso podemos hacerlo con un pipe, sin fichero intermedio

```
rados mkpool test_import2
rados export -p test - | rados -p test_import2 import -
```

  * El proceso export/import conserva los metadatos de los objetos

## Borrar un pool

  * Por seguridad, el borrado de pools esta deshabilitado por defecto. Es necesario habilitarlo en la configuracion de los MON

```
ceph tell mon.* injectargs '--mon_allow_pool_delete=1'
```

  * Ademas, tambien por seguridad, para borrar un pool es necesario poner el nombre dos veces y una opcion "especial"

```
ceph osd pool delete test test --yes-i-really-really-mean-it
```

  * Una vez que hemos borrado el pool, es recomendable volver a deshabilitar el borrado

```
ceph tell mon.* injectargs '--mon_allow_pool_delete=0'
```

