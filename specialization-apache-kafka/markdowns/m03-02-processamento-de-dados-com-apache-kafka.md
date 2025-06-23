# Processamento de Dados com Apache Kafka: Uma Visão Detalhada

Este documento apresenta uma explicação detalhada sobre o processamento de dados em streaming utilizando Apache Kafka e suas principais integrações com frameworks modernos como Apache Flink, Apache Spark Structured Streaming e ByteWax. O objetivo é fornecer uma visão clara dos conceitos, arquiteturas e exemplos práticos, incluindo códigos em PySpark para ilustrar a aplicação dos conceitos.

---

## 1. Introdução ao Processamento de Streaming

O processamento de dados em streaming permite analisar, transformar e agir sobre dados em tempo real, à medida que eles são gerados. No ecossistema de dados moderno, Apache Kafka se destaca como uma das principais plataformas para ingestão e transporte de eventos em tempo real.

### Principais Ferramentas de Processamento de Streaming

- **Apache Flink:** Plataforma robusta para processamento de streams, com suporte a APIs de alto e baixo nível, SQL e integração com diversos sistemas.
- **Apache Spark Structured Streaming:** Evolução do Spark Streaming, baseada em micro-batches, com APIs DataFrame e integração nativa com Kafka.
- **ByteWax:** Biblioteca Python para processamento de streams, simples e eficiente, com suporte a operadores de janela, mapeamento e integração com Kafka.

---

## 2. Apache Flink: Arquitetura e Aplicação

### 2.1. Arquitetura do Flink

O Flink é composto por três principais componentes:

- **Client:** Envia jobs para o cluster.
- **JobManager:** Gerencia a execução dos jobs.
- **TaskManager (Worker):** Executa as tarefas distribuídas.

O Flink permite distribuir o processamento em múltiplos slots, garantindo alta performance e escalabilidade.

### 2.2. APIs do Flink

- **DataSet API:** Processamento batch.
- **DataStream API:** Processamento streaming.
- **Table API e SQL:** Processamento declarativo com SQL.

### 2.3. Exemplo de Uso: PyFlink com Kafka

```python
from pyflink.table import EnvironmentSettings, TableEnvironment

# Configuração do ambiente
env_settings = EnvironmentSettings.in_streaming_mode()
table_env = TableEnvironment.create(env_settings)

# Definição da tabela Kafka Source
table_env.execute_sql("""
CREATE TABLE stock_data (
    symbol STRING,
    shares INT,
    price DOUBLE,
    transaction_date TIMESTAMP(3),
    WATERMARK FOR transaction_date AS transaction_date - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'stock_topic',
    'properties.bootstrap.servers' = 'localhost:9092',
    'format' = 'json',
    'scan.startup.mode' = 'latest-offset'
)
""")

# Consulta SQL para agregação por janela de 1 hora
result = table_env.sql_query("""
SELECT
    symbol,
    TUMBLE_START(transaction_date, INTERVAL '1' HOUR) as window_start,
    SUM(shares) as total_shares,
    AVG(price) as avg_price
FROM stock_data
GROUP BY symbol, TUMBLE(transaction_date, INTERVAL '1' HOUR)
""")

# Impressão do resultado (para fins de exemplo local)
table_env.to_pandas(result).head()
```

---

## 3. Apache Spark Structured Streaming

O Spark Structured Streaming trabalha com micro-batches, processando dados em pequenos lotes com baixa latência (tipicamente 100ms). Ele utiliza DataFrames e permite integração direta com Kafka.

### 3.1. Exemplo de Uso: PySpark Structured Streaming

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col, window
from pyspark.sql.types import StructType, StringType, IntegerType, DoubleType, TimestampType

# Inicialização da sessão Spark
spark = SparkSession.builder.appName("KafkaSparkStreaming").getOrCreate()

# Esquema dos dados
schema = StructType() \
    .add("symbol", StringType()) \
    .add("shares", IntegerType()) \
    .add("price", DoubleType()) \
    .add("transaction_date", TimestampType())

# Leitura do stream Kafka
df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "stock_topic") \
    .load()

# Conversão e seleção dos campos
json_df = df.select(from_json(col("value").cast("string"), schema).alias("data")).select("data.*")

# Agregação por janela de 1 hora
agg_df = json_df.groupBy(
    window(col("transaction_date"), "1 hour"),
    col("symbol")
).agg(
    {"shares": "sum", "price": "avg"}
)

# Escrita do resultado em console (pode ser Kafka, Delta, etc.)
query = agg_df.writeStream \
    .outputMode("update") \
    .format("console") \
    .start()

query.awaitTermination()
```

---

## 4. ByteWax: Processamento Simples em Python

O ByteWax é uma biblioteca Python para processamento de streams, ideal para aplicações simples e prototipagem rápida.

### 4.1. Exemplo de Uso: ByteWax

```python
from bytewax.dataflow import Dataflow
from bytewax.inputs import KafkaInputConfig
from bytewax.outputs import KafkaOutputConfig

def process(record):
    # Exemplo de processamento: filtrar transações acima de 10.000
    data = record.value
    if data["transaction_value"] > 10000:
        return data
    return None

flow = Dataflow()
flow.input("input", KafkaInputConfig("localhost:9092", "stock_topic"))
flow.map(process)
flow.output("output", KafkaOutputConfig("localhost:9092", "filtered_stock_topic"))
```

---

## 5. Design Patterns em Processamento de Streaming

### 5.1. Simple Event Processing

Processamento simples de eventos, como filtragem e mapeamento direto.

### 5.2. Stateful Processing

Agregações, joins e operações que dependem de estado, utilizando State Stores (Flink, Spark, ByteWax).

### 5.3. Multiphase Processing

Várias fases de processamento, cada uma em um tópico Kafka diferente, permitindo pipelines reativos e escaláveis.

### 5.4. Stream-Table Join

Joins entre streams e tabelas, comuns em casos de enriquecimento de dados.

### 5.5. Out-of-Sequence Handling

Tratamento de eventos fora de ordem, utilizando janelas e tolerância a atrasos (late arrivals).

---

## 6. Recomendações de Uso

- **Flink:** Para casos enterprise, alta performance, múltiplos tópicos e SQL avançado.
- **Spark Structured Streaming:** Para integração com Delta Lake, pipelines analíticos e processamento batch/streaming híbrido.
- **ByteWax:** Para protótipos rápidos, aplicações Python puras e casos simples.

---

## 7. Conclusão

O ecossistema de processamento de streaming evoluiu rapidamente, oferecendo múltiplas opções para diferentes necessidades. Apache Kafka, aliado a frameworks como Flink, Spark e ByteWax, permite construir pipelines robustos, escaláveis e reativos para atender às demandas modernas de dados em tempo real.

---
