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

  * Instalamos sofware cliente S3 (libs3-tools)

```
zypper in libs3-tools
```

  * Establecemos las variables de entorno con las credenciales que obtuvimos al crear el usuario

```
export S3_ACCESS_KEY_ID=K9HE6BFD9BYCCL9NRHF8
export S3_SECRET_ACCESS_KEY=QmQJAhmT33Ch1kE8FnIkoDMWtBvjQQhN8lBfbpOW
export S3_HOSTNAME=192.168.122.21
```

  * Puesto que RadosGW está usando HTTP y el comando "s3" usa HTTPS por defecto, tenemos que forzarle a usar HTTP con la opción "-u"

  * Listamos los "buckets"

```
s3 -u list
```

  * Creamos un bucket de prueba (test-bucket)

```
s3 -u create test-bucket
```

  * Almacenamos un objeto

```
echo "Test object" | s3 -u put test-bucket/test
```

  * Verificamos que el objeto se ha creado

```
s3 -u list test-bucket
```

  * Descargamos el objeto

```
s3 -u get test-bucket/test
```

  * Borramos el objeto

```
s3 -u delete test-bucket/test
```

