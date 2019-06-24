# RadosGW

## S3 API

En esta sección configuraremos la interfaz de objetos mediante API S3 con RadosGW

## Creacion de usuario

  * Creamos un usuario de prueba

```
radosgw-admin user create --uid s3test --display-name="Test user for S3 API"
```

  * Esto creará un usuario, su "access_key" y su "secret_key"

## Usar la API S3

Estos pasos pueden hacerse desde cualquier máquina con acceso al nodo RadosGW

  * Instalamos sofware cliente s3cmd

```shell
zypper ar https://download.opensuse.org/repositories/Cloud:/Tools/SLE_12_SP4/Cloud:Tools.repo                     
zypper in s3cmd                                                                                                   
```

  * Lanzamos configuracion

```shell
s3cmd --configure
```

  * Indicamos los siguientes parametros
    * Access Key y Secret Key: Las del usuario que queremos usar
    * Default Region: Dejamos US
    * S3 Endpoint: ceph-mon1 (o donde tengamos la pasarela)
    * DNS-style bucket: ceph-mon1 (igual que arriba)
    * Use HTTPS protocol: No

  * Listamos los "buckets"

```
s3cmd ls
```

  * Creamos un bucket de prueba (test)

```
s3cmd mb s3://test
```

  * Almacenamos un objeto

```
echo "Test object" | s3cmd put - s3://test/object1
```

  * Verificamos que el objeto se ha creado

```
s3cmd ls s3://test
```

  * Descargamos el objeto

```
s3cmd get s3://test/object1
```

  * Borramos el objeto

```
s3cmd rm s3://test/object1
```

  * Borramos el bucket

```
s3cmd rb s3://test
```

