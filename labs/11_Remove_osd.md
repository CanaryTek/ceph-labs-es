# Eliminar un OSD

En este lab eliminaremos un osd. Esta acción habrá que realizarla cuando un disco ha fallado y queremo eliminar el osd que tenia asociado

## Eliminar el osd

  * Podemos monitorizar el estado del cluster durante todo el proceso con

```shell
ceph -w
```

  * Identificamos el osd asociado al disco que ha fallado

```shell
ceph-admin:~ # ceph helth detail | grep down
HEALTH_WARN 1 osds down; Degraded data redundancy: 132/714 objects degraded (18.487%), 149 pgs unclean, 149 pgs degraded, 149 pgs undersized
osd.0 (root=default,host=ceph-osd3) is down
```

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

  * Eliminar la entrada de ese disco en la configuracion de DeepSea (/srv/pillar/ceph/stack/default/ceph/minions/ceph-osdX.yml)

  * Ejecutamos la fase de "limpieza" de DeepSea

```shell
salt-run state.orch ceph.stage.5
```
