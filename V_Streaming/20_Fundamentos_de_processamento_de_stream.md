
# Capítulo 20 - Fundamentos de processamento de stream



## Introdução

<div style="text-align: justify; padding-top: 10px;"> 
Processamento de stream é requisito-chave para muitas aplicações Bigdata. Assim que uma aplicação computa algo de valor - digamos, um relatório sobre atividade de cliente ou um novo modelo de machine learning - uma organização irá querer computar este resultado continuamente em produção. Como resultado, organizações de todos os tamanhos estão começando a incorporar processamento de stream, muitas vezes até na primeira versão do aplicativo.    
<br><br>
Por sorte, Apache Spark tem uma longa história de suporte de alto nível para streaming. Em 2012, o projeto incorporou a API Spark Streaming e DStreams, uma das primeiras APIs que disponibilizaram processamento de stream usando operadores funcionais de alto nível como <i>map</i> e <i>reduce</i>. Centenas de organizações agora usam DStreams em produção para aplicações de tempo-real, muitas vezes processando terabytes de dados por hora. A API DStream é semelhante à API RDD, no entanto, a API DStream é baseada em operações de nível relativamente baixo em objetos Java/Python que limitam otimizações de alto-nível. Em 2016, o projeto Spark adicionou a API Structured Streaming, uma nova API de streaming criada diretamente em DataFrames que suporta otimizações de código com DataFrames e Datasets. A API Structured Streaming foi marcada como <i>estável</i> a partir da <b>versão 2.2 do Apache Spark</b>. 
<br><br>
O foco desse material será sobre a API Structured Streaming, que integra códigos diretamente Dataframes e Datasets, sendo que a integração com Datasets será discutida brevemente.
<br><br>        
   <b>ATENÇÃO - O suporte à API DStreaming está depreciada a partir da versão 2.3 do Apache Spark</b>
</div>

## O que é processamento de stream?


<div style="text-align: justify; padding-top: 10px;"> 
<br>
Processamento de stream é o ato continúo de incorporar novos dados para computar um resultado. Em processamento de stream a entrada de dados é ilimitada e não tem início e fim pré-determinados. É simplesmente uma forma de obter séries de dados e eventos que chegam no sistema de processamento de stream( ex: transações de cartão de crédito, cliques de um website ou leituras de sensores em dispositivos IoT). Aplicações de usuários podem então computar várias consultas sobre um stream de eventos(ex: rastrear um contador para cada tipo de evento ou agregá-los em janelas de horas). A aplicação irá gerar múltiplas versões de resultados enquanto executa, ou talvez manterá em um sistema externo de "sink" como armazenamento por chave-valor.  
<br><br>
Naturalmente, nós podemos comparar o processamento de stream com o processamento em batch, em que a computação executa em um conjunto de dados fixo. Muitas vezes isso pode ser um dataset de larga escala em um data warehouse que contém todos os eventos históricos de uma aplicação(ex: todas as visitas em um website ou leituras de sensores do último mês). Processamento batch também pode envolver consultas para computar, similarmente ao sistema de streaming, mas somente computa o resultado uma única vez.
<br><br>
Embora processamento em batch e streaming soem diferentes, na prática, eles geralmente precisam trabalhar juntos. Por exemplo, aplicações de streaming normalmente precisam juntar dados em um dataset escrevendo periodicamente por um processamento batch, e a saída do job de streaming são normalmente arquivos ou tabelas que são consultadas em jobs em batch. Mais, qualquer lógica de negócio em suas aplicações precisam trabalhar consistentemente através de execuções em streaming e batch, por exemplo: se você tem um código que processa faturas de um usuário, seria prejudicial recuperar diferentes resultados durante a execução em streaming versus o "modo batch". Para lidar com essas necessidades, Structured Streaming foi projetada partindo do princípio da fácil interoperabilidade com o resto da API Spark, incluindo operações de aplicações batch. De fato, desenvolvedores do Structured Streaming cunharam o termo <i>aplicações contínuas</i> para capturar aplicações <i>end-to-end</i> que consistem em streaming, batch e jobs interativos. Todos trabalhando <b>nos mesmos dados</b> para entregar o produto final. Structured Streaming é focado em tornar simples construir aplicações em modo <i>end-to-end</i> ao invés de lidar com registros no nível do stream.
</div>

