# Ubicación de datos en Ceph

En esta práctica vamos a ver un ejemplo práctico de cómo ubica Ceph los datos en los diferentes Placement Groups (PG)
Crearemos un pool y un objeto y veremos cómo se ha almacenado dicho objeto

  * Creamos un pool replicado con 128 PG

        ceph osd pool create data_test 128 128

  * Creamos un objeto con datos de ejemplo

        echo "Ceph data test" > ceph-test.txt
        rados -p data_test put ceph-test ceph-test.txt

  * Verificamos el objeto que acabamos de crear

        rados -p data_test ls
        rados -p data_test get ceph-test -

  * Consultamos la ubicación de dicho objeto

        ceph-deploy:~ # ceph osd map data_test ceph-test
        osdmap e209 pool 'data_test' (8) object 'ceph-test' -> pg 8.e18db830 (8.30) -> up ([0,7,5], p0) acting ([0,7,5], p0)

  * En este ejemplo, el objeto esta almacenado en el PG 8.30 (pg 30 del pool 8), que esta localizado en los osd 0, 7 y 5. Y el osd primario es el 0

