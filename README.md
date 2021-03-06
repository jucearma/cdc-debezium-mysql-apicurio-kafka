# Monitoreo de Base de Datos MySQL con Debezium

En este tutorial, iniciará los servicios de Debezium, ejecutará un servidor de base de datos MySQL con una base de datos de ejemplo simple y utilizará Debezium para monitorear la base de datos en busca de cambios.

## Herramientas

### Debezium

> Debezium es una plataforma distribuida que convierte sus bases de datos existentes en flujos de eventos, para que las aplicaciones puedan ver y responder de inmediato a cada cambio de nivel de fila en las bases de datos.

### Apicurio Registry

Epicurio Registry es una implementacion de Esquema de Registro la cual actua como una API de Codigo Abierto, ademas de proporcionar bibliotecas cliente y una excelente integracion con Apache Kafka y Kafka Connect en forma de serializadores y convertidores. [Apicurio](https://github.com/Apicurio/apicurio-registry)

## Consideraciones

Puede visualizar los logs de cada contenedor con el siguiente comando: *docker logs --follow --name **container-name***

## Precondiciones

1. Clonar ejemplos de Debezium de su repositorio oficial. [Debezium](https://github.com/debezium/debezium-examples/tree/master)

2. Los servicios necesarios a utilizar en este tutorial son:
    * Zookeper
    * Kafka
    * Base de Datos Mysql
    * Cliente de linea de comandos Mysql
    * Kafka Connect

3. Tener instalado Docker en su maquina

## Usando Mysql

1. Establecer la variable de entorno *DEBEZIUM_VERSION* en su version actual que en nuestro caso es la 1.5
    * *export DEBEZIUM_VERSION=1.5*

2. Inicializar los contenedores de Mysql, Kafka y Zookeper
    * *docker-compose -f ./docker-compose/docker-compose-mysql.yaml up*

Validamos que los servcios se encuentren inicializados
![image info](./images/image_1.png)

3. Inicializamos el MySQL Connector

> Después de iniciar los servicios Debezium y MySQL, está listo para implementar el conector Debezium MySQL para que pueda comenzar a monitorear la base de datos MySQL de muestra
---
> Al registrar el conector Debezium MySQL, el conector comenzará a monitorear los BinLogs del servidor de base de datos MySQL. La *binlog* registra todas las transacciones de la base de datos (como cambios en filas individuales y cambios en los esquemas). Cuando cambia una fila en la base de datos, Debezium genera un evento de cambio.

    * *curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register/register-mysql.json*

4. Consumimos mensajes desde un Topico Kafka

> docker-compose -f ./docker-compose/docker-compose-mysql.yaml exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic dbserver1.inventory.customers

Ejecutamos
> docker logs --follow **container_id**

5. Procedemos a modificar registros de la Base de Datos Inventory que tenemos en nuestro motor de Base de Datos MySQL

> docker-compose -f ./docker-compose/docker-compose-mysql.yaml exec mysql bash -c 'mysql -u $MYSQL_USER -p$MYSQL_PASSWORD inventory'

Ejecutar los siguiente comandos para validar:
* use inventory;
* show tables;
* SELECT * FROM customers;

![image info](./images/image_2.png)

6. Continuando el paso 5. Actualizamos un registro de la tabla Customer

> UPDATE customers SET first_name='Anne Marie' WHERE id=1004;

7. Validamos la Captura del Cambio llevado a cabo el cual se ve reflejado en el Topico descrito en el paso 4. Observar los agregados *before* , *after* y *op* el cual para actualizacion toma el valor *u*

![image info](./images/image_3.png)

8. Eliminamos un registro y observamos el cambio

> DELETE FROM addresses WHERE customer_id=1004;
> DELETE FROM customers WHERE id=1004;

Observar que el tag *op* es igual a *d*

9. Detener los servicios y destruir los recursos creados

> docker-compose -f ./docker-compose/docker-compose-mysql.yaml down

## Usando MySQL y Apicurio Registry

> Apicurio Registry es una API de código abierto y un registro de esquemas que, entre otras cosas, se puede utilizar para almacenar esquemas de registros de Kafka. Proporciona:

* Su propio convertidor Avro nativo y serializador Protobuf
* Un convertidor JSON que exporta su esquema al registro
* Una capa de compatibilidad con otros registros de esquemas como IBM o Confluent; se puede  utilizar con el convertidor Confluent Avro.

![image info](./images/docker-compose-mysql-apicurio.png)

Tomado de: <https://raw.githubusercontent.com/debezium/debezium-examples/master/tutorial/docker-compose-mysql-apicurio.png>

### Consumir Mensajes Formato JSON

1. Establecer la variable de entorno *DEBEZIUM_VERSION* en su version actual que en nuestro caso es la 1.5
    * *export DEBEZIUM_VERSION=1.5*

2. Inicializar los contenedores de Mysql, Kafka, Zookeper y Apicurio
    * *docker-compose -f ./docker-compose/docker-compose-mysql-apicurio.yaml up*

3. Inicializamos el MySQL Connector y especificamos la configuracion de Esquema Registry

* *curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register/register-mysql-apicurio-converter-json.json*

4. Consultamos el Schema Registry de la entidad Customers

> curl -X GET http://localhost:8080/api/artifacts/dbserver1.inventory.customers-value | jq .

5. Consumimos mensajes desde un Topico Kafka

> docker-compose -f ./docker-compose/docker-compose-mysql-apicurio.yaml exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic dbserver1.inventory.customers

Llevamos a cabo la actualizacion de un registro de la tabla Customer

> UPDATE customers SET first_name='Anne Marie' WHERE id=1004;

Como se observa el mensaje solo obtiene un Payload y una referencia al Id del esquema de registro que tiene asociado ese mensaje. Ya el schema no se infiera como la primera vez sino que esta asociado a un esquema de registro en Apicurio

![image info](./images/image_4.png)

Ahora si deseamos el esquema por el Id retornado para ver el Payload contra cual Schema Registry esta mapeado bastara con ejecutar:

> curl -X GET http://localhost:8080/api/ids/106 | jq .

![image info](./images/image_5.png)

---

### Consumir Mensajes Formato AVRO usando Apicurio Avro Converter

Mismos pasos que en Formato JSON pero en el paso inicializamos el MySQL Connector y especificamos la configuracion de Esquema Registry

* *curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register/register-mysql-apicurio-converter-avro.json*

---

### Consumir Mensajes Formato AVRO usando Confluent Avro Converter

Mismos pasos que en Formato JSON pero en el paso inicializamos el MySQL Connector y especificamos la configuracion de Esquema Registry

* *curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register/register-mysql-apicurio.json*
