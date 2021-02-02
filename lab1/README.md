  
# Documentation


Seguir :
https://docs.ksqldb.io/en/latest/tutorials/etl/?_ga=2.60504622.223966942.1601300126-1324303483.1594711257&_gac=1.54318426.1601363011.Cj0KCQjwtsv7BRCmARIsANu-CQdSYWeeFNn8jPsc-EMCpmoN6HTyEDateRIZgbUOpW-EM0vV8FGTrxwaAqphEALw_wcB

Antes de ejecutar el yml:

export AWS_ACCESS_KEY_ID=******
export AWS_SECRET_ACCESS_KEY=*****
export BUCKET_NAME=rmarquez
export REGION=eu-west-2

ejectuar loadOrders.py

1) Para Storage
_____________

docker-compose exec broker kafka-topics \
    --bootstrap-server localhost:9092 \
    --create \
    --topic test-topic \
    --partitions 1

 docker-compose exec broker kafka-producer-perf-test --topic test-topic \
    --num-records 5000000 \
    --record-size 5000 \
    --throughput -1 \
    --producer-props \
        acks=all \
        bootstrap.servers=localhost:9092 \
        batch.size=8196

Schema Validation:
__________________
Recordar crear el topic schemaValidation con schema validation enable al final

kafka-console-producer --broker-list localhost:9092 --topic testschema --property parse.key=true --property key.separator=,

Parte Streaming:
__________________

1)
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088

2)

SET 'auto.offset.reset' = 'earliest';

3) 

CREATE SOURCE CONNECTOR customers_reader WITH (
    'connector.class' = 'io.debezium.connector.postgresql.PostgresConnector',
    'database.hostname' = 'postgres',
    'database.port' = '5432',
    'database.user' = 'postgres-user',
    'database.password' = 'postgres-pw',
    'database.dbname' = 'customers',
    'database.server.name' = 'customers',
    'table.whitelist' = 'public.customers',
    'transforms' = 'unwrap',
    'transforms.unwrap.type' = 'io.debezium.transforms.ExtractNewRecordState',
    'transforms.unwrap.drop.tombstones' = 'false',
    'transforms.unwrap.delete.handling.mode' = 'rewrite'
);

4)

CREATE SOURCE CONNECTOR logistics_reader WITH (
    'connector.class' = 'io.debezium.connector.mongodb.MongoDbConnector',
    'mongodb.hosts' = 'mongo:27017',
    'mongodb.name' = 'my-replica-set',
    'mongodb.authsource' = 'admin',
    'mongodb.user' = 'dbz-user',
    'mongodb.password' = 'dbz-pw',
    'collection.whitelist' = 'logistics.*',
    'transforms' = 'unwrap',
    'transforms.unwrap.type' = 'io.debezium.connector.mongodb.transforms.ExtractNewDocumentState',
    'transforms.unwrap.drop.tombstones' = 'false',
    'transforms.unwrap.delete.handling.mode' = 'drop',
    'transforms.unwrap.operation.header' = 'true'
);

CREATE STREAM customers WITH (
    kafka_topic = 'customers.public.customers',
    value_format = 'avro'
);

CREATE STREAM orders WITH (
    kafka_topic = 'my-replica-set.logistics.orders',
    value_format = 'avro',
    timestamp = 'ts',
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
);

CREATE STREAM shipments WITH (
    kafka_topic = 'my-replica-set.logistics.shipments',
    value_format = 'avro',
    timestamp = 'ts',
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss'
);

CREATE TABLE customers_by_key AS
    SELECT id,
           latest_by_offset(name) AS name,
           latest_by_offset(age) AS age
    FROM customers
    GROUP BY id
    EMIT CHANGES;

Pasos
1)docker exec -it postgres /bin/bash
2)psql -U postgres-user customers
update customers set age=99 where name='ramon';


select * from customers_by_key emit changes;
select * from customers_by_key where id='4';

CREATE STREAM enriched_orders AS
    SELECT o.order_id,
           o.price,
           o.currency,
           c.id AS customer_id,
           c.name AS customer_name,
           c.age AS customer_age
    FROM orders AS o
    LEFT JOIN customers_by_key c
    ON o.customer_id = c.id
    EMIT CHANGES;

CREATE STREAM shipped_orders WITH (
    kafka_topic = 'shipped_orders'
)   AS
    SELECT o.order_id,
           s.shipment_id,
           o.customer_id,
           o.customer_name,
           o.customer_age,
           s.origin,
           o.price,
           o.currency
    FROM enriched_orders AS o
    INNER JOIN shipments s
    WITHIN 7 DAYS
    ON s.order_id = o.order_id
    EMIT CHANGES;




sqls:
 qdocker exec -it mongo /bin/bash

INSERT INTO customers (id, name, age) VALUES ('1', 'fred', 34);
INSERT INTO customers (id, name, age) VALUES ('2', 'sue', 25);
INSERT INTO customers (id, name, age) VALUES ('3', 'bill', 51);
INSERT INTO customers (id, name, age) VALUES ('4', 'ramon', 36);
INSERT INTO customers (id, name, age) VALUES ('5', 'juan', 28);
INSERT INTO customers (id, name, age) VALUES ('6', 'federico', 56);
INSERT INTO customers (id, name, age) VALUES ('7', 'luis', 37);
INSERT INTO customers (id, name, age) VALUES ('8', 'pedro', 27);
INSERT INTO customers (id, name, age) VALUES ('9', 'pablo', 59);
INSERT INTO customers (id, name, age) VALUES ('10', 'peter', 59);

