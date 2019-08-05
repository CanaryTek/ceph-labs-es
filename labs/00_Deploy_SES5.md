# Despliegue del cluster SES 5

Para empezar esta fase, asumimos que las 9 maquinas del lab estan completamente instaladas y con la red configurada de acuerdo a las especificaciones de la fase de preparacion

Salvo en los casos en los que se especifique lo contrario, todas las acciones descritas deben ejecutarse en la maquina ceph-admin

## Preparacion

  * Creamos fichero /etc/hosts

```bash
cat >> /etc/hosts <<__EOF__
10.10.11.1      ceph-admin
10.10.11.10     ceph-test
10.10.11.11     ceph-mon1
10.10.11.12     ceph-mon2
10.10.11.13     ceph-mon3
10.10.11.21     ceph-osd1
10.10.11.22     ceph-osd2
10.10.11.23     ceph-osd3
10.10.11.24     ceph-osd4
__EOF__
```

  * Configuramos acceso sin clave a todas las maquinas

```bash
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; ssh-copy-id root@$h ; done
```

  * Verificamos acceso a todas las maquinas

```bash
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; ssh root@$h hostname ; done
```

  * Copiamos el /etc/hosts al resto de maquinas

```bash
for h in ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; scp /etc/hosts root@$h:/etc ; done
```

  * Verificamos conectividad entre los OSD

```bash
for h in ceph-osd{1,2,3,4}; do echo "*** $h"; ssh root@$h ping -c3 ceph-mon1 ; done
```

  * Desactivamos ipv6 en todos los nodos (para evitar retardos en el arranque)

```bash
cat > /etc/sysctl.d/disable-ipv6.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
for h in ceph-test ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo "*** $h"; scp /etc/sysctl.d/disable-ipv6.conf root@$h:/etc/sysctl.d/disable-ipv6.conf; done
```

## Instalacion de repositorios en ceph-admin

En esta sección vamos a alojar los repositorios de  instalacion de SLE12SP3 y SES5 en la maquina ceph-admin

  * Creamos los directorios para los repos

```bash
mkdir -p /srv/www/htdocs/repos/{sle12sp4,ses5}
```

  * Montamos el DVD de SLE12SP3 y copiamos el contenido del directorio "suse" a uno de los directorios que hemos creado

```bash
rsync -avz suse/ root@192.168.122.11:/srv/www/htdocs/repos/sle12sp4
```

  * Montamos el DVD de SES5 y copiamos el contenido del directorio "suse" alotro de los directorios que hemos creado

```bash
rsync -avz suse/ root@192.168.122.11:/srv/www/htdocs/repos/ses5
```

  * Instalamos apache y createrepo (desde el CD de instalacion)

```bash
zypper in apache createrepo
```

  * Arrancamos y activamos el servicio apache2

```bash
systemctl start apache2; systemctl enable apache2
```

  * Creamos los repos

```bash
cd /srv/www/htdocs/repos/sle12sp4; rm -rf repodata; createrepo -v .
cd /srv/www/htdocs/repos/ses5; rm -rf repodata; createrepo -v .
```
  * Borramos el repo del CD de instalacion y creamos los nuevos (en todas las maquinas)

```bash
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; ssh root@$h zypper rr 1 ; done
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; ssh root@$h zypper ar -p 20 http://ceph-admin/repos/sle12sp4 sle2sp4 ; done
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; ssh root@$h zypper ar -p 10 http://ceph-admin/repos/ses5 ses5 ; done
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; ssh root@$h zypper -n --no-gpg-checks ref ; done
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4} ceph-test; do echo "*** $h"; ssh root@$h zypper lr ; done
```

  * Definimos una prioridad mayor para el repo de SES5 porque en caso de que en un paquete exista en ambos repos, queremos que se instale el de SES5. Por ejemplo, en SLE12SP4 viene una version de librados2 incompatible con SES5

## Instalacion de DeepSea

  * Instalar DeepSea (en ceph-admin)

```shell
zypper in deepsea
```

  * Arrancar servicio salt-master

```shell
systemctl enable salt-master.service ; systemctl start salt-master.service
systemctl enable salt-api.service ; systemctl start salt-api.service
```

  * Instalar y arrancar salt-minion en todas las maquinas del cluster

```shell
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo "*** $h"; ssh $h 'zypper --non-interactive in salt-minion'; done
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo "*** $h"; ssh $h 'echo "master: ceph-admin" > /etc/salt/minion.d/master.conf'; done
for h in ceph-admin ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo "*** $h"; ssh $h 'systemctl start salt-minion; systemctl enable salt-minion'; done
```

  * Aceptar todas las claves

```shell
salt-key -A
```

  * Comprobar que responden todos los nodos

```shell
salt "*" test.ping
salt "*" test.version
```

## Despliegue de SES5 con DeepSea

  * En la configuracion por defecto, la maquina de despliegue actua de servidor NTP, asi que tenemos que tenerlo activo y sincronizado

```shell
ntpdate -dv pool.ntp.org
systemctl enable ntpd; systemctl start ntpd
```

  * Añadir el "grain" deepsea a todos los nodos

```shell
salt "ceph*" grains.append deepsea default
```

  * El resto de los pasos es recomendable realizarlos con una consola de monitorizacion de deepsea abierta

```shell
deepsea monitor
```

  * Ejecutar stage 0 (preparacion y actualizacion)

```shell
salt-run state.orch ceph.stage.0
```

  * Ejecucion stage 1 (discover)

```shell
salt-run state.orch ceph.stage.1
```

  * Configurar fichero policy  /srv/pillar/ceph/proposals/policy.cfg

```
## Cluster Assignment
cluster-ceph/cluster/ceph-*.sls

## Roles
# ADMIN
role-master/cluster/ceph-admin*.sls
role-admin/cluster/ceph-admin*.sls

# MON
role-mon/cluster/ceph-mon*.sls

# MGR (mgrs are usually colocated with mons)
role-mgr/cluster/ceph-mon*.sls

# MDS
role-mds/cluster/ceph-mon*.sls

# ISCSI GW
role-igw/cluster/ceph-mon*.sls

# Rados GW
role-rgw/cluster/ceph-mon*.sls

# NFS
role-ganesha/cluster/ceph-mon*.sls

# openATTIC
role-openattic/cluster/ceph-admin.sls

# COMMON
config/stack/default/global.yml
config/stack/default/ceph/cluster.yml

## Profiles
profile-default/cluster/*.sls
profile-default/stack/default/ceph/minions/*.yml
```

  * Tambien se pueden cambiar algunos parametros de configuracion, como la red de los OSD en:

```
/srv/pillar/ceph/proposals/config/stack/default/ceph/cluster.yml
```

  * Ejecutar stage 2 (setup)

```shell
salt-run state.orch ceph.stage.2
```

  * Verificar pillar

```shell
salt '*' pillar.items
```

  * Ejecutar stage 3 (Depliegue de cluster RADOS)

```shell
salt-run state.orch ceph.stage.3
```

  * Una vez completada esta fase, tenemos un cluster RADOS basico. Verificar estado con

```shell
ceph -s
```

  * Ejecutar stage 4 (Instalar servicios) 

```shell
salt-run state.orch ceph.stage.4
```

  * Verficar estado

```shell
ceph -s
```

