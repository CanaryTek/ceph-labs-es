# Configurar la pasarela Ceph iSCSI

En este lab configuramos el acceso iSCSI MPIO con multipath

## Crear un target iSCSI

  * Acceder a OpenATTIC
  * Ir a "iSCSI"
  * Si no tenemos ya un target "demo", crearlo
    * Añadir todas las pasarelas a la lista de "portals"
    * Añadir o crear una imagen RBD

## Cliente iSCSI

Ahora vamos a hacer la configuracion en el cliente. Usaremos la maquina ceph-test

  * Conectar a la máquina ceph-test (10.10.11.10)

```shell
ssh root@ceph-test
```

  * Instalar software necesario

```shell
zypper in yast2-iscsi-client yast2-storage
```

  * Activamos y arrancamos multipathd

```shell
systemctl enable multipathd ; systemctl start multipathd
```
 
  * Ejecutamos "yast"
    * Network Services -> iSCSI Initiatior
    * Ir a "Discovered Targets"
    * Indicar la IP de cualquiera de las pasarelas (p.ej. 10.10.11.11)
    * Deberian aparecer todos los "portals" de ese target, pero parece que hay un bug e yast que hace que se muestren uno encima de otro
    * Un truco "sucio" es borrar el "portal" hasta que aparece el que nos queremos conectar, y luego volver a repetir el discover. Y asi hasta que nos conectemos a todas
    * En "Connected Targets" asegurarnos que estamos conectados a todas las pasarelas

  * Restart multipathd

```shell
systemctl restart multipathd
```

  * Verificar que vemos el target a traves de un dispositivo multipath con multiples paths activos


```shell
# multipath -l
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=50 status=active
  |- 3:0:0:0 sda 8:0  active ready running
  |- 4:0:0:0 sdb 8:16 active ready running
  `- 2:0:0:0 sdc 8:32 active ready running
```

  * Creamos sistema de ficheros y montamos

```shell
mkfs -t ext4 /dev/mapper/360014054dfef910bfd8306da0fd4980d
mount /dev/mapper/360014054dfef910bfd8306da0fd4980d /mnt
```

  * Creamos fichero de prueba

```shell
echo "TEST1" > /mnt/__Test_file.txt
cat /mnt/__Test_file.txt
```

### Test de failover

  * Asegurarse de que estan activos los 3 paths

```shell
ceph-test:~ # multipath -l
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=0 status=active
  |- 3:0:0:0 sda 8:0  active undef unknown
  |- 4:0:0:0 sdb 8:16 active undef unknown
  `- 2:0:0:0 sdc 8:32 active undef unknown
```

  * Parar una de las pasarelas (apagado de maquina)
  * Deberia verse el evento de fallo en el kernel

```shell
dmesg -T
```

  * Uno de los paths deberia aparecer como "failed" pero todo deberia seguir funcionando

```shell
ceph-test:~ # multipath -l
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=0 status=active
  |- 3:0:0:0 sda 8:0  failed undef unknown
  |- 4:0:0:0 sdb 8:16 active undef unknown
  `- 2:0:0:0 sdc 8:32 active undef unknown

```

  * Asegurarse de que aun hay acceso al disco iSCSI

```shell
echo "TEST1" > /mnt/__Test_file.txt
cat /mnt/__Test_file.txt
```

  * Volver a arrancar la pasarela

  * Despues de algun tiempo, podremos ver el evento de reconexion en el log del kernel

```shell
dmesg -T
```

  * Y deberian volver a activarse los 3 paths

```shell
ceph-test:~ # multipath -l
360014054dfef910bfd8306da0fd4980d dm-0 SUSE,RBD
size=1.0G features='2 queue_if_no_path retain_attached_hw_handler' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=0 status=active
  |- 3:0:0:0 sda 8:0  active undef unknown
  |- 4:0:0:0 sdb 8:16 active undef unknown
  `- 2:0:0:0 sdc 8:32 active undef unknown
```

