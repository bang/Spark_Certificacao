
# Capítulo 21. Básico de Structured Streaming


## Intro
<br>
<div style="text-align: justify">
Strucutured Streaming é a API mais recente do projeto Apache Spark para processamento em stream. Sua principal característica é a capacidade de integração com Dataframes e Datasets o que torna tudo muito familiar para quem já trabalha com Spark fora do contexto de streaming.
<br><br>
Outra característica dessa API é que o processamento dos dados de streaming ocorre de forma semelhante ao processamento em batch, embora os dados sejam disponibilizados a partir de um <i>stream</i>, de fato. Isso signficia que além das features de stream específicas estão disponíveis também as mesmas features de Dataframes e Datasets.
<br><br>
Uma maneira de ver enxergar a idéia do processamento via Structured Stream é tratar o stream de dados como uma tabela(de forma análoga a como se trata dados em um Dataframe ou Dataset). Porém, mantenha em mente que essa tabela é constantemete atualizada. Cada consulta pode ser especificada em batch ou streaming. Internamente a API Structured Stream irá "entender" como incrementar os dados para sua consulta, de forma que seja tolerante à falhas.
</div>


```python
import sys,os,time
sys.path.append("/usr/local/lib/python2.7/dist-packages")
import pyspark # para kernels Python que chamam o pyspark
from pyspark.sql import functions as F
from pyspark.sql import SparkSession,Row,Column
from pyspark.sql.types import *
spark = SparkSession.builder.appName("SparkStreamingExamples").getOrCreate()
```

![image.png](attachment:image.png)


<div style="text-align: justify">
Em termos simples, seu Dataframe, por exemplo, continua sendo um Dataframe, só que atualizado continuamente via stream. Claro que há limitações sobre consultas que você pode fazer, devido a própria natureza do streaming. Mas o que se quer que se mantenha em mente é que você pode manipular dados de streaming usando Dataframes usando a API Strucutred Streaming.

## Fontes de entrada ou *source*

<div style="text-align: justify; padding-top: 10px;">
    
<p>
    Entenda uma <i>source</i> como "drivers" de <b>leitura</b> de dados, ou <i>reader</i>. Alguns que são disponibilizados hoje:
</p>

</div>


* Apache Kafka 0.10

* Sistemas de arquivo distribuídos como HDFS ou S3

* Sockets

<div style="text-align: justify; padding-top: 10px;">

<p>
    Uma <i>source</i> é o ponto de contato com a fonte de dados é exatamente a "ponta de entrada". Você pode fazer transformações antes de enviar para uma <i>sink</i> (que veremos a seguir). 
</p>

<p>
    Para implementar uma <i>source</i>, utiliza-se uma propriedade da <b>sessão do spark</b> chamada <i>readStream</i>.
</p>
</div>


```python
# Exemplo (Não executável):
spark.readStream\
    .schema(someStaticDataFrameSchema)\
    .options("maxFilesPerTrigger",1)\
    .json('/data/jsonDir/')
```

## Saída de dados ou *sink*


### O que é Sink ?

<div style= "padding-top:10px;">
    
<p style="text-align: justify;">Em tradução livre, "pia". Em termos técnicos(e úteis), sinks são como "drivers" para <b>escrita</b> de dados, ou <i>writers</i>. Alguns que estão disponíveis:
</p>
</div>

   * *kafka*, ou seja, o driver para o Spark Apache Kafka 0.10
    
   * praticamente **todos** os formatos de arquivos e sockets
    
   * *foreach* para executar computação arbitrária em registros de saída
    
   * *console* para testes
    
   * *memory* para debugging


<div style="text-align: justify">

<p style="text-align: justify;">Pode-se pensar numa <i>sink</i> como um repositório de dados do stream. Uma vez estabelecido, o sistema de streaming passa a ler dados do <i>source</i> e escrever dados na <i>sink</i>. Desse modo, estabelece-se o fluxo de dados, e a partir da sink obtem-se os dados conforme o tipo de consulta que se quer da <i>source</i>. Dessa forma, pode-se criar várias <i>sinks</i> diferentes com filtros e transformações diferentes da mesma <i>source</i>, com a vantagem de não ser necessário ter absolutamente todos os dados disponíveis de uma vez, o que exigiria uma quantidade de memória e/ou espaço em disco normalmente não-disponível dentro do contexto de Bigdata. Isso é eficiência!
</p>
<p style="text-align: justify;">Para utilizar uma <i>sink</i>, deve-se implementar uma propriedade do dataframe chamada <i>writeStream</i>, como demostrado mais abaixo.
</p>
</div>


