# Ubicación de datos en Ceph

En esta práctica vamos a ver un ejemplo práctico de cómo ubica Ceph los datos en los diferentes Placement Groups (PG)
Crearemos un pool y un objeto y veremos cómo se ha almacenado dicho objeto

  * Creamos un pool replicado con 128 PG

```
ceph osd pool create data_test 128 128
```

  * Creamos un objeto con datos de ejemplo

```
echo "Ceph data test" > ceph-test.txt
rados -p data_test put ceph-test ceph-test.txt
```

  * Verificamos el objeto que acabamos de crear

```
rados -p data_test ls
rados -p data_test get ceph-test -
```

  * Consultamos la ubicación de dicho objeto

```
# ceph osd map data_test ceph-test
osdmap e209 pool 'data_test' (8) object 'ceph-test' -> pg 8.e18db830 (8.30) -> up ([0,7,5], p0) acting ([0,7,5], p0)
```

  * En este ejemplo, el objeto esta almacenado en el PG 8.30 (pg 30 del pool 8), que esta localizado en los osd 0, 7 y 5. Y el osd primario es el 0

  * Un detalle interesante del comando anterior es que realmente no es necesario que el objeto por el que preguntamos exista. Hay que recordar que la ubicación de objetos en CRUSH depende del pool en el que esté, y del nombre del objeto. En caso de ejecutar el comando anterior con un objeto que no exista, lo que nos devuelve es **donde se almacenaría** un objeto que tuviera ese nombre y estuviera en ese pool. Por ejemplo:

```
# ceph osd map test objeto_que_no_existe
osdmap e869 pool 'test' (12) object 'objeto_que_no_existe' -> pg 12.155e0aa9 (12.29) -> up ([3,4,5], p3) acting ([3,4,5], p3)
```
