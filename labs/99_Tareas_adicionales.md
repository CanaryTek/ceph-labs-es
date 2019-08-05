# Tareas adicionales

En esta sección se describen algunas tareas de administración adicionales útiles para la gestión del cluster

## Instalacion de claves SSH

Una opción interesante es automatizar la instalación de claves de acceso SSH en todos los nodos del cluster.

Para hacerlo, vamos a definir una variable en el "pillar" que contenga todas las claves publicas SSH con acceso a los nodos, y un estado para instalar dichas claves

### Preparacion del "pillar"

  * En el fichero /srv/pillar/local/ssh-keys.sls definimos una variable con las claves SSH a instalar en los nodos

```shell
# /srv/pillar/local/ssh-keys.sls
ceph-ssh-keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDBslD3aryvHxSdxmO05oJf/tLQsj9EFdj78w1kfKKji3DtQV32rww3nTboHYeOZ93RmAqfhmgj0YAOGcejsiAvoatBqV29/QCsjwJrhDUPRhk7OXbEEWThcG69xQUnpkE3KzpbbgqstQgVogXMsHWuuanGvqGeH2FZcYlL43mi75fsWcUlrkCFmYPLv2LfyjLvAr8OFjkUnCNGkkmxSGm2rGWSJ7q0jf6ZpFYhOBsBotRKVWL4O8WHlhhvCJuRluTvszYmdOuxaMAuAhQlMRyw+RSjxSaQOtQirHeo7WEhyyMNabT8g4RCr8nkp71PH20NcPtw7kKBzo837qwzTEw9 ssh-key-1
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCfZtCo6K7QGpuDaVTOBFxQ7BmcmQsPy00dm8dfAF480PObgfj0FgnPmtRSP/jjl1ezQ357zo0+feWTbS3kQQNpelAxkAs0hrOVXqwHJ+wwQtHB3ZsQeg/4WWzFoX9Nr0PjUnuFLzuHH9FKQo1KEC2Fj/7/BwQP7o4r5nkyiR5ZY5JX2elsLv1tzX0DgnS1NE1Nk5tPg0q9xxY4aOPL2t7a44NLdTPhUlokLNNhbmK7Pet1NqIQE91uVz3esNm2tbVCwfhTyenIlzTNMG9zh8QGIC+pm9rYVKr1N6lNp6KsLEpXCTHkEMrKagS1ymyW2xUXEHUljX+ksiW8Vsau9rud ssh-key-2

```

  * En el fichero /srv/pillar/local/init.sls, incluimos el fichero anterior

```shell
# /srv/pillar/local/init.sls
{% include 'local/ssh-keys.sls' ignore missing %}
```

  * Modificar /srv/pillar/top.sls para que incluya las variables definidas en local
 
```shell
# /srv/pillar/top.sls
base:
  '*':
    - ceph
    - local
```

  * Una vez realizados los cambios, forzar la recarga del "pillar" en todos los nodos

```shell
salt "*" saltutil.refresh_pillar
```

### Preparacion del "state"

  * Crear el fichero /srv/salt/local/ssh/init.sls con el siguiente contenido

```shell
# SSH keys
sshkey-root:
  ssh_auth.present:
    - user: root
    - names: {{ pillar['ceph-ssh-keys'] }}
```

  * Aplicar el "state" a todos los nodos

```shell
salt "*" state.apply local.ssh
```

### Añadir una nueva clave

Para añadir una nueva clave, solo habra que añadirla al "pillar" y aplicar el "state"

  * Editar el fichero /srv/pillar/local/ssh-keys.sls y añadir una nueva linea con la nueva clave SSH a instalar

  * Aplicar el "state"

```shell
salt "*" state.apply local.ssh
```

## Personalizar configuracion NTP

La configuracion por defecto de SES5 es usar como servidor NTP para todos los nodos la maquina de administracion (en nuestro caso, ceph-admin).

Si ya tenemos servidores NTP que podamos usar, podemos modificar la configuracion para usar ese servidor

  * Definimos el servidor /srv/pillar/ceph/stack/default/global.yml

```shell
time_init: ntp
time_server: 'ntp.example.com'
```

  * Redesplegamos configuracion

```shell
salt-run state.orch ceph.stage.3
```

  * Reiniciamos servicio NTP en todos los nodos

```shell
salt "*" cmd.run "systemctl restart ntpd"
```

  * Verificamos estado de sincronizacion de todos los nodos. Transcurrido un tiempo, deberian estar sincronizados con la maquina que hemos definido

```shell
salt "*" cmd.run "ntpq -pcrv"
```