```python
# Exemplo (Não executável):
spark.writeStream\
    .format('kafka')\
    .options(...)\
    .start()
```

<div style="text-align: justify">
Repare que o método <i>start()</i> deve ser implementado, afim de iniciar o fluxo de entrada e saída de dados. Novas modificações nesse fluxo devem ser implementadas como um novo fluxo de entrada e saída no stream de dados.
</div>

## Modos de saída
<br>
<p style="text-align: justify;">Assim como se define como os dados vão entrar, também devemos definir como os dados vão sair. No caso de streaming isso vai um pouco além. Definimos também <b>quais</b> dados vão sair. As opções são:
</p>

* *append* ( somente novos registros vão para a sink )

* *update* ( atualiza os dados modificados )

* *complete* ( reescreva TUDO o que está na sink )

<p style="text-align: justify;">
Note que alguns tipos de consulta estão disponíveis em alguns tipos de sinks, como consultas do tipo <b>map</b>, por exemplo. 

</p>

## Processamento em "tempo de evento"
<br>
<p>Structured Streaming também suporta processamento de dados em <i>tempo de evento</i> (ex: dados processados com base no timestamp que podem chegar fora de ordem). Existem duas idéias-chave que vocẽ irá precisar comprender aqui neste momento:   
</p>

* *dados em tempo de evento*

<p>Significa campos de tempo que são "embuídos" no seu dado. Isto significa processar dados no tempo em que o dado foi gerado, ao invés de processar dados conforme eles vão chegando no seu sistema, ou seja, independentemente da ordem de chegada. Basicamente, uma coluna timestamp (ou outro formato de dado que expresse o momento de geração do dado), junto com o dado;</p>

<div style="width:100%; padding-top: 20px;">

