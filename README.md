# Dataproc + Sqoop 
![dataproc y sqoop](https://user-images.githubusercontent.com/50117113/179570300-60a13548-6e49-4c8c-9b4e-9052427ace73.png)
## Objetivo
#### Mediante un cluster de Dataproc que tenga instalada la herramienta Sqoop, migrar las tablas almacenadas en una instancia CloudSQL-SQLServer hacia Google Cloud Storage
<p align="center">
<img width="488" alt="sqoop_y_dp" src="https://user-images.githubusercontent.com/50117113/179613477-257c8c24-7a36-4354-9cdf-a05e8eae8060.png">
</p>

## 1. Creacion del cluster de Dataproc con la herramienta Sqoop
Por defecto el cluster de Dataproc no tiene instalada la herramienta Sqoop, por lo cual es necesario especificar la initialization action correspondiente a Sqoop para que se instale automaticamente en el cluster junto con sus respectivas dependencias, mediante el parametro `initialization-actions`. En entornos de produccion no es recomendable hacer referencias a initialization actions que se encuentren en buckets publicos, en su lugar se recomienda copiar el script de initialization action, colocarlo en un bucket propio y hacer referencia a éste.
Ejecutar el siguiente comando en la Cloud Shell de nuestro proyecto, con el objetivo de crear un cluster de Dataproc que tenga instalado Sqoop.
```
gcloud dataproc clusters create <CLUSTER_NAME> \
--region <MY_REGION> \
--tags=cluster-dataproc \
--master-machine-type n1-standard-2 \
--master-boot-disk-size 90 \
--num-workers 2 \
--worker-machine-type n1-standard-2 \
--worker-boot-disk-size 90 \
--image-version 2.0-debian10 \
--initialization-actions 'gs://bkt-actions/sqoop.sh' \
--scopes sql-admin \
--properties=dataproc:dataproc.conscrypt.provider.enable=false
```

Luego de unos minutos se creara nuestro cluster de Dataproc, y si nos dirigimos al servicio de Compute Engine podremos ver tres maquinas corriendo, un master y dos workers correspondientes al cluster previamente creado.
El parametro `region` es obligatorio indicarlo, a modo de ejemplo podria usar `us-central1`. No es necesario que la region del cluster sea la misma que la region donde se encuentra la instancia CloudSQL.\
El parametro `scopes` se refiere a los permisos que tendrán tanto el master como los workers del cluster, en nuestro caso solo indicamos `sql-admin` pero pueden especificarse mas de uno si los separamos por comas. El scope puede ser indicado mediante un alias como lo hemos hecho o mediante la URI del permiso, ver la siguiente [Tabla](https://cloud.google.com/sdk/gcloud/reference/dataproc/clusters/create#:~:text=Available%20aliases%20are%3A-,Alias,-URI) para mas detalle.\
El parametro `properties=dataproc:dataproc.conscrypt.provider.enable=false` deshabilita Conscrypt como el principal proveedor de seguridad de Java, y por lo tanto estamos recurriendo a la implementacion de SSL incorporada por Java.\
Luego tenemos algunos parametros que tienen que ver con el hardware que tendra tanto el master como los workers del cluster, entre ellos `master-machine-type`, `master-boot-disk-size`, `worker-machine-type`, `worker-boot-disk-size`. Tambien se pueden especificar la cantidad de workers con `num-workers` donde el minimo es 2.\
Y por ultimo en cuanto al software se puede especificar la version del SO mediante `image-version`. Ver el siguiente [link](https://cloud.google.com/dataproc/docs/concepts/versioning/dataproc-versions) para conocer sobre versiones soportadas por Dataproc.

## 2. Conexion del cluster Dataproc a la instancia CloudSQL-SQLServer
Para establecer una conexion entre el cluster de Dataproc y la instancia de CloudSQL-SQLServer, necesitamos haber creado previamente la instancia CloudSQL-SQLServer. Luego de crear la instancia nos dirijimos al servicio de Compute Engine y accedemos mediante SSH a las tres compute engine pertenecientes al cluster de Dataproc y ejecutamos los siguientes comandos en cada una de ellas con el objetivo de inicializar un proxy hacia el SQL Server.\
Como no hemos creado una VPC para poder comunicar los servicios de nuestro proyecto, lo que haremos sera inicializar un proxy Basicamente el proxy sirve como un puente para poder conectar la instancia CloudSQL con el master y los workers del cluster Dataproc.\

`wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy`\
`chmod +x cloud_sql_proxy`\
`./cloud_sql_proxy -instances=<INSTANCE_CONNECTION_NAME>=tcp:<PORT>`

**NOTA:** El parametro INSTANCE_CONNECTION_NAME podras verlo en la pestaña del servicio CloudSQL correspondiente a tu instancia creada, su formato suele ser `myproject:myregion:myinstance`. En cuanto a PORT, en SQL Server normalmente es 1433, en MySQL 3306 y en Postgre 5432.

## 3 Correr job de Sqoop para migrar una tabla completa
Abrir una nueva conexion mediante SSH al master del cluster Dataproc, para diferenciar al master debemos notar que éste lleva una letra "m" al final de su nombre, mientras que los workers terminan su nombre con una letra "w" seguido de un numero que indicando el indice. Una vez abierta la conexion SSH al master, debemos ejecutar el siguiente comando para correr el job de Sqoop que migrará los datos al destino indicado.\
`sqoop import -Dmapred.job.queuename=<SOME_NAME> \`\
`--connect <JDBC_STR> \`\
`--username <MY_USERNAME> \`\
`--password <MY_PASSWORD> \`\
`--table <TABLE_NAME> \`\
`--as-avrodatafile \`\
`--target-dir <MY_BUCKET> \`\
`--m 1`

#### Explicacion de los parametros:
`SOME_NAME` Nombre a eleccion para identificar al job.\
`JDBC_STR` Cadena de conexion que nos permitira conectarnos a la base de datos, por ejemplo `jdbc:sqlserver://127.0.0.1:1433;database=my_db`\
`MY_USERNAME` Usuario asociado a la base de datos.\
`MY_PASSWORD` Contraseña.\
`TABLE_NAME` Nombre de la tabla que queremos migrar, no es necesario indicar nombre de la base de datos.\
`MY_BUCKET` Nombre del bucket de GCS donde seran exportados los datos, especificarlo con el formato `gs://my_bucket`\
`m 1` En este caso se especificó utilizar 1 "mapper", por defecto Sqoop utiliza 4.

## 4 Correr job de Sqoop para cargas incrementales
Una vez que hemos migrado una tabla, es probable que luego querramos migrar solo los datos nuevos que vayan ingresando a la instancia CloudSQL-SQLServer. Para tal situacion existen las cargas incrementales,