## Casos de uso de processamento de stream

<div style="text-align: justify;">
<br>
Nós definimos processamento de streaming como processamento incremental de datasets ilimitados, mas é uma maneira estranha de motivar casos de uso. Antes nós precisamos entender as vantagens e as desvantagens do streaming, e porque se deve utilizá-lo. Abaixo alguns casos de uso
</div>

### Notificações e alertas
<br>
<div style="text-align: justify;">
Provavelmente o caso de uso de streaming mais óbvio envolve notificações e alertas. Dado uma série de eventos, uma notificação ou alerta deve ser engatilhado se algum tipo de evento ou série ocorrer. Isto não necessariamente implica em tomadas de decisões preprogramadas ou autônomas; alertas podem também serem usados para notificar um humano, em contrapartida, para que este tome alguma ação necessária. Exemplo: um alerta pode ser dirigido a um funcionário de um centro de distribuição para que ele pegue algum item em um armazém e envie a um cliente. Em ambos os casos, a notificação precisa ser rápida. 
</div>


### ETL incremental
<br>
<div style="text-align: justify;">
Uma das aplicações mais comuns de streaming é reduzir a latência da obtenção de dados de um data warehouse - em suma, "um batch job, só que streaming". Jobs do tipo batch são normalmente usados para cargas por ETL que transformam o dado "cru" em um formato estruturado como parquet permitindo consultas eficientes. Usando batch jobs com o Structured Streaming esses jobs batchs são incorporados no streaming o que aumenta significativamente a velocidade de carregamento dos dados. 
<br><br>
Outros exemplos como atualização de dados em tempo-real, tomada de decisões em tempo-real e machine learning online podem ser exploradas em maiores detalhes no material.
</div>

## Vantagens do processamento em stream
<br>
<div style="text-align: justify;">
Resumidamente, a principal vantagem de processamento em stream sobre o processamento em batch é a baixa latência, o que permite produzir aplicações de "quase tempo-real" ou em tempo-real, ou seja, muito rápido! Particularmente em sistemas de tomada de decisão o processamento em stream se tornou algo crítico. Em sistemas que exigem atualizações constantes de informação, mas não em tempo-real também podem se utilizar das vantagens do streaming, já que o fluxo de dados é constante.
</div>

## Desafios do processamento de stream

    
   <p style="text-align: justify">
    Quando há dependência de informações, o processamento em stream torna-se um problema. Exemplo: Imagine a seguinte série de dados vinda num sistema de stream abaixo:
    </p>


```python
# json (não executável)
{value: 1, time: "2017-04-07T00:00:00"}
{value: 2, time: "2017-04-07T01:00:00"}
{value: 5, time: "2017-04-07T02:00:00"}
{value: 10, time: "2017-04-07T01:00:00"}
{value: 7, time: "2017-04-07T03:00:00'}
```

<div style="text-align: justify">
Se precisarmos analisar isso sequencialmente considerando a data, haveria um problema para ordenar os dados aqui. E considere que a série pode não estar completa, ou seja, ou a série está incompleta, ou perdeu-se dados. Como detectar isso? Como proceder nesses casos? Esse é um dos vários desafios no processamento de streaming. Para sumarizar os desafios, segue a lista a seguir:
<br>
    
* Processamento baseado em timestamp, mas fora de ordem (tambem conhecido como "evento de tempo");

* Manutenção de grandes quantidades de estados;

* Suporte à altíssimas taxas de transferência de dados;

* Processar cada evento **exatamente uma vez** independente de falhas do servidor;

* Lidar com desbalanceamento de carga e retarda-atalhos;

* Resposta à eventos de baixa-latência;

* Joining com dados externos em outros sistemas de armazenamento (bancos de dados relacionais, por exemplo);

* Determinar como novos eventos que chegam irão atualizar as sinks;

* Escrever dados transacionalmente;

* Atualizar sua lógica de negócio na aplicação em tempo de execução;



Material baseado em exemplos do livro __Spark - The Definitive Guide. Bill Chambers e Matei Zaharia__
