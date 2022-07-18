# Dataproc + Sqoop 
![dataproc y sqoop](https://user-images.githubusercontent.com/50117113/179570300-60a13548-6e49-4c8c-9b4e-9052427ace73.png)
## Objetivo
#### Mediante un cluster de Dataproc que tenga instalada la herramienta Sqoop, migrar las tablas almacenadas en una instancia CloudSQL-SQLServer hacia Google Cloud Storage
## 1. Creacion del cluster de Dataproc con la herramienta Sqoop
Ejecutar el siguiente comando en la Cloud Shell de nuestro proyecto:
`gcloud dataproc clusters create <CLUSTER_NAME> \`\
`--region <MY_REGION> \`\
`--tags=cluster-dataproc \`\
`--master-machine-type n1-standard-2 \`\
`--master-boot-disk-size 90 \`\
`--num-workers 2 \`\
`--worker-machine-type n1-standard-2 \`\
`--worker-boot-disk-size 90 \`\
`--image-version 2.0-debian10 \`\
`--initialization-actions 'gs://bkt-actions/sqoop.sh' \`\
`--scopes sql-admin \`\
`--properties=dataproc:dataproc.conscrypt.provider.enable=false`

Luego de unos minutos se creara nuestro cluster de Dataproc, y si nos dirigimos al servicio de Compute Engine podremos ver tres maquinas corriendo, un master y dos workers correspondientes al cluster previamente creado.

## 2. Conexion del cluster Dataproc a la instancia CloudSQL-SQLServer
Para establecer una conexion entre el cluster de Dataproc y la instancia de CloudSQL-SQLServer, necesitamos haber creado previamente la instancia CloudSQL-SQLServer. Luego de crear la instancia nos dirijimos al servicio de Compute Engine y accedemos mediante SSH a las tres compute engine pertenecientes al cluster de Dataproc y ejecutamos los siguientes comandos en cada una de ellas con el objetivo de inicializar un proxy hacia el SQL Server.
`wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy`\
`chmod +x cloud_sql_proxy`\
`./cloud_sql_proxy -instances=<INSTANCE_CONNECTION_NAME>=tcp:<PORT>`\

**NOTA:** El parametro INSTANCE_CONNECTION_NAME podras verlo en la pestaña del servicio CloudSQL correspondiente a tu instancia creada, su formato suele ser `myproject:myregion:myinstance`. En cuanto a PORT, en SQL Server normalmente es 1433, en MySQL 3306 y en Postgre 5432.

## 3. Correr job de Sqoop para realizar la migracion
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