* *marca d'água (Watermarks)*
</div>
<p>A expressão <i>Watermarks</i> (marca d'água) expressa intervalos de tempo ou "janelas" que limitam a informação que chega do streaming;
</p>

## Menos "conversa" e mais código!


#### Carregando dados no modo estático

<p>
Apenas para visualizar a diferença de como se carrega dados no modo <i>estático</i> e posteriormente, no modo <i>streaming</i>

</p>


```python
# Exemplo de leitura de dados em um ou mais arquivos em um diretório no HDFS:
path = "file:///root/Spark_Certificacao/data/activity-data/"
df_static = spark.read.json(path)
dataSchema = df_static.schema
```


```python
# O esquema:
df_static.printSchema()
```

    root
     |-- Arrival_Time: long (nullable = true)
     |-- Creation_Time: long (nullable = true)
     |-- Device: string (nullable = true)
     |-- Index: long (nullable = true)
     |-- Model: string (nullable = true)
     |-- User: string (nullable = true)
     |-- gt: string (nullable = true)
     |-- x: double (nullable = true)
     |-- y: double (nullable = true)
     |-- z: double (nullable = true)
    



```python
# Uma amostra do DataFrame:
df_static.show(3)
```

    +-------------+-------------------+--------+-----+------+----+-----+------------+------------+------------+
    | Arrival_Time|      Creation_Time|  Device|Index| Model|User|   gt|           x|           y|           z|
    +-------------+-------------------+--------+-----+------+----+-----+------------+------------+------------+
    |1424686735090|1424686733090638193|nexus4_1|   18|nexus4|   g|stand| 3.356934E-4|-5.645752E-4|-0.018814087|
    |1424686735292|1424688581345918092|nexus4_2|   66|nexus4|   g|stand|-0.005722046| 0.029083252| 0.005569458|
    |1424686735500|1424686733498505625|nexus4_1|   99|nexus4|   g|stand|   0.0078125|-0.017654419| 0.010025024|
    +-------------+-------------------+--------+-----+------+----+-----+------------+------------+------------+
    only showing top 3 rows
    


### Carregando dados no "modo streaming"

<div style="text-align: justify;">

<p>Abaixo, o código configura o streaming, partindo do princípio que Spark executa comandos no modo <i>lazy</i>, ou seja, podemos configurar e até aplicar algumas transformações antes de executar de fato. No caso de streaming, tudo começa a executar depois a partir da invocação do método <b><i>start()</i></b>, como será demonstrado mais adiante.
</p>
    
<p>Para a API Structured Streaming, a idéia é manter fluxos de dados, ou seja, entradas e saídas, que correspondam respectivamente, a configurar um <i>reader</i> e um <i>writer</i> para o streaming de dados. No código abaixo, um exemplo de um <i>reader</i> típico. Repare que está disponível a partir da própria sessão do Spark (pyspark.sql.SparkSession). A única diferença é que ao invés de obter a leitura do streaming de dados através da propriedade "read" (como no exemplo utilizado na seção "Carregando dados no modo estático"), usamos <i>readStream</i>.      
</p>
</div>


```python
# Configurando a leitura do streaming de dados.
df_streaming = spark.readStream \
    .schema(dataSchema) \
    .option("maxFilesPerTrigger",1) \
    .json(path)
```


```python
# Esse é o dado que efetivamente se quer ver, expresso na transformação abaixo. Mas isso não será executado agora! 
df_activityCounts = df_streaming.groupBy("gt").count()
```


```python
# Como se espera a execução disso em uma máquinia pequena ou mesmo numa VM, é razoável 
# configurar o número de partições em um valor pequeno.
spark.conf.set("spark.sql.shuffle.partitions", 5)
```

Atendendo aos requisitos da API Structured Streaming, se existe um *readStream*, deve existir um *writeStream*. Em outras palavras, deve se definir como e o que será despejado na *sink*. Exemplo:


```python
# Configurando a escrita do resultado da transformação. 
# O nome 'activity_counts' estará disponível como uma entidade através do spark.sql como demonstrado mais adiante.
# o formato da 'sink' é 'memory', ou seja, osda sink serão despejados diretamente na memória
# o modo de escrita na sink é 'complete', ou seja, espera-se que TODOS os dados disponíveis do streaming 
# estejam na sink.

activityQuery = df_activityCounts.writeStream \
    .queryName("activity_counts") \
    .format("memory") \
    .outputMode("complete") \
    .start()

# O método 'start()' executa a consulta de forma assíncrona por padrão. Então, para evitar que outro processamento que seja 
# necessário sobrescreva algum resultado ou execute em um momento que não deveria, o método *awaitTermination()* é requerido. 
# Em outras palavras, é preciso "sinalizar" para o Spark, que se quer esperar até que o processamento do streaming 
# termine antes de executar algum código após a invocação do método *start()*.
```


```python
activityQuery.awaitTermination()
```

O método *awaitTermination()* evita o processo de sair enquanto a query está ativa, mas não funciona no Jupyter. Por favor, leia a documentação indicada abaixo:

* https://stackoverflow.com/questions/55840424/spark-streaming-awaittermination-in-jupyter-notebook

* https://docs.databricks.com/spark/latest/structured-streaming/production.html

* http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html

* https://docs.databricks.com/spark/latest/structured-streaming/demo-notebooks.html

<div style="text-align: justify">
E assim a configuração do fluxo de dados é definida, e o início é executado através do método <i>start()</i>. Repare que o <i>writer</i>(writeStream), possui uma propriedade chamada <i>queryName</i> Essa propriedade é definida com o objetivo de identificar onde <i>spark.sql</i> irá recuperar dados no caso de uma consulta, de forma análoga como se identifica tabelas e views, como demonstrado mais adiante.
</div>


```python
# Exibe os streamings ativos na sessão do Spark.
print str(spark.streams.active)
```

    [<pyspark.sql.streaming.StreamingQuery object at 0x7f8c480d0e50>]



```python
# Considere 
from time import sleep
for x in range(5):
    spark.sql("SELECT * FROM activity_counts").show(3)
    sleep(1)
```

    +----------+------+
    |        gt| count|
    +----------+------+
    |       sit|984714|
    |     stand|910783|
    |stairsdown|749059|
    +----------+------+
    only showing top 3 rows
    
    +----------+------+
    |        gt| count|
    +----------+------+
    |       sit|984714|
    |     stand|910783|
    |stairsdown|749059|
    +----------+------+
    only showing top 3 rows
    
    +----------+------+
    |        gt| count|
    +----------+------+
    |       sit|984714|
    |     stand|910783|
    |stairsdown|749059|
    +----------+------+
    only showing top 3 rows
    
    +----------+------+
    |        gt| count|
    +----------+------+
    |       sit|984714|
    |     stand|910783|
    |stairsdown|749059|
    +----------+------+
    only showing top 3 rows
    
    +----------+------+
    |        gt| count|
    +----------+------+
    |       sit|984714|
    |     stand|910783|
    |stairsdown|749059|
    +----------+------+
    only showing top 3 rows
    


## Transformações em Fluxos
<br>
<div style="text-align: justify">As transformações de streaming, como mencionamos, incluem quase todas as transformações estáticas do DataFrame que você já viu na Parte II. Todas as transformações de seleção, filtro e simples são suportadas, assim como todas as funções do DataFrame e manipulações de coluna individuais. As limitações surgem em transformações que não fazem sentido no contexto de dados de fluxo contínuo. Por exemplo, no Apache Spark 2.2, os usuários não podem classificar fluxos que não são agregados e não podem executar vários níveis de agregação sem usar o <i>Stateful Processing</i>. Essas limitações podem ser removidas à medida que o <b>Structured Streaming</b> (Fluxo contínuo estruturado) continua a ser desenvolvido. Recomendamos que você verifique a documentação da sua versão do Spark para atualizações.</div>

### Seleção e Filtros
<br>
<div style="text-align: justify">
Como dito anteriormente nos capítulos 20 e 21, a API Structured Streaming disponibiliza os dados em objetos DataFrame "incrementáveis". Isso significa, basicamente, os métodos de seleção e filtros (.select(), .filter(), .where(), etc.), estarão disponíveis. Basta apenas que novas queries sejam disponibilizadas em <b>um novo <i>writer</i></b> (propriedade writeStream), ou seja, toda vez que você precisar de dados com uma nova seleção e filtragem, você deve disponibilizar isso em um novo <i>writer</i>.<br> 
</div>


```python
# Exemplo:  
from pyspark.sql.functions import expr
simpleTransform = df_streaming.withColumn('stairs', expr("gt like '%stairs%'"))\
    .where("stairs")\
    .where("gt is not null")\
    .select("gt", "model", "arrival_time", "creation_time")\
    .writeStream\
    .queryName("simple_transform")\
    .format("memory")\
    .outputMode("append")\
    .start()
```

<div style="text-align: justify">
Aqui, a referência para o novo conjunto de dados é <i>simple_transform</i>. Esse novo dataset foi gerado a partir de um <b>novo writer</b>, de forma análoga ao que foi referenciado pelo nome <i>activity_counts</i>, e melhor! Ambos como objetos <i>DataFrame</i>, o que permite praticamente tudo o que está disponível via módulo <i>spark.sql<i/> como agregações, joins, etc..
</div>


```python
spark.sql("SELECT * from simple_transform").show(2)
```

    +--------+------+-------------+-------------------+
    |      gt| model| arrival_time|      creation_time|
    +--------+------+-------------+-------------------+
    |stairsup|nexus4|1424687983734|1424687981741908919|
    |stairsup|nexus4|1424687984030|1424687982033901107|
    +--------+------+-------------+-------------------+
    only showing top 2 rows
    



```python
activityQuery.stop()
```

### Agregações

<div style="text-align: justify;padding-top: 10px;">
Structured Streaming suporta agregações! Já vimos isso quando usamos 'count', que é um método de agregação, certo? Agora, veremos as agregações de outra maneira mais 'exótica' montando um 'cubo' com as médias das acelerações dos sensores.
</div>


```python
deviceModelStats = df_streaming.cube("gt","model")\
                    .avg()\
                    .drop("avg(Arrival_time)")\
                    .drop("avg(Creation_Time)")\
                    .drop("avg(Index)")\
                    .writeStream.queryName("device_counts")\
                    .format("memory")\
                    .outputMode("complete")\
                    .start()
```


```python
spark.sql("SELECT * FROM device_counts").show(5)
```

    +-----+------+--------------------+--------------------+--------------------+
    |   gt| model|              avg(x)|              avg(y)|              avg(z)|
    +-----+------+--------------------+--------------------+--------------------+
    |  sit|  null|-5.20306357426065...|4.277122408352292...|-1.59707753258045...|
    |stand|  null|-3.66649186763284...|5.027077296179175E-4|3.636363101712778...|
    |  sit|nexus4|-5.20306357426065...|4.277122408352292...|-1.59707753258045...|
    |stand|nexus4|-3.66649186763284...|5.027077296179175E-4|3.636363101712778...|
    | null|  null|-0.00789107648757...|-0.00192346751228...|-6.73493799349196...|
    +-----+------+--------------------+--------------------+--------------------+
    only showing top 5 rows
    



```python
# Checando o writer
spark.sql("SELECT * from device_counts where model IS NOT NULL").show(2)
```

    +-----+------+--------------------+--------------------+--------------------+
    |   gt| model|              avg(x)|              avg(y)|              avg(z)|
    +-----+------+--------------------+--------------------+--------------------+
    |  sit|nexus4|-5.28592641392478E-4|4.600768453448703...|-4.11482783467380...|
    |stand|nexus4|-4.41379069038206...|5.811279945410626E-4|2.254789355204209...|
    +-----+------+--------------------+--------------------+--------------------+
    only showing top 2 rows
    


### Joins
<div style="text-align: justify;padding-top: 10px">
A partir da versão 2.3 do Spark, é possível implementar <i>joins</i> com múltiplos streams e também incluir objetos DataFrame estáticos no mesmo join!
</div>


```python
# Exemplo de join de um DataFrame estático(static) com um streaming
df_historicalAgg = df_static.groupBy("gt", "model").avg()

deviceModelStats = df_streaming.drop("Arrival_Time", "Creation_Time", "Index")\
    .cube("gt", "model").avg()\
    .join(df_historicalAgg, ["gt", "model"])\
    .writeStream.queryName("device_counts2").format("memory")\
    .outputMode("complete")\
    .start()
```


```python
df_historicalAgg.show(2)
```

    +----+------+--------------------+--------------------+-----------------+--------------------+--------------------+--------------------+
    |  gt| model|   avg(Arrival_Time)|  avg(Creation_Time)|       avg(Index)|              avg(x)|              avg(y)|              avg(z)|
    +----+------+--------------------+--------------------+-----------------+--------------------+--------------------+--------------------+
    |bike|nexus4|1.424751134339985...|1.424752127369589...|326459.6867328154|0.022688759550866855|-0.00877912156368...|-0.08251001663412343|
    |null|nexus4|1.424749002876339...|1.424749919482127...|219276.9663669269|-0.00847688860109...|-7.30455258739187...|0.003090601491419931|
    +----+------+--------------------+--------------------+-----------------+--------------------+--------------------+--------------------+
    only showing top 2 rows
    


### Lendo de sources e escrevendo em sinks
<br>
<div style="text-align: justify; padding-top:10px;">
Nessa seção, vamos falar um pouco mais especificamente sobre alguns tipos de fluxo de entrada e saída utilizando <i>source</i> e <i>sink</i>.
</div>

#### Sink Kafka

<div style="text-align: justify; padding-top:10px;">
Como já sabemos, para lidar com streams, temos entradas de dados (<i>source</i>), que são implementadas através da propriedade <i>readStream</i>. No caso do Kafka isso também é verdade, porém, existem especificidades a se considerar. Cada registro do Kafka traz, minimamente: chave, valor e o momento (timestamp) em que o registro foi gravado no stream de dados. Existem também os <i>tópicos</i>, que são divisões imutáveis de segmentos de dados identificáveis pelo Kafka através de strings.
</div>    


### Lendo de uma fonte Kafka
<br>
<div style="text-align: justify">    
A <b>leitura</b> de dados em um <i>tópico</i> do Kafka é chamada de <i>subscribing</i> (inscrição em tradução livre para o português).
 
Na leitura, é possível ser mais preciso quanto aos dados que se quer trazer no streaming, incluindo dados de vários tópicos de uma vez. Para isso utiliza-se *assign*, passando os intervalos de dados e os tópicos.

<br>
Ex: 

{"topicA":[0,1], "topicB":[2,4]};

Ou, pode-se obter dados através das opções *subscribe* e *subscribePattern*.


* Exemplo de *subscribe*: "produtos.eletro"  

* Exemplo de *subscribePattern*: "produtos."  


Outro detalhe a se considerar, é a origem dos dados. Para o caso do Kafka, utiliza-se a opção "kafka.bootstrap.servers" para determinar uma ou várias origems de dados (no caso de várias, separa-se por ',').

A opção <i>failOnDataLoss</i> define se o streaming será abortado ou não em caso de falha(True ou False). Por padrão a opção é True.

Outras opções na documentação de [Integração do Spark 2.3.2 com o Kafka](https://spark.apache.org/docs/2.3.2/structured-streaming-kafka-integration.html)
    
Se nehuma transformação de dados é necessária, basta invocar o método *load()*

Cada linha de uma *source* tem o seguinte esquema(schema):

* key: binary
* value: binary
* topic: string
* partition: int
* offset: long
* timestamp: long
    


```python
# Exemplos de leitura de dados de uma *source* Kafka (Não executáveis)

# Inscrição(subscribing) em 1 tópico
df1 = spark.readStream.format("kafka")\
  .option("kafka.bootstrap.servers", "host1:port1, host2:port2")\
  .option("subscribe", "topic1")\
  .load()

# Inscrição(subscribing) em múltiplos tópicos
df2 = spark.readStream.format("kafka")\
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")\
  .option("subscribe", "topic1, topic2")\
  .load()

# Inscrição(subscribing) usando padrões
df3 = spark.readStream.format("kafka")\
  .option("kafka.bootstrap.servers", "host1:port1, host2:port2")\
  .option("subscribePattern", "topic.*")\
  .load()
```

<div style="text-align: justify;padding-top: 10px">
A <b>escrita</b> dos dados em um <i>tópico</i> é chamada de <i>publishing</i> (publicação em tradução livre para o português).

As opções no caso de leitura seriam:    

* *kafka.bootstrap.servers*: define os endereços dos nós do cluster que podem ser usados para gravar os dados;
<br><br>
* *checkpointLocation* não é uma opção exclusiva do Kafka (faz parte da API Strucutred Streaming), porém no caso do Kafka pode ser usada como implementação de tolerância à falhas, podendo salvar informações no HDFS/S3 etc. e usar esses dados como referência para recuperar dados do streaming posteriormente. 
</div>


```python
#### Exemplos de escrita em uma *sink* Kafka (Não executáveis)

df1.selectExpr("topic", "CAST(key AS STRING)", "CAST(value AS STRING)")\
  .writeStream\
  .format("kafka")\
  .option("kafka.bootstrap.servers", "host1:port1, host2:port2")\
  .option("checkpointLocation", "/to/HDFS-compatible/dir")\
  .start()
 
df1.selectExpr("CAST(key AS STRING)", "CAST(value AS STRING)")\
  .writeStream\
  .format("kafka")\
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")\
  .option("checkpointLocation", "/to/HDFS-compatible/dir")\
  .option("topic", "topic1")\
  .start()  
```

### Sources e Sinks para teste

<div style="text-align: justify; padding-top: 10px">
Spark também fornece sinks para teste, prototipação e debugging de consultas. São eles:
</div>

#### Socket source

<div style="text-align: justify; padding-top: 10px">
    <p>
        Estabelecendo a leitura do streaming. Nesse caso, uma conexão socket local.
    </p>

</div>


```python
socketDF = spark.readStream.format("socket")\
    .option("host","localhost")\
    .option("port",9999)\
    .load()
```

<div style="text-align: justify; padding-top: 10px">
Estabelecendo o "despejo" dos dados em uma <i>sink</i> tipo <b>socket</b>
</div>


```python
socketDF.writeStream\
    .format("memory")\
    .queryName("socket_stream")\
    .start()
```




    <pyspark.sql.streaming.StreamingQuery at 0x7f8c4804fb10>



<div style="text-align: justify; padding-top: 10px">
Usando o netcat (Linux), vamos gravar alguns dados no socket, para que o flu
</div>


```python
spark.sql("SELECT * FROM socket_stream").show(5, False)   
```

    +-----+
    |value|
    +-----+
    +-----+
    


### Output Modes

<div style="text-align:justify;padding-top:10px">
Agora que sabemos <b>onde</b> os dados fluem no fluxo de streaming (<i>sources e sinks</i>), falta detalhar um pouco mais sobre <b>como</b> os dados são gravados numa <i>sink</i>. Para isso utiliza-se uma opção da propriedade <i>writeStream</i> chamada <i>outputMode</i>. Como já foi mencionado anteriormente, esses modos de saída podem ser:  
</div> 
</br>

* **Append mode:** é o modo "padrão"! Quando novas linhas são adicionadas à fonte de dados, já transformadas, filtradas etc, eventualmente elas serão despejadas numa *sink*. Esse modo garante que ela será despejada na *sink* **apenas uma vez**, assumindo que você implementou um *sink* tolerante à falhas. <br />

* **Complete mode:** Independente do estado atual do dado no stream, ele será despejado na *sink*. 

* **Update mode:** Semelhante ao modo completo, exceto que apenas as linhas modificadas do estado anterior serão despejadas na *sink*

#### Quando usar cada modo?

<div style="text-align:justify;padding-top:10px">
Basicamente a escolha do modo limita o tipo de query que você pode fazer. Por exemplo, se você precisa apenas de operações do tipo <i>map</i>, a API Structured Streaming não irá permitir o modo *complete*, porque isso iria requerer obter todos os dados desde o início do job e reescrever toda a tabela de saída. Esse requisito é proibitivamente caro em termos de recurso. Abaixo uma tabela que resume tipos de operação e algumas situações suportadas para cada modo de saída.
</div>

![image.png](attachment:image.png)

### Triggers

<div style="text-align: justify;padding-top: 10px; ">
Triggers são formas de se controlar a saída de dados para uma <i>sink</i>. Por padrão, Structured Streaming irá iniciar dados tão logo o processamento deles esteja completo. Você pode usar triggers para garantir que a sua <i>sink</i> não ficará sobrecarregada de updates ou para controlar tamanho de arquivos na saída. A seguir, falaremos de alguns tipos de trigger.
</div>

#### Triggers de processamento de tempo

<div style="text-align: justify;padding-top: 10px; ">
Basicamente, define o intervalo de tempo em que dados serão despejados na <i>sink</i> expresso em segundos.
</div>
<br>
Exemplo:


```python
df_activityCounts.writeStream.trigger(processingTime='5 seconds')\
    .format("console")\
    .outputMode("complete")\
    .start()
```

<div style="text-align: justify;padding-top: 10px; ">
Esse código faz com que os dados sejam despejados na <i>sink</i> usando no intervalo dado (5 segundos), não importando quantos dados estejam disponíveis na <i>sink</i>. Se algum dado for perdido, ou se o despejo de dados cessar, a API faz com que o sistema aguarde até o próximo despejo de dados obedecendo ao intervalo.
</div>

#### Once Trigger

<div style="text-align: justify;padding-top: 10px; ">
Você pode querer usar um trigger apenas uma vez. Para isso, utilize *once trigger*, passando o parâmetro *once=True*. Ex:
    </p>

</div>



```python
df_activityCounts.writeStream.trigger(once=True)\
    .format("console")\
    .outputMode("complete")\
    .start()
```

Material baseado em exemplos do livro __Spark - The Definitive Guide. Bill Chambers e Matei Zaharia__
