1. Levantar docker compose


2. Create elasticsearch template:

Elastic version 7 :

curl -XPUT "http://localhost:9200/_template/iotsantander/" -H 'Content-Type: application/json' -d'
{
  "index_patterns": "*",
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
      "dynamic_templates": [
        {
          "dates": {
            "match": "*TIMESTAMP",
            "mapping": {
              "type": "date",
              "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
            }
          }
        },
        {
          "geopoint": {
            "match": "*LOCATION",
            "mapping": {
              "type": "geo_point"
            }
          }
        },
        {
          "geopoint2": {
            "match": "*location",
            "mapping": {
              "type": "geo_point"
            }
          }
        },
        {
          "non_analysed_string_template": {
            "match": "identifier,temperatura_promedio,ozono_promedio",
            "match_mapping_type": "string",
            "mapping": {
              "type": "keyword"
            }
          }
        }
      ]
  }
}'


Cambia también el postman, ver archivos postman.
V 7+

{
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "connection.url": "http://localhost:9200",
    "tasks.max": "1",
    "topics": "TRANSPORTE_FILTRADO_JSON",
    "type.name": "_doc",
    "name": "elastic_sink_test",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "key.converter.schemas.enable": "false",
    "schema.ignore" : "true",
    "key.ignore": "true",
    "topic.index.map": "TRANSPORTE_FILTRADO_JSON:transporte"
}


Crear topic:

2. Create topic TRANSPORTE;


Producer:
kafka-producer-perf-test \
    --topic TRANSPORTE \
    --throughput 5 \
    --producer.config /Users/ramon/Desktop/config \
    --payload-file ../data/santanderDatos.json \
    --producer-props acks=all linger.ms=10 \
    --num-records 100000 

Datos.json en el folder

El producer config es para el api key y secret.

On premise
kafka-producer-perf-test \
    --topic TRANSPORTE \
    --throughput 5 \
    --payload-file santanderDatos.json \
    --producer-props acks=all linger.ms=10 bootstrap.servers=localhost:9092 \
    --num-records 100000 


KSQL:
CREATE STREAM TRANSPORTE_PUBLICO
(
    PARTICLES STRING,
    NO2 STRING,
    TYPE STRING,
    LATITUDE STRING,
    temperature STRING,
    altitude STRING,
    speed STRING,
    modified STRING,
    identifier STRING,
    longitude STRING,
    odometer STRING,
    course STRING,
    ozone STRING
)
WITH (
    KAFKA_TOPIC = 'TRANSPORTE',
    VALUE_FORMAT = 'JSON'
);

CREATE STREAM TRANSPORTE_PUBLICO_KEY 
WITH (
    KAFKA_TOPIC = 'TRANSPORTE_PUBLICO_KEYED',
    VALUE_FORMAT = 'JSON'
) AS
SELECT
    NO2 ,
    TYPE ,
    LATITUDE ,
    temperature ,
    modified ,
    identifier ,
    longitude ,
    ozone 
FROM TRANSPORTE_PUBLICO
PARTITION BY identifier;

Cambiar a Avro:


CREATE TABLE TRANSPORTE_FILTRADO_JSON
WITH (
    KAFKA_TOPIC = 'TRANSPORTE_FILTRADO_JSON',
    VALUE_FORMAT = 'JSON',
    TIMESTAMP_FORMAT='yyyy-MM-dd HH:mm:ss',
    TIMESTAMP='timestamp'
) AS
SELECT
    identifier,
    avg(cast(TEMPERATURE as DOUBLE)) as temperatura_promedio,
    avg(cast(OZONE as double)) as ozono_promedio,
    latest_by_offset(LATITUDE  +','+ longitude) as LOCATION,
    latest_by_offset(TIMESTAMPTOSTRING(ROWTIME, 'yyyy-MM-dd HH:mm:ss')) AS timestamp
FROM TRANSPORTE_PUBLICO_KEY
group by identifier
having avg(cast(OZONE as double)) < 500
emit changes;


create source connector sinkElastic with (
    "connector.class"= 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    "connection.url" = 'http://es01:9200',
    "tasks.max" = '1',
    "topics" = 'TRANSPORTE_FILTRADO_JSON',
    "type.name" = '_doc',
    "value.converter" = 'org.apache.kafka.connect.json.JsonConverter',
    "value.converter.schemas.enable" = 'false',
    "key.converter"='org.apache.kafka.connect.storage.StringConverter',
    "key.converter.schemas.enable"='false',
    "schema.ignore" ='true',
    "key.ignore"='true',
    "topic.index.map"='TRANSPORTE_FILTRADO_JSON:transporte'
);


Orden:

Producir mensajes a confluent cloud.
Crear KSQLDB flow
Cargar Connector a ElasticSearch en Connect (local)
Importar Dashboard de Kibana y ver resultados.


En demos:
Borrar connector en Connect
Mostrar flujo ksql
Cargar connector para que cargue a elastic
Mostrar kibana (30 segundos atras en el tiempo y que refresque cada 15 secs)



