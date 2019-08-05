# CloudLab: SuSE Enterprise Storage 5

En esta CloudLab desplegaremos y usaremos un cluster de almacenmiento Ceph con SuSE Enterprise Storage 5

## Preparacion del Lab

Normalmente un cluster Ceph requiere 2 redes diferentes:

  * Red de servicio: Es la red en la que se ofrece el servicio de almacenamiento. A esta red deben estar conectados todos los nodos del cluster
  * Red de replicacion de datos: Es la red que utilizan los OSD para replicarse los datos entre ellos. Solos los OSD necesitan estar conectados a esta red

En este Lab ademas simularemos una configuracion bastante habitual: El cluster Ceph esta aislado del resto de la red y solo es accesible desde fuera a traves de la maquina de gestion del cluster, que ademas de la conexion a la red de servicio Ceph, tendra una conexion adicional a una red externa, que es laque utilizaremos para acceder al cluster

Por tanto, utilizaremos 3 redes

| Red | Bridge | Rango IP | Descripcion |
|-----|--------|-------|-------------| 
| Externa | virbr0 | 192.168.122.0/24 | Red externa. Solo ceph-admin |
| Ceph | sesbr0 | 10.10.11.0/24 | Red de servicio Ceph |
| OSD | osdbr0 | 10.10.12.0/24 | Red de replicacion de datos de los OSD |

Ademas, de esas 3 redes, utilizaremos un total de  9 maquinas, cuyo uso y configuracion detallamos a continuación

| Nombre | IP | Recursos | Discos | Servicios | Descripcion |
|------|----|-----------|------|----------|-------------| 
| ceph-admin | 192.168.122.11,10.10.11.1 | 4GB, cores | 1x40G | OpenATTIC, salt-master | Deploy and monitoring, network gateway |
| ceph-test | 10.10.11.10 | 1GB, 1 core | 1x40G | None | Just a test host to use as a client for iSCSI, NFS, etc |
| ceph-mon{1,2,3} | 10.10.11.1{1,2,3} | 1GB, 1 core| 1x40G | MON, MGR, MDS, iSCSI Gateway, NFS-Ganesha, RadosGW | Monitors and service gateways |
| ceph-osd{1,2,3,4} | 10.10.11.2{1,2,3,4},10.10.12.2{1,2,3,4} | 1GB, 1 core | 1x40G + 2x30G | OSD | Ceph OSD storage hosts |

## Preparar el lab

  1. Descargar los CD de instalacion de SuSE Linux 12 SP4 y SuSE Enterprise Storage 5 (ambos disponibles para evaluacion)

  2. Copiar las imagenes descargadas al directorio isos

  3. Si usa btrfs, crear el volumen "vm" para poder crear snapshots del entorno

```shell
sudo btrfs subvolume create vm
```

  4. Initializar las maquinas del Lab

```shell
sudo rake init_vms
```

  5. Apagar todas las maquinas para ir arrancandolas e instalandolas una a una

```shell
sudo rake destry_vms
```

  6. Instalar SLE SP4 en todas las maquinas con las siguientes opciones:

   * No registrar el sistema (Skip Registration)
   * Instalar perfil "Default System" (Luego ajustaremos que paquetes instalar)
   * En Partitioning, seleccionar "Create Partition Setup"
     * Seleccionar **unicamente** el primer disco
     * Seleccionar "Edit Proposal" y desactivar la opcion a "Separate home partition"
   * Seleccionar "Skip user creation"
   * Cuando lleguemos al resumen final, seleccionar el software a instalar y deseleccionar Gnome y X Windows
   * Seleccionar "Disable Firewall"

  7. Configurar la red en todas las maquinas con la configuracion indicada en la tabla de arriba (Yast -> System -> Network Settings)

## Tareas habituales 

  1. Arrancar/parar todas las maquinas del lab

```shell
sudo rake start_vms
sudo rake stop_vms
```

  2. Crear un snapshot

```shell
sudo btrfs snap -r vm .snapshots/01_after_installation
```

  3. Volvar al estado almacenado en un snapshot

```shell
sudo btrfs subvol delete vm
sudo btrfs snap .snapshots/01_after_installation vm
```

## Labs

  * [Lab 0: Despliegue de SuSE Enterprise Storage 5](labs/00_Deploy_SES5.md)
  * [Lab 1: Parada y Arranque del Cluster](labs/01_Start_Stop_Cluster.md)
  * [Lab 2: Pools](labs/02_Pools.md)
  * [Lab 3: RBD Rados Block Device](labs/03_RBD_Rados_Block_Device.md)
  * [Lab 4: Ubicación de datos](labs/04_Ubicacion_de_datos.md)
  * [Lab 5: Crush Maps](labs/05_Crush_Maps.md)
  * [Lab 6: CephFS](labs/06_CephFS.md)
  * [Lab 7: Usuarios y Permisos](labs/07_Usuarios_y_Permisos.md)
  * [Lab 8: RadosGW](labs/08_RadosGW.md)
  * [Lab 9: Ganesha NFS](labs/09_Ganesha_NFS.md)
  * [Lab 10: iSCSI Gateway](labs/10_iSCSI_GW.md)
  * [Lab 11: Eliminar un osd](labs/11_Remove_osd.md)
  * [Lab 12: Añadir un osd](labs/12_Add_osd.md)

**TODO**

  * [Lab X: OSD failure]
  * [Lab X: MON failure]
  * [Lab X: Replace a MON]
