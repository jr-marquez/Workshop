1. Levantar docker compose


2. Create elasticsearch template:

Elastic version 7 :
```bash
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
```

## Crear topic dentro de C3:
1. http://localhost:9021/

2. Create topic TRANSPORTE;


3. Cargar Datos
```bash
docker-compose exec broker bash -c 'kafka-producer-perf-test --topic TRANSPORTE --throughput 5 --payload-file /tmp/datos/santanderDatos.json --producer-props acks=all linger.ms=10 bootstrap.servers=localhost:9092 --num-records 100000'
```

## comandos ksqldb
1. 
```bash
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
```
2. 
```bash
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
```

3. 
```bash
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
```

## Creamos sink a Elastic
```bash
create sink connector sinkElastic with ( 
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
```

## Abrimos Kibana 
1. http://localhost:5601/
2. importamos dashboard Management --> Kibana --> Saved Objects --> import
3. Generamos indice con el timestamp

# Orden:
1. Producir mensajes a confluent cloud.
2. Crear KSQLDB flow
3. Cargar Connector a ElasticSearch en Connect (local)
4. Importar Dashboard de Kibana y ver resultados. (30 segundos atras en el tiempo y que refresque cada 15 secs)



