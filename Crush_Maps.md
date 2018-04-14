# CRUSH maps

CRUSH es el algoritmo utilizado por Ceph para decidir como y donde almacenar los datos.
El "CRUSH map" es el mapa que utiliza Ceph para tomar esas decisiones

En esta práctica modificaremos el CRUSH map para adaptarlo a nuestras necesidades

## Distribuir los datos en Racks

Por defecto, Ceph distribuye los datos a nivel de host, es decir, que intenta que las diferentes copias de cada objeto estén repartidas en diferentes servidores. 
Ceph nos ofrece diferentes niveles para distribuir la información en diferentes dominios de fallo (failure domain), por ejemplo: datacenter, rack, pdu, etc
En esta práctica vamos a modificar nuestro CRUSH map para repartir los servidores en racks y que la distribución de los objetos se haga a nivel de rack

  * Ver la topología de nuestros osd

        ceph osd tree

  * Añadimos los 3 racks que vamos a usar

```
  ceph osd crush add-bucket rack01 rack 
  ceph osd crush add-bucket rack02 rack 
  ceph osd crush add-bucket rack03 rack 
```

  * Vemos la topología, deberíamos ver los racks, pero aun sin ningún contenido

        ceph osd tree

  * Movemos los racks debajo del "bucket" root

```
  ceph osd crush move rack01 root=default
  ceph osd crush move rack02 root=default
  ceph osd crush move rack03 root=default
```

  * Movemos cada host a su rack. **OJO!** Esto puede provocar que los datos se redistribuyan, así que es recomendable moverlos de uno en uno, esperando a que el cluster se marque como "healthy" antes de mover el siguiente host (podemos ver el estado del cluster con "ceph -s")

```
  ceph osd crush move ceph-osd1 rack=rack01
  ceph osd crush move ceph-osd2 rack=rack02
  ceph osd crush move ceph-osd3 rack=rack03
  ceph osd crush move ceph-osd4 rack=rack03
```

En este punto hemos definido la "estructura" de nuestro cluster, ahora falta decirle a CRUSH que queremos distribuir los objetos a nivel de rack

  * Descargamos el crushmap a fichero

        ceph osd getcrushmap -o crushmap.bin

  * Convertimos el crushmap binario a texto para poder editarlo

        crushtool -d crushmap.bin -o crushmap.txt

  * Para distribuir los objetos a nivel de rack en lugar de a nivel de host, en la regla 0 cambiamos las siguientes líneas

        -step chooseleaf firstn 0 type host
        +step chooseleaf firstn 0 type rack

  * "Compilamos" el crushmap a binario y aplicamos los cambios

        crushtool -c crushmap.txt -o crushmap.bin
        ceph osd setcrushmap -i crushmap.bin

  * Esta última operación provocará otra redistribución de datos para adaptarse a las nuevas reglas

## Instalar discos SSD y crear un pool que se almacene solo en SSD

Si tenemos discos de rotación y SSD en el mismo cluster Ceph, necesitamos definir reglas para que los datos no se "mezclen", porque no aprovecharíamos las ventajas de los discos SSD.

En esta practica vamos a instalar discos SSD en el cluster y definiremos reglas para que los datos de ese pool solo se almacenen en SSD y del resto de pools se almacenen solo en discos de rotación

Esta vez vamos a usar una nueva funcionalidad denominada "device class". En las versiones anteriores que no soportaban los "device class" era necesario definir un bucket root diferente para los discos SSD y duplicar la estructura de los OSD debajo de este root, pero con la inclusión de los "device class" se simplifica enormemente este escenario

  * Activar la opción "noup" para mantener los osd desconectados del cluster hasta que todo este preparado (para evitar movimientos de datos innecesarios)

        ceph osd set noup

  * Añadir los discos SSD a los hosts e inicializar los osd en esos discos

  ceph-disk prepare --bluestore /dev/vdd

  * En la mayoría de los casos reales, el tipo de disco se detecta automaticamente y se le establece el "device-class" correcto (ssd, hdd, etc). También se puede forzar en el momento de inicializar el osd añadiendo al comando anterior la opcion ``--device-class ssd``

  * También podemos cambiar el device-class una vez creado el osd con los siguientes comandos (para los osd 8, 9, 10 y 11)

        $ ceph osd crush rm-device-class osd.8 osd.9 osd.10 osd.11
        done removing class of osd(s): 8,9,10,11
        $ ceph osd crush set-device-class ssd osd.8 osd.9 osd.10 osd.11
        set osd(s) 8,9,10,11 to class 'ssd'

