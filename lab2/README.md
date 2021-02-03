  
# Documentation


## Configuración inicial

Antes de ejecutar el yml poner los datos del bucket de S3:
```bash
export BUCKET_NAME=rmarquez 
export REGION=eu-west-2
export AWS_ACCESS_KEY_ID=*******  
export AWS_SECRET_ACCESS_KEY=*********
```
Si no se quiere hacer esta parte de la demo comentar las siguientes lineas de docker-compose.yml dentro del servicio broker:
```bash
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_CONFLUENT_LICENSE_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CONFLUENT_BALANCER_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9101
      KAFKA_CONFLUENT_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: broker:29092
      CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
      CONFLUENT_METRICS_ENABLE: 'true'
      CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'
      #KAFKA_CONFLUENT_TIER_FEATURE: 'true'
      #KAFKA_CONFLUENT_TIER_ENABLE: 'true'
      #KAFKA_CONFLUENT_TIER_BACKEND: S3
      #KAFKA_CONFLUENT_TIER_S3_BUCKET: ${BUCKET_NAME}
      #KAFKA_CONFLUENT_TIER_S3_REGION: ${REGION}
      KAFKA_CONFLUENT_TIER_LOCAL_HOTSET_MS: 60000 # hotset of 1 minute
      KAFKA_CONFLUENT_TIER_TOPIC_DELETE_CHECK_INTERVAL_MS: 300000 # check every 5 min for topic deletion
      KAFKA_CONFLUENT_TIER_METADATA_REPLICATION_FACTOR: 1 # only one broker, so replication factor is one
      KAFKA_LOG_SEGMENT_BYTES: 1048576000 # 100 MB log segments
      KAFKA_LOG_RETENTION_MS: -1 # eternal
      #KAFKA_LOG_RETENTION_MS: 600000 # 10 minute retention
      #AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      #AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
```


```bash
docker-compose up -d

docker exec -it postgres /bin/bash

psql -U postgres-user customers

CREATE TABLE customers (id TEXT PRIMARY KEY, name TEXT, age INT);
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
```

```bash
docker exec -it mongo /bin/bash

mongo -u $MONGO_INITDB_ROOT_USERNAME -p mongo-pw admin

rs.initiate()

use config

db.createRole({
    role: "dbz-role",
    privileges: [
        {
            resource: { db: "config", collection: "system.sessions" },
            actions: [ "find", "update", "insert", "remove" ]
        }
    ],
    roles: [
       { role: "dbOwner", db: "config" },
       { role: "dbAdmin", db: "config" },
       { role: "readWrite", db: "config" }
    ]
})

use admin

db.createUser({
  "user" : "dbz-user",
  "pwd": "dbz-pw",
  "roles" : [
    {
      "role" : "root",
      "db" : "admin"
    },
    {
      "role" : "readWrite",
      "db" : "logistics"
    },
    {
      "role" : "dbz-role",
      "db" : "config"
    }
  ]
})

use logistics

db.createCollection("orders")

db.createCollection("shipments")
```

1. Si tenemos Python ejectuar loadOrders.py. Sino dentro  de mongo:
```bash
db.orders.insert({"customer_id": "2", "order_id": "13", "price": 50.50, "currency": "usd", "ts": "2020-04-03T11:20:00"})
db.orders.insert({"customer_id": "7", "order_id": "29", "price": 15.00, "currency": "aud", "ts": "2020-04-02T12:36:00"})
db.orders.insert({"customer_id": "5", "order_id": "17", "price": 25.25, "currency": "eur", "ts": "2020-04-02T17:22:00"})
db.orders.insert({"customer_id": "5", "order_id": "15", "price": 13.75, "currency": "usd", "ts": "2020-04-03T02:55:00"})
db.orders.insert({"customer_id": "7", "order_id": "22", "price": 29.71, "currency": "aud", "ts": "2020-04-04T00:12:00"})
```
## Demo Storage (tienen que tener el API key y Secret Correctamente configurados)


1. 
```bash
docker-compose exec broker kafka-topics 
    --bootstrap-server localhost:9092 
    --create 
    --topic test-topic 
    --partitions 1
```

2. 
```bash
docker-compose exec broker kafka-producer-perf-test --topic test-topic 
    --num-records 5000000 
    --record-size 5000 
    --throughput -1 
    --producer-props 
        acks=all 
        bootstrap.servers=localhost:9092 
        batch.size=8196
```

## Schema Validation:

1. Crear el topic schemaValidation con schema validation enable

```bash
docker-compose exec broker bash -c 'kafka-console-producer --broker-list localhost:9092 --topic testschema --property parse.key=true --property key.separator=,'
```

## Parte Streaming:


1. 
```bash 
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```

2. 
```bash
SET 'auto.offset.reset' = 'earliest';
```

3. 
```bash
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
```

4. 
```bash
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
```

5. 
```bash
CREATE STREAM customers WITH ( 
    kafka_topic = 'customers.public.customers', 
    value_format = 'avro' 
); 
```

6. 
```bash
CREATE STREAM orders WITH ( 
    kafka_topic = 'my-replica-set.logistics.orders', 
    value_format = 'avro', 
    timestamp = 'ts', 
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss' 
); 
```

7. 
```bash
CREATE STREAM shipments WITH ( 
    kafka_topic = 'my-replica-set.logistics.shipments', 
    value_format = 'avro', 
    timestamp = 'ts', 
    timestamp_format = 'yyyy-MM-dd''T''HH:mm:ss' 
); 
```

8. 
```bash
CREATE TABLE customers_by_key AS 
    SELECT id, 
           latest_by_offset(name) AS name, 
           latest_by_offset(age) AS age 
    FROM customers 
    GROUP BY id 
    EMIT CHANGES; 
```
### Ver como cambia en ktable
1. 
```bash
docker exec -it postgres /bin/bash
```
2. 
```bash
psql -U postgres-user customers
```
3. 
```bash
update customers set age=99 where name='ramon';
```

### Dentro de ksqldb
1. 
```bash
select * from customers_by_key emit changes;
```
2. 
```bash
select * from customers_by_key where id='4';
```

3. 
```bash
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
```

4. 
```bash
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
```



