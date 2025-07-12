# Entrega de Dados com Apache Kafka: Guia Detalhado

## Introdução

Neste documento, exploramos o processo de consumo de dados no Apache Kafka, abordando conceitos fundamentais, práticas recomendadas e exemplos práticos com PySpark para pipelines reais.

---

## 1. Fundamentos do Consumo no Kafka

### 1.1. Conceitos Básicos

```mermaid
flowchart LR
    T[Tópico] -->|Dividido em| P[Partições]
    P -->|Armazenadas em| B[Broker]
    C[Consumer Group] -->|Consome| P
```

- **Tópico**: Canal lógico onde as mensagens são publicadas.
- **Partição**: Subdivisão de um tópico, permitindo paralelismo.
- **Broker**: Servidor Kafka responsável por armazenar dados.
- **Consumer Group**: Agrupamento de consumidores que trabalham juntos para ler dados de um tópico.

### 1.2. Como Funciona o Consumo

```mermaid
sequenceDiagram
    participant App as Aplicação
    participant Kafka as Cluster Kafka
    App->>Kafka: Conecta e define group.id
    Kafka->>App: Distribui partições
    App->>Kafka: Consome dados das partições
```

- A aplicação se conecta ao cluster Kafka e define um `group.id`.
- O Kafka distribui as partições entre as instâncias do grupo.
- Cada instância lê dados de uma ou mais partições.

**Importante:** Não existe consumidor sem `Consumer Group`. Se não for informado, o Kafka cria um automaticamente.

---

## 2. Consumer Groups e Partições

### 2.1. Distribuição de Partições

```mermaid
flowchart LR
    P1[Partição 1] --> CG1[Instância 1]
    P2[Partição 2] --> CG2[Instância 2]
    P3[Partição 3] --> CG3[Instância 3]
    subgraph Consumer Group
        CG1
        CG2
        CG3
    end
```

- Instâncias são distribuídas conforme o número de partições.
- Instâncias excedentes ficam ociosas; instâncias insuficientes consomem múltiplas partições.

### 2.2. Rebalanceamento

```mermaid
sequenceDiagram
    participant Kafka
    participant Inst1 as Instância 1
    participant Inst2 as Instância 2
    Inst2->>Kafka: Sai do grupo
    Kafka-->>Inst1: Pausa consumo
    Kafka-->>Inst1: Redistribui partições
    Kafka-->>Inst1: Retoma consumo
```

- O Kafka pausa o consumo, redistribui partições e retoma após o balanceamento.

**Atenção:** Adicionar/remover partições ou instâncias pode causar indisponibilidade temporária.

---

## 3. Configurações Essenciais do Consumer

```mermaid
graph TD
    A[bootstrap.servers] -->|Endereço| Kafka
    B[group.id] -->|Identificador| ConsumerGroup
    C[key/value.deserializer] -->|Deserialização| Mensagens
    D[auto.offset.reset] -->|Início/Fim| Consumo
    E[enable.auto.commit] -->|Commit automático| Offset
```

- **bootstrap.servers**: Endereço dos brokers Kafka.
- **group.id**: Identificador do grupo de consumidores.
- **key.deserializer/value.deserializer**: Como deserializar as mensagens.
- **auto.offset.reset**: Define se começa a ler do início (`earliest`) ou do fim (`latest`).
- **enable.auto.commit**: Se o offset será comitado automaticamente.

---

## 4. Offset e Commit

### 4.1. O que é Offset?

```mermaid
flowchart LR
    M1[Mensagem 1] -->|Offset 0| P[Partição]
    M2[Mensagem 2] -->|Offset 1| P
    M3[Mensagem 3] -->|Offset 2| P
```

- O **offset** é o identificador sequencial de cada mensagem dentro de uma partição.

### 4.2. Commit de Offset

- **Auto Commit**: O Kafka comita o offset automaticamente em intervalos.
- **Manual Commit**: A aplicação controla quando o offset é comitado.

#### Exemplo de Commit Manual em PySpark

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("KafkaConsumerExample") \
    .getOrCreate()

df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "meu-topico") \
    .option("startingOffsets", "earliest") \
    .load()

from pyspark.sql.functions import col
df = df.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")

query = df.writeStream \
    .format("console") \
    .option("checkpointLocation", "/tmp/checkpoint") \
    .start()

query.awaitTermination()
```

> **Nota:** O Spark gerencia os commits de offset automaticamente via checkpoint.

---

## 5. Lag e Monitoramento

```mermaid
flowchart LR
    A[Último Offset Disponível] -->|Diferença| B[Offset Lido pelo Consumidor]
    B -->|Lag| C[Monitoramento]
```

- **Lag**: Diferença entre o último offset disponível e o offset lido pelo consumidor.
- Lag alto pode indicar problemas de performance ou falhas no consumidor.

#### Exemplo de Monitoramento de Lag

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group meu-consumer-group
```

---

## 6. Serialização e Schema Registry

```mermaid
flowchart LR
    J[JSON] -->|Opcional| SR[Schema Registry]
    A[Avro] -->|Recomendado| SR
    P[Protobuf] -->|Recomendado| SR
```

- **JSON**: Não exige Schema Registry, mas pode ser usado para padronização.
- **Avro/Protobuf**: Recomendado usar Schema Registry para garantir compatibilidade e evolução de schemas.

#### Exemplo de Consumo de Avro com PySpark

```python
df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "meu-topico-avro") \
    .load()

from pyspark.sql.avro.functions import from_avro

avro_schema = '''
{
  "type": "record",
  "name": "Exemplo",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "valor", "type": "string"}
  ]
}
'''

df = df.withColumn("dados", from_avro(col("value"), avro_schema))
df.select("dados.*").writeStream.format("console").start().awaitTermination()
```

---

## 7. Boas Práticas

```mermaid
flowchart TD
    A[Dimensione partições e instâncias]
    B[Defina group.id e client.id]
    C[Monitore o lag]
    D[Use commit manual quando necessário]
    E[Utilize Schema Registry para formatos binários]
    A --> B --> C --> D --> E
```

- Dimensione corretamente o número de partições e instâncias.
- Sempre defina `group.id` e `client.id`.
- Monitore o lag dos consumidores.
- Use commit manual para garantir processamento exato quando necessário.
- Utilize Schema Registry para formatos binários (Avro/Protobuf).

---

## 8. Resumo Visual

```mermaid
flowchart LR
    A[Producer] -->|Envia mensagem| B[Kafka Topic]
    B -->|Partições| C[Consumer Group]
    C -->|Instâncias| D[Aplicações]
    D -->|Processa| E[Pipeline de Dados]
```

---

## 9. Conclusão

O consumo de dados no Kafka é altamente escalável e flexível, mas exige atenção ao gerenciamento de offsets, rebalanceamento e monitoramento de lag. Utilizando PySpark, é possível integrar o consumo de dados Kafka em pipelines robustos de processamento distribuído.

---

## 10. Referências

- [Documentação Oficial do Apache Kafka](https://kafka.apache.org/documentation/)
- [Structured Streaming + Kafka Integration Guide (PySpark)](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [Confluent Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)

---