# Dataproc + Sqoop 
![dataproc y sqoop](https://user-images.githubusercontent.com/50117113/179570300-60a13548-6e49-4c8c-9b4e-9052427ace73.png)
## Objetivo
#### Mediante un cluster de Dataproc que tenga instalada la herramienta Sqoop, migrar las tablas almacenadas en una instancia CloudSQL-SQLServer hacia Google Cloud Storage
<p align="center">
<img width="488" alt="sqoop_y_dp" src="https://user-images.githubusercontent.com/50117113/179613477-257c8c24-7a36-4354-9cdf-a05e8eae8060.png">
</p>

## Dataproc
Dataproc es un servicio administrado de Spark y Hadoop con el que puede aprovechar herramientas de datos de código abierto para procesamientos por lotes, búsquedas, transmisiones y machine learning. Con la automatización de Dataproc, podrá crear clústeres rápidamente, administrarlos con facilidad y ahorrar dinero desactivándolos cuando no los necesite. Con un gasto menor de tiempo y dinero en administración, puedes enfocarte en tus trabajos y datos.\
Cuando se lo compara con productos tradicionales, locales y servicios en la nube de la competencia, Dataproc tiene varias ventajas únicas para clústeres de tres a cientos de nodos:
- Bajo costo: Dataproc tiene un precio de solo 1 centavo de dolar por CPU virtual en tu clúster por hora, además de los otros recursos de Cloud Platform que uses. Además de este costo bajo, los clústeres de Dataproc pueden incluir instancias interrumpibles que tienen costos de procesamiento más bajos, lo que reduce aún más tus costos.
- Muy rapido: si no se usa Dataproc, puede tardar de cinco a 30 minutos crear clústeres locales de Spark y Hadoop o a través de los proveedores de IaaS. En comparación, los clústeres de Dataproc se inician, escalan y cierran rápido; cada una de estas operaciones tarda 90 segundos o menos en promedio.
- Integrado: ataproc tiene integración incorporada con otros servicios de Google Cloud Platform, como BigQuery, Cloud Storage, Cloud Bigtable, Cloud Logging y Cloud Monitoring, por lo que tienes más que un clúster de Spark o Hadoop: tienes una plataforma de datos completa.
- Fully Managed: usa los clústeres de Spark y Hadoop sin la asistencia de un administrador o un software especial. Puedes interactuar con facilidad con los clústeres y con los trabajos de Spark o Hadoop a través de Google Cloud Console, el SDK de Cloud o la API de REST de Dataproc. Cuando terminas de usar un clúster, puedes apagarlo para que no gastes dinero en un clúster inactivo.
- Simple y conocido: no necesitas aprender a usar herramientas o API nuevas para usar Dataproc, lo que facilita el traslado de proyectos existentes a Dataproc sin volver a desarrollarlos. Spark, Hadoop, Pig y Hive se actualizan con frecuencia, por lo que puedes ejecutar tus tareas con rapidez.

## Sqoop
Sqoop es una herramienta diseñada para transferir datos entre Hadoop y bases de datos relacionales o mainframes. Se puede utilizar Sqoop para importar datos de un sistema de administración de bases de datos relacionales (RDBMS) a HDFS. Sqoop utiliza MapReduce para importar y exportar los datos, lo que proporciona operación paralela y tolerancia a fallos. Entre los formatos de archivos resultantes de las importaciones podemos encontrar CSV, Avro o SequenceFiles.\
La mayoría de los aspectos de los procesos de importación, generación de código y exportación se pueden personalizar. Para las bases de datos, puede controlar el rango de filas específico o las columnas importadas. Puede especificar delimitadores y caracteres de escape particulares para la representación basada en archivos de los datos, así como el formato de archivo utilizado.

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
Para establecer una conexion entre el cluster de Dataproc y la instancia de CloudSQL-SQLServer, necesitamos haber creado previamente la instancia CloudSQL-SQLServer con un usuario y contraseña que luego necesitaremos en el paso 3. Luego de crear la instancia nos dirijimos al servicio de Compute Engine y accedemos mediante SSH* a las tres compute engine pertenecientes al cluster de Dataproc y ejecutamos los siguientes comandos en cada una de ellas con el objetivo de inicializar un proxy con el CloudSQL-SQL Server.\
El proxy de autenticacion de CloudSQL nos proporciona acceso seguro a nuestra instancia CloudSQL sin necesidad de tener redes autorizadas ni de configurar SSL. Basicamente el proxy usa un tunel seguro para establecer una comunicacion entre la instancia de CloudSQL y las maquinas (tanto master como workers) del cluster de Dataproc.

**Descarga el proxy de autenticacion de CloudSQL:**\
`wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy`\
**Haz que el proxy de autenticación de Cloud SQL sea ejecutable:**\
`chmod +x cloud_sql_proxy`\
**Inicia el proxy de autenticacion:**\
`./cloud_sql_proxy -instances=<INSTANCE_CONNECTION_NAME>=tcp:<PORT>`

**NOTA:** El parametro INSTANCE_CONNECTION_NAME podras verlo en la pestaña del servicio CloudSQL correspondiente a tu instancia creada, su formato suele ser `myproject:myregion:myinstance`. En cuanto a PORT, en SQL Server normalmente es 1433, en MySQL 3306 y en Postgre 5432.

*Para acceder mediante SSH podemos hacerlo mediante la interfaz web del servicio Compute Engine, o ejecutando el siguiente comando en Cloud Shell:
`gcloud compute ssh <INSTANCE_NAME>`

## 3 Correr job de Sqoop para migrar una tabla completa
Abrir una nueva conexion mediante SSH al master del cluster Dataproc, para diferenciar al master debemos notar que éste lleva una letra "m" al final de su nombre, mientras que los workers terminan su nombre con una letra "w" seguido de un numero que indicando el indice. Una vez abierta la conexion SSH al master, debemos ejecutar el siguiente comando para correr el job de Sqoop que migrará los datos al destino indicado.
```
sqoop import -Dmapred.job.queuename=<JOB_NAME> \
--connect <JDBC_STR> \
--username <MY_USERNAME> \
--password <MY_PASSWORD> \
--table <TABLE_NAME> \
--as-avrodatafile \
--target-dir <MY_BUCKET> \
--m 1
```

#### Explicacion de los parametros:
`JOB_NAME` Nombre a eleccion para identificar al job.\
`JDBC_STR` Cadena de conexion que nos permitira conectarnos a la base de datos, por ejemplo `jdbc:sqlserver://127.0.0.1:1433;database=my_db`\
`MY_USERNAME` Usuario de la instancia CloudSQL.\
`MY_PASSWORD` Contraseña asociada al usuario de la instancia CloudSQL.\
`TABLE_NAME` Nombre de la tabla que queremos migrar, no es necesario indicar nombre de la base de datos.\
`MY_BUCKET` Nombre del bucket de GCS donde seran exportados los datos, especificarlo con el formato `gs://my_bucket`\
`m 1` En este caso se especificó utilizar 1 "mapper", por defecto Sqoop utiliza 4.

## 4 Correr job de Sqoop para cargas incrementales
Una vez que hemos migrado una tabla, es probable que luego querramos migrar solo los datos nuevos que vayan ingresando a la instancia CloudSQL-SQLServer. Para tal situacion existen las cargas incrementales, y podemos llevarlas a cabo mediante la ejecucion del siguiente comando. 
```
sqoop import -Dmapred.job.queuename=<JOB_NAME> \
--connect <JDBC_STR> \
--username <MY_USERNAME> \
--password <MY_PASSWORD> \
--table <TABLE_NAME> \
--check-column <COLUMN_NAME> \
--incremental append \
--last-value <VALUE> \
--as-avrodatafile \
--target-dir <MY_BUCKET> \
--m 1
```
Como puede observar algunos parametros son los mismos del paso 3 y por lo tanto no seran explicados nuevamente. Un nuevo parametro es `check-column` el cual indica cual columna será examinada para determinar si hay nuevas filas en la tabla `TABLE_NAME`, la columna no puede ser del tipo CHAR/NCHAR/VARCHAR/VARNCHAR/ LONGVARCHAR/LONGNVARCHAR.\
Luego tenemos otro parametro llamado `incremental` al cual debemos proveerle el modo en que Sqoop determinará cuales filas son nuevas, por un lado tenemos el modo `append` que es el utilizado en este ejemplo y por otro lado tenemos el modo `last-modified`. El modo `append` debe utilizarse cuando a medida que se agreguen nuevas filas, los valores de la columna `COLUMN_NAME` sean crecientes, entonces Sqoop observará `VALUE` del parametro `last-value` e importará aquellas filas que tengan un valor mayor a este.



