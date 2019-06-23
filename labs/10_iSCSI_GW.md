# Configurar la pasarela Ceph iSCSI

**AVISO IMPORTANTE:**
El soporte de la pasarela iSCSI en CentOS 7 es **experimental**. En este lab usaremos utilidades que no están incluidas en los repositorios oficiales. Además, el balanceo de carga real con MPIO no esta soportado, actualmente sólo funciona en configuración activo/standby, en el que sólo uno de los paths disponibles está activo, y en caso de fallo, se activa otro de los posibles paths.

Necesitaremos instalar los siguientes paquetes no oficiales en todos los nodos que actúen de pasarela iSCSI:

  * **kernel:** a 4.17 kernel con parches necesarios (https://github.com/ceph/ceph-client)
  * **ceph-iscsi-config:** módulos de configuración para gestionar pasarelas Ceph iSCSI (https://github.com/ceph/ceph-iscsi-config)
  * **ceph-iscsi-cli:** CLI para gestionar pasarelas Ceph iSCSI, usando LIO (https://github.com/ceph/ceph-iscsi-cli)
  * **tcmu-runner:** utilidad para gestionar el backstore LIO TCM-User, compilado con la ultima versión de librbd1 con el soporte de gestión de bloqueos necesario (https://github.com/open-iscsi/tcmu-runner/)
  * **python-rtslib**: Libreria Python para gestionar el soporte de Target iSCSI del Kernel de Linux (LIO) (https://github.com/open-iscsi/rtslib-fb)

Para simplificar la instalación, hemos creado un repositorio con todas las utilidades necesarias y sus dependencias (http://yumrepo.modularit.net/repos/ceph-iscsi-centos-7/ceph-iscsi-centos-7.repo)

## Preparación

  * Instalar el repositorio adicional:

```shell
for h in ceph-deploy ceph-mon{1,2,3} ceph-osd{1,2,3,4}; do echo $h; ssh $h "curl http://yumrepo.modularit.net/repos/ceph-iscsi-centos-7/ceph-iscsi-centos-7.repo -o /etc/yum.repos.d/ceph-iscsi-centos-7.repo"; done
```

  * Instalar el kernel en las maquinas pasarela

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "yum -y install kernel --disablerepo ceph-iscsi-centos-7"; done
```

  * Make sure the configured kernel for next boot is the new 4.17 kernel

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "grub2-editenv list"; done
```

  * If it is not, configure grub to boot from first kernel (the one we just installed)

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "grub2-set-default 0"; done
```

  * Reiniciar las maquinas para que arranquen con el nuevo kernel

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "reboot"; done
```

  * Cuando las maquinas arranquen, asegurarse que la versión del kernel es la correcta en todas las maquinas

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h "uname -a"; done
```

  * Crear el pool rbd

```
ceph osd pool create rbd 64 64
ceph osd pool application enable rbd rbd
```

  * Instalar paquetes necesarios en las maquinas pasarela

```
for h in ceph-mon{1,2,3}; do echo $h; ssh $h yum -y install tcmu-runner ceph-iscsi-cli ceph-iscsi-config ; done
```

  * Crear fichero de configuración de la pasarela (con secure=false)

```shell
cat > iscsi-gateway.cfg <<EOF
[config]
cluster_name = ceph
gateway_keyring = ceph.client.admin.keyring
api_secure = false
EOF
```

  * Copiar el fichero anterior las pasarelas

```shell
for h in ceph-mon{1,2,3}; do echo $h; scp iscsi-gateway.cfg $h:/etc/ceph; done
```

  * Reiniciar y activar los servicios en las pasarelas

```shell
for h in ceph-mon{1,2,3}; do echo $h; ssh $h 'systemctl daemon-reload'; done
for h in ceph-mon{1,2,3}; do echo $h; ssh $h 'systemctl start tcmu-runner rbd-target-gw rbd-target-api; systemctl enable tcmu-runner rbd-target-gw rbd-target-api'; done
```

  * Ejecutar "gwcli" y crear el target iSCSI con 3 portals

```shell
# gwcli
/> cd /iscsi-target 
/iscsi-target> create iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw
ok
/iscsi-target> cd iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw/gateways 
/iscsi-target...-igw/gateways> create ceph-mon1 192.168.122.21 skipchecks=true
OS version/package checks have been bypassed
Adding gateway, sync'ing 0 disk(s) and 0 client(s)
ok
/iscsi-target...-igw/gateways> create ceph-mon2 192.168.122.22 skipchecks=true
OS version/package checks have been bypassed
Adding gateway, sync'ing 0 disk(s) and 0 client(s)
ok
/iscsi-target...-igw/gateways> create ceph-mon3 192.168.122.23 skipchecks=true
OS version/package checks have been bypassed
Adding gateway, sync'ing 0 disk(s) and 0 client(s)
ok
```

  * Crear un disco de prueba (desde "gwcli")

```shell
/iscsi-target...-igw/gateways> cd /disks
/disks> create pool=rbd image=disk_1 size=1G
ok
```

  * Registrar un cliente de prueba (inititator) sin autenticacion, y asignarle el disco anterior

```shell
/disks> cd /iscsi-target/iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw/
/iscsi-target...-gw:iscsi-igw> cd hosts 
/iscsi-target...csi-igw/hosts> create iqn.1994-05.com.redhat:ceph-test
ok
/iscsi-target...hat:ceph-test> auth nochap
ok
/iscsi-target...hat:ceph-test> disk add rbd.disk_1
ok
```

  * Ya esta todo configurado en los gateways. Puede comprobarse con:

```shell
gwcli ls
```

## Cliente iSCSI

Ahora configuraremos la maquina ceph-test como cliente iscsi para probar el multipath y failover

  * Conectar por SSH a la maquina ceph-test (192.168.122.12)

```shell
ssh root@192.168.122.12
```

  * Configurar DNS y gateway (y reiniciar)

```shell
echo "nameserver 192.168.122.1" > /etc/resolv.conf
echo "GATEWAY=192.168.122.1" >> /etc/sysconfig/network-scripts/ifcfg-eth0; reboot
```

  * Cuando la maquina ceph-test este otra vez en linea, connectar por SSH

```shell
ssh root@192.168.122.12
```

  * Instalar el software necesario

```shell
yum install -y device-mapper-multipath iscsi-initiator-utils 
```

  * Configurar multipath en modo ALUA failover

```
cat > /etc/multipath.conf <<EOF
devices {
        device {
                vendor                 "LIO-ORG"
                hardware_handler       "1 alua"
                path_grouping_policy   "failover"
                path_selector          "queue-length 0"
                failback               60
                path_checker           tur
                prio                   alua
                prio_args              exclusive_pref_bit
                fast_io_fail_tmo       25
                no_path_retry          queue
        }
}
EOF
```

  * Reniciar multipathd

```shell
systemctl restart multipathd
```

  * Configuramos el InitiatorName al configurado en las pasarelas (/etc/iscsi/initiatorname.iscsi)

```shell
echo "InitiatorName=iqn.1994-05.com.redhat:ceph-test" > /etc/iscsi/initiatorname.iscsi
```

  * Reiniciamos servicio iSCSI

```shell
systemctl restart iscsi
```

  * Escaneamos targets iSCSI disponibles

```shell
iscsiadm -m discovery -t sendtargets -p 192.168.122.21
```

  * Iniciamos sesion

```shell
iscsiadm -m node -T iqn.2003-01.com.redhat.iscsi-gw:iscsi-igw --login
```

  * Ahora deberiamos ver un dispositivo multipath con 3 paths, y solo uno de ellos activo

```shell
[root@ceph-test ~]# multipath -l
3600140508f8ad6c1f3d496abc91d6617 dm-0 LIO-ORG ,TCMU device
size=1.0G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='queue-length 0' prio=0 status=active
| `- 3:0:0:0 sdb 8:16 active undef unknown
|-+- policy='queue-length 0' prio=0 status=enabled
| `- 4:0:0:0 sdc 8:32 active undef unknown
`-+- policy='queue-length 0' prio=0 status=enabled
  `- 2:0:0:0 sda 8:0  active undef unknown
```

  * Creamos un sistema de ficheros XFS y lo montamos

```shell
mkfs -t xfs /dev/mapper/3600140508f8ad6c1f3d496abc91d6617
mount /dev/mapper/3600140508f8ad6c1f3d496abc91d6617 /mnt
```

  * Creamos un fichero de prueba

```shell
echo "TEST1" > /mnt/__Test_file.txt
```

### Test de failover

  * Asegurarnos que tenemos los 3 paths, pero solo uno activo

```shell
[root@ceph-test ~]# multipath -l
3600140508f8ad6c1f3d496abc91d6617 dm-0 LIO-ORG ,TCMU device
size=1.0G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='queue-length 0' prio=0 status=active
| `- 3:0:0:0 sdb 8:16 active undef unknown
|-+- policy='queue-length 0' prio=0 status=enabled
| `- 4:0:0:0 sdc 8:32 active undef unknown
`-+- policy='queue-length 0' prio=0 status=enabled
  `- 2:0:0:0 sda 8:0  active undef unknown
```

  * Comprobamos que pasarela es la activa. Buscar "Owner" en la salida de "gwcli ls"

```
     o- hosts .......................................................................................................... [Hosts: 1]
        o- iqn.1994-05.com.redhat:ceph-test ................................................. [LOGGED-IN, Auth: None, Disks: 1(1G)]
          o- lun 0 ............................................................................. [rbd.disk_1(1G), Owner: ceph-mon2]
```

  * En este ejemplo, el gateway activo es **ceph-mon2**
  * Matar la pasarela activa **ceph-mon2** (apagado "a lo bestia")
  * Intentar crear un nuevo fichero

```shell
echo "TEST2" > /mnt/__Test_file2.txt
```

  * Las operaciones de I/O se bloquearan durante unos segundos
  * Tras el timeout configurado, el multipath activara uno de los paths disponibles y la operacion de escritura se completará

  * Verifica el estado del multipath. Ahora deberia mostrar otro de los paths como activo, y el anterior deberia estar marcado como "failed"


