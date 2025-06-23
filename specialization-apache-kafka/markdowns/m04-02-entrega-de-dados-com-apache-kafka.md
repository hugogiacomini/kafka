# Entrega de Dados com Apache Kafka: Uma Abordagem Detalhada

## Introdução

Neste documento, vamos explorar em detalhes a entrega de dados utilizando o Apache Kafka, abordando desde conceitos fundamentais até exemplos práticos com código PySpark. O objetivo é fornecer uma compreensão profunda sobre como conectar, transformar e entregar dados em pipelines modernos de dados, utilizando conectores, formatos e ferramentas de análise em tempo real.

---

## Anatomia do Kafka Connect

O **Kafka Connect** é um framework robusto para integrar sistemas externos ao Kafka, seja para ingestão (Source) ou entrega (Sink/Sync) de dados. A estrutura básica de um conector envolve:

- **Conector**: Responsável por conectar ao sistema externo.
- **Transformação (SMT - Single Message Transform)**: Permite modificar mensagens em trânsito.
- **Conversão**: Adapta o formato dos dados (JSON, Avro, Protobuf, String, etc).

> **Dica:** Sempre consulte a documentação oficial do conector para entender suas capacidades e limitações.

---

## Principais Conectores de Sink

### JDBC Sink Connector

Permite entregar dados do Kafka para bancos relacionais (PostgreSQL, SQL Server, MySQL, etc). Suporta configurações como:

- **pk.mode**: Define como a chave primária é gerada (por exemplo, a partir do valor ou da chave da mensagem).
- **auto.create** e **auto.evolve**: Controlam a criação e evolução automática de tabelas.
- **CDC (Change Data Capture)**: Permite refletir operações de deleção e atualização no destino.

**Exemplo de configuração:**

```json
{
    "name": "jdbc-sink-connector",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
        "topics": "meu-topico",
        "connection.url": "jdbc:postgresql://host:5432/db",
        "auto.create": "true",
        "auto.evolve": "true",
        "pk.mode": "record_value",
        "insert.mode": "upsert"
    }
}
```

---

### AWS S3 Sink Connector

Permite entregar dados do Kafka para buckets S3 (ou compatíveis, como MinIO), em formatos como Parquet, JSON ou Avro. É fundamental configurar corretamente o particionamento e o tamanho dos arquivos para evitar o problema de "small files".

**Configurações importantes:**

- `flush.size`: Número de registros por arquivo.
- `rotate.interval.ms`: Intervalo de tempo para rotação dos arquivos.
- `partitioner.class`: Define a estratégia de particionamento (por tempo, campo, etc).

**Exemplo de configuração:**

```json
{
    "name": "s3-sink-connector",
    "config": {
        "connector.class": "io.confluent.connect.s3.S3SinkConnector",
        "topics": "meu-topico",
        "s3.bucket.name": "meu-bucket",
        "s3.region": "us-east-1",
        "flush.size": "10000",
        "rotate.interval.ms": "600000",
        "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
        "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
        "path.format": "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH",
        "locale": "pt_BR",
        "timezone": "America/Sao_Paulo"
    }
}
```

---

## Exemplos Práticos com PySpark

### Leitura de Dados do S3 em Parquet

Após a entrega dos dados no S3, é comum utilizar o PySpark para processá-los.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
        .appName("Leitura S3 Parquet") \
        .getOrCreate()

# Leitura dos arquivos Parquet particionados por ano, mês, dia e hora
df = spark.read.parquet("s3a://meu-bucket/year=*/month=*/day=*/hour=*")

df.printSchema()
df.show(5)
```

### Processamento e Escrita de Dados no Kafka

Você pode processar dados e enviar resultados para um tópico Kafka:

```python
from pyspark.sql.functions import to_json, struct

# Supondo que df seja o DataFrame processado
df_to_kafka = df.select(to_json(struct("*")).alias("value"))

df_to_kafka.write \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "localhost:9092") \
        .option("topic", "topico-processado") \
        .save()
```

---

## User-Facing Analytics com Apache Pinot

O **Apache Pinot** é uma solução analítica colunar, otimizada para consultas em tempo real com alta concorrência e baixa latência. Ele consome dados diretamente do Kafka, sem necessidade de conectores intermediários.

### Características do Pinot

- **Ingestão em tempo real e offline**
- **Segmentação baseada em timestamp**
- **Índices plugáveis (ex: StarTree Index)**
- **Alta performance para queries analíticas**

### Exemplo de Esquema e Tabela Pinot

```json
// Exemplo de schema
{
    "schemaName": "stock_transactions",
    "dimensionFieldSpecs": [
        {"name": "id", "dataType": "INT"},
        {"name": "user_id", "dataType": "INT"},
        {"name": "symbol", "dataType": "STRING"}
    ],
    "metricFieldSpecs": [
        {"name": "shares", "dataType": "INT"},
        {"name": "purchase", "dataType": "DOUBLE"}
    ],
    "dateTimeFieldSpecs": [
        {
            "name": "update_at",
            "dataType": "LONG",
            "format": "1:MILLISECONDS:EPOCH",
            "granularity": "1:DAYS"
        }
    ]
}
```

```json
// Exemplo de tabela
{
    "tableName": "stock_transactions",
    "tableType": "REALTIME",
    "segmentsConfig": {
        "timeColumnName": "update_at",
        "retentionTimeUnit": "DAYS",
        "retentionTimeValue": "60"
    },
    "tableIndexConfig": {
        "invertedIndexColumns": ["id"]
    },
    "ingestionConfig": {
        "streamIngestionConfig": {
            "streamConfigMaps": [
                {
                    "streamType": "kafka",
                    "stream.kafka.topic.name": "topico-processado",
                    "stream.kafka.broker.list": "localhost:9092",
                    "stream.kafka.consumer.type": "lowlevel",
                    "stream.kafka.decoder.class.name": "org.apache.pinot.plugin.stream.kafka.KafkaJSONMessageDecoder"
                }
            ]
        }
    }
}
```

---

## Consultas Analíticas em Tempo Real

Com o Pinot, é possível executar queries SQL analíticas com alta performance, mesmo com milhões de registros.

**Exemplo de query:**

```sql
SELECT symbol, SUM(shares) AS total_shares
FROM stock_transactions
WHERE update_at >= 1680000000000
GROUP BY symbol
ORDER BY total_shares DESC
LIMIT 10;
```

---

## Boas Práticas e Dicas

- **Evite small files**: Ajuste `flush.size` e `rotate.interval.ms` nos conectores S3.
- **Particionamento inteligente**: Use particionamento por tempo ou campo para facilitar consultas e organização.
- **Monitoramento**: Utilize logs e métricas dos conectores e do Kafka Connect.
- **Documentação**: Sempre consulte a documentação oficial dos conectores e ferramentas.

---

## Conclusão

A entrega de dados com Apache Kafka, conectores Sink e ferramentas como PySpark e Apache Pinot permite construir pipelines de dados robustos, escaláveis e analíticos em tempo real. Com as práticas e exemplos apresentados, você estará apto a implementar soluções modernas de streaming analytics.

---

## Referências

- [Documentação Kafka Connect](https://kafka.apache.org/documentation/#connect)
- [Confluent S3 Sink Connector](https://docs.confluent.io/kafka-connect-s3/current/index.html)
- [Apache Pinot](https://docs.pinot.apache.org/)
- [PySpark Documentation](https://spark.apache.org/docs/latest/api/python/)

