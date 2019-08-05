# Añadir un OSD

En este lab añadiremos un nuevo osd al cluster

Partimos del supuesto de que hemos añadido un nuevo disco a un nodo (nuevo o en sustitucion de uno dañado), y queremos añadirlo al cluster.

**¡OJO!**: Este procedimiento utiliza el descubrimiento automático de discos, y solo puede ser usado si no hemos hecho modificaciones en las configuraciones

## Añadir el osd

  * Para que nos genere nuevos proposals en las maquinas donde se han instalado nuevos  discos, tenemos que borrar los proposals que ya existan en /srv/pillar/ceph/proposals/profile-default/stack/default/ceph/minions/

  * Podemos monitorizar el estado del cluster durante todo el proceso con

```shell
ceph -w
```

Ejecutar stage 1,2 y 3


  * Como medida de protección, en caso de que el disco a añadir contenga datos reconocibles, no se añadira al custer. Asi que el primer paso será inicializarlo para eliminar particiones y datos reconocibles
  * En la maquina donde tenemos el nuevo disco, lo inicializamos con: (asumimos que es el disco /dev/vdd)

```shell
ceph-disk zap /dev/vdd
```

  * Podemos monitorizar los eventos de DeepSea con

```shell
deepsea monitor
```

  * Ejecutamos las fases 1 y 2. Se nos deberian generar nuevos proposals para las maquinas donde hemos instalado nuevos discos

```shell
salt-run state.orch ceph.stage.1
salt-run state.orch ceph.stage.2
```

  * Ejecutar la fase 3 para que se creen los nuevos osd y se añadan al cluster

```shell
salt-run state.orch ceph.stage.3
```
