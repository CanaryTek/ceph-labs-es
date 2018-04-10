# Trabajar con Pools

Las "Pools" son la base del almacenamiento en Ceph. Todos los diferentes servicios de Ceph se implementan con Pools

## Operraciones basicas

  * Listar los pools (la opcion "detail" añade los detalles de los pools)

        ceph osd pool ls [detail]

  * Crear un pool con 128 Placement Groups (PG)

        ceph osd pool create test 128

  * Crear un objeto en el pool

        echo TEST > mydata
        rados -p test put test_object mydata

  * Tambien se puede hacer a traves de la entrada estandar

        echo TEST rados -p test put test_object -

  * Ver contenido de un pool

        rados -p test ls

  * Leer (descargar) un objeto

        rados -p test get test_object mydata
        cat mydata

  * Tambien se puede volcar a la salida estandar

        rados -p test get test_object -

## Snapshots

Ceph permite crear snapshots de los pools

  * Creamos un objeto de prueba

        echo OBJECT1 | rados -p test put object1 -

  * Listamos el contenido del pool

        rados ls -p test

  * Creamos un snapshot llamado "test-snap01"

        rados mksnap test-snap01 -p test

  * Listamos los snapshots para asegurarnos de que se ha creado

        rados lssnap -p test

  * Borramos el objeto de prueba

        rados rm object1 -p test

  * Listamos el contenido del pool

        rados ls -p test

  * SORPRESA! Aun vemos el objeto, porque esta referenciado por un snapshot, pero si intentamos listar el objeto no nos devolvera nada

        rados ls object1 -p test

  * Ver los objetos en los snapshots. El "overlap" debe ser cero porque solo lo tenemos en un snapshot

        rados -p test listsnaps object1

  * Hacer rollback de un snapshot (volver atras)

        rados rollback object1 test-snap01 -p test 

## Borrar un pool

  * Por seguridad, el borrado de pools esta deshabilitado por defecto. Es necesario habilitarlo en la configuracion de los MON

        ceph tell mon.* injectargs '--mon_allow_pool_delete=1'

  * Ademas, tambien por seguridad, para borrar un pool es necesario poner el nombre dos veces y una opcion "especial"

        ceph osd pool delete test test --yes-i-really-really-mean-it

  * Una vez que hemos borrado el pool, es recomendable volver a deshabilitar el borrado

        ceph tell mon.* injectargs '--mon_allow_pool_delete=0'

