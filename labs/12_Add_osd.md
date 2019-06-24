# Añadir un OSD

En este lab añadiremos un nuevo osd al cluster

Partimos del supuesto de que hemos añadido un nuevo disco a un nodo (nuevo o en sustitucion de uno dañado), y queremos añadirlo al cluster.

## Añadir el osd

  * Podemos monitorizar el estado del cluster durante todo el proceso con

```shell
ceph -w
```

  * Como medida de protección, en caso de que el disco a añadir contenga datos reconocibles, no se añadira al custer. Asi que el primer paso será inicializarlo para eliminar particiones y datos reconocibles
  * En la maquina donde tenemos el nuevo disco, lo inicializamos con: (asumimos que es el disco /dev/vdd)

```shell
ceph-disk zap /dev/vdd
```

  * Podemos monitorizar los eventos de DeepSea con

```shell
deepsea monitor
```

  * Ejecutamos la fase de descubrimiento de discos

```shell
salt-run state.orch ceph.stage.1

```

  * En la propuesta "proposal" de esa maquina, nos deberia aparecer el nuevo disco

  * El osd que ha fallado es el 0, lo eliminamos


```shell
salt-run remove.osd 0
[ERROR   ] ('Safety is not disengaged...refusing to remove OSD', ' run "salt-run disengage.safety" first THIS WILL CAUSE DATA LOSS.')
```

  * Como mecanismo de seguridad, DeepSea implementa un mecanismo de bloqueo para evitar borrados accidentales. Tenemos que desactivar esa proteccion

```shell
salt-run disengage.safety
```

  * Ahora si nos dejara eliminar el osd

```shell
salt-run remove.osd 0
[ERROR   ] ('Safety is not disengaged...refusing to remove OSD', ' run "salt-run disengage.safety" first THIS WILL CAUSE DATA LOSS.')
```

  * No es necesario volver a activar la protección, porque se activa automaticamente pasados unos minutos

  * Si ahora miramos el arbol de OSD veremos que ya no aparece el 0, y el cluster esta en HEALTH_OK

```shell
ceph -s
ceph osd tree
```
