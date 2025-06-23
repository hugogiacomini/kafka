# Ingestão de Dados com Apache Kafka Connect

## Introdução

Neste documento, vamos detalhar o funcionamento do **Kafka Connect**, um framework open source de integração para ingestão e exportação de dados no ecossistema Apache Kafka. O objetivo é explicar os conceitos, arquitetura, boas práticas e exemplos práticos, incluindo trechos de código em PySpark para ilustrar como consumir e processar dados ingeridos via Kafka Connect.

---

## O que é o Kafka Connect?

O **Kafka Connect** é um framework de integração que permite conectar fontes de dados externas (bancos de dados, sistemas de arquivos, APIs, etc.) ao Kafka, bem como exportar dados do Kafka para outros sistemas. Ele utiliza o conceito de **conectores** (connectors), que podem ser de dois tipos:

- **Source Connector**: Lê dados de uma fonte externa e publica no Kafka.
- **Sink Connector**: Lê dados do Kafka e envia para um destino externo.

Os conectores são plugins que encapsulam toda a lógica de integração, tornando o processo declarativo e sem necessidade de programação customizada.

---

## Arquitetura do Kafka Connect

A arquitetura do Kafka Connect é composta por:

- **Cluster de Connect**: Um ou mais nós (workers) que executam os conectores.
- **Workers**: Nós que executam tarefas (tasks) dos conectores.
- **Tasks**: Unidades de execução paralela de um conector.
- **Configurações**: Arquivos declarativos (JSON ou propriedades) que definem como o conector irá operar.

### Modos de Deploy

- **Standalone**: Útil para desenvolvimento, roda em um único worker e armazena offsets/configurações localmente.
- **Distributed**: Recomendado para produção, permite alta disponibilidade e distribuição de tarefas entre múltiplos workers, armazenando offsets/configurações em tópicos internos do Kafka.

---

## Anatomia de um Conector

Um conector é definido por:

- **Classe do Conector**: Define o tipo (ex: JDBC, MongoDB, Elasticsearch).
- **Transformações (SMT)**: Permite pequenas transformações nos dados em trânsito (ex: cast, drop de campos, mascaramento).
- **Converter**: Define o formato de serialização (JSON, Avro, Protobuf).
- **Configurações Específicas**: Parâmetros como conexão, autenticação, tabelas, etc.

Exemplo de configuração simplificada (JSON):

```json
{
    "name": "meu-jdbc-connector",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url": "jdbc:postgresql://localhost:5432/meubanco",
        "connection.user": "usuario",
        "connection.password": "senha",
        "table.whitelist": "minha_tabela",
        "mode": "incrementing",
        "incrementing.column.name": "id",
        "topic.prefix": "meu_topico_",
        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "value.converter": "io.confluent.connect.avro.AvroConverter",
        "value.converter.schema.registry.url": "http://localhost:8081"
    }
}
```

---

## Boas Práticas com Conectores

- **Versionamento**: Sempre utilize versões atualizadas dos conectores e valide compatibilidade com as versões dos sistemas de origem/destino.
- **Ambiente de Testes**: Teste conectores em ambientes isolados antes de promover para produção.
- **Gerenciamento de Dependências**: Certifique-se de que todos os JARs necessários estejam presentes na pasta de plugins do Connect.
- **Monitoramento**: Utilize comandos e APIs para monitorar status, tasks e logs dos conectores.
- **Documentação**: Consulte sempre a documentação oficial do conector para entender limitações e configurações suportadas.

---

## Exemplos de Conectores Populares

### JDBC Source Connector

Permite ingestão de dados de bancos relacionais (Postgres, MySQL, SQL Server, Oracle, etc.) via JDBC.

- **Modos de Operação**:
    - `incrementing`: Usa uma coluna incremental (ex: ID auto-incremento).
    - `timestamp+incrementing`: Usa timestamp e coluna incremental para detectar novos registros.
    - `bulk`: Lê toda a tabela a cada execução.

**Atenção**: Buracos em colunas incrementais podem causar perda de dados. Recomenda-se criar views ou colunas artificiais sequenciais se necessário.

### MongoDB Source Connector

Conector oficial do MongoDB, utiliza Change Streams para capturar alterações em coleções.

- Permite publicar documentos completos ou parciais.
- Suporta configurações para CDC (Change Data Capture).

### Debezium CDC Connector

Especializado em CDC para bancos como MySQL, Postgres, SQL Server, Oracle, etc.

- Requer habilitação de logs de transação (binlog, replication, CDC).
- Permite capturar operações de insert, update e delete.
- Suporta configuração de tombstone para deletados.

---

## Transformações Simples (SMT)

As SMTs permitem pequenas transformações nos dados, como:

- **Criar chave a partir de campo**:
- **Mascarar campos sensíveis**:
- **Conversão de tipos**:

Exemplo de configuração SMT para criar chave a partir do campo `id`:

```json
"transforms": "CreateKey",
"transforms.CreateKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
"transforms.CreateKey.fields": "id"
```

---

## Consumo e Processamento dos Dados com PySpark

Após a ingestão dos dados no Kafka, é comum utilizar frameworks como **PySpark** para processar esses dados em tempo real ou batch.

### Exemplo: Consumindo Dados do Kafka com PySpark

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StringType, IntegerType

# Cria sessão Spark
spark = SparkSession.builder \
        .appName("KafkaConnectIngestion") \
        .getOrCreate()

# Define o schema dos dados (ajuste conforme seu caso)
schema = StructType() \
        .add("id", IntegerType()) \
        .add("nome", StringType()) \
        .add("email", StringType())

# Lê dados do Kafka
df = spark.readStream \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "localhost:9092") \
        .option("subscribe", "meu_topico_minhatabela") \
        .load()

# Converte o valor de binário para string e aplica o schema
json_df = df.selectExpr("CAST(value AS STRING) as json") \
        .select(from_json(col("json"), schema).alias("data")) \
        .select("data.*")

# Exemplo de transformação: filtrar registros
filtered_df = json_df.filter(col("email").endswith("@empresa.com"))

# Escreve o resultado em console (ou outro sink)
query = filtered_df.writeStream \
        .outputMode("append") \
        .format("console") \
        .start()

query.awaitTermination()
```

---

## Dicas Finais

- **SMT**: Use com moderação, apenas para transformações simples.
- **Schema Registry**: Utilize para garantir compatibilidade de esquemas (especialmente com Avro/Protobuf).
- **CDC**: Entenda as limitações do conector e do banco de origem.
- **Monitoramento**: Sempre monitore tasks, offsets e status dos conectores.
- **Testes**: Teste exaustivamente antes de subir para produção.

---

## Referências

- [Documentação Oficial do Kafka Connect](https://kafka.apache.org/documentation/#connect)
- [Confluent Hub](https://www.confluent.io/hub/)
- [Stream Reactor](https://github.com/lensesio/stream-reactor)
- [Debezium](https://debezium.io/documentation/)

---

## Conclusão

O Kafka Connect é uma poderosa ferramenta para ingestão e integração de dados no ecossistema Kafka, facilitando a construção de pipelines de dados robustos e escaláveis. Combinado com ferramentas de processamento como PySpark, permite criar soluções de streaming analytics e integração de dados em tempo real de forma eficiente e declarativa.