Ahora necesitamos redefinir la regla de asignacion (placement rule) por defecto (la 0) para que solo utilice dispositivos con device-class "hdd", para que cuando activemos los osd "ssd" no se migren datos desde los pools actuales en discos "hdd" a los nuevos discos "ssd". 

Básicamente lo que tendremos es que la "placement rule" por defecto (la 0), que es la que ya esta aplicada a los pools existentes, almacenará datos solo en discos "hdd", y crearemos una nueva "placement rule" que solo utilice discos "ssd" y la asociaremos al nuevo pool. De esa forma, en los pools donde necesitemos almacenamiento rápido, solo habrá que configurarle que use la nueva regla


  * Usamos el mismo método de arriba para volcar y "decompilar" el crushmap

```
ceph osd getcrushmap -o crushmap.bin                                                                                                    
crushtool -d crushmap.bin -o crushmap.txt                                                                                               
```

  * Editamos el crushmap.txt y modificamos la regla por defecto (id 0) para que solo utilice discos "hdd", cambiando las siguientes líneas

```
  -step take default
  +step take default class hdd
```

  * Compilamos y aplicamos los cambios

```
crushtool -c crushmap.txt -o crushmap.bin
ceph osd setcrushmap -i crushmap.bin
```

  * Cuando todos los nuevos osd que hemos definidos estén preparados, desactivamos la opción "nohup" para que se activen los nuevos osd

        ceph osd unset nohup

  * Si hemos hecho todo bien, no deberíamos ver movimiento de datos, porque los pools que ya estaban creados estarán usando la regla 0, a la que le hemos dicho que solo use discos "hdd"

  * Ahora crearemos una nueva "placemen rule" llamada "ssd-disks" para que solo utilice discos SSD. 

        ceph osd crush rule create-replicated ssd-disks default host ssd

  * El comando anterior crea la regla llamada "ssd-disks" del tipo "replicated" (es decir, que se guardan varias copias de cada objeto), que utiliza el root bucket "default", distribuye la información a nivel de "host" y utiliza dispositivos con "device-class" "ssd"
    * En Ceph existen 2 tipos de asignación de espacio en los pools: "replicted" que almacena varias copias de cada objeto, y "erasure-code" que segmenta los objetos y almacena información de redundancia, de manera similar a RAID
    * Como puede intuirse en este comando, la practica anterior se podría haber hecho definiendo una nueva regla indicándole que distribuya datos a nivel de rack, y asignando esta nueva regla a los pools, pero hemos querido mostrar como se modifica el crushmap a bajo nivel

  * Si queremos distribuir los datos a nivel de rack (como en la practica anterior), definiriamos la regla sustituyendo "host" por "rack"

        ceph osd crush rule create-replicated ssd-disks default rack ssd

  * Ahora creamos un nuevo pool y le asignamos la nueva regla.

        ceph osd pool create fast-pool 128 128 replicated ssd-disks

  * Verificamos que el nuevo pool usa una regla diferente a las demas

        ceph osd pool ls detail

Ahora podemos almacenar datos en el nuevo pool y verificar que se ha almacenado exclusivamente en los nuevos osd "ssd"

  * Creamos un objeto en el nuevo pool

        echo TEST | rados put object1 - -p fast-pool

  * Verificamos que se ha creado

        rados ls -p fast-pool

  * Comprobamos la ubicación del objeto (si hemos hecho todo bien, debería estar ubicado exclusivamente en los nuevos osd que hemos añadido en esta práctica)

        # ceph osd map fast-pool object1
        osdmap e869 pool 'fast-pool' (11) object 'object1' -> pg 11.bac5debc (11.3c) -> up ([9,8,11], p9) acting ([9,8,11], p9)

