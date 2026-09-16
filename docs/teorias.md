# Imersão - Engenharia de Dados

## DataBricks

O Databricks é uma plataforma unificada de dados e inteligência artificial, construída em torno do Apache Spark (que os próprios fundadores criaram).

Na prática, ela reúne em um só lugar:

Engenharia de dados: pipelines de ETL/ELT para processar grandes volumes de dados.
Data lakehouse: combina a flexibilidade de um data lake com a estrutura e confiabilidade de um data warehouse (usando o formato Delta Lake).
Ciência de dados e machine learning: notebooks colaborativos (parecidos com Jupyter), treinamento e deploy de modelos de ML/IA.
Analytics e BI: consultas SQL e dashboards sobre os mesmos dados, sem precisar duplicar tudo em outra ferramenta.
Governança: o Unity Catalog centraliza permissões e catalogação de dados e modelos. 

## Arquitetura Medalhão

A arquitetura medalhão (medallion architecture) é um padrão de organização de dados em camadas dentro de um data lakehouse (muito usado no Databricks). Ela divide o fluxo de dados em três níveis, cada um representado por uma "medalha":

🥉 Bronze (raw)
Dados brutos, exatamente como chegam da fonte — sem tratamento, sem validação. Serve como histórico fiel e ponto de partida, permitindo reprocessar tudo se algo der errado nas etapas seguintes.

🥈 Silver (limpo/validado)
Dados limpos, filtrados e padronizados: remoção de duplicatas, correção de tipos, junções entre tabelas, tratamento de valores nulos. Já é um dado mais confiável, usado por times de engenharia e análise para explorações mais profundas.

🥇 Gold (agregado/negócio)
Dados agregados e modelados para consumo direto por dashboards, relatórios de BI e modelos de machine learning. Já está no formato que responde perguntas de negócio (ex: vendas por região e mês).

Por que usar esse padrão:

Cada camada melhora a qualidade da anterior progressivamente.
Facilita debugging (dá pra rastrear onde um dado "quebrou").
Reaproveita as camadas intermediárias para múltiplos casos de uso, sem reprocessar tudo do zero.
É um padrão popular especialmente no Delta Lake / Databricks, mas o conceito se aplica a qualquer arquitetura de dados em camadas.

## Spark
O Apache Spark é um motor de processamento de dados distribuído, feito para trabalhar com grandes volumes de dados de forma rápida e paralela — dividindo o trabalho entre vários computadores (nós) de um cluster.

**Principais características:**

- **Processamento em memória:** ao contrário de ferramentas mais antigas (como o Hadoop MapReduce), o Spark processa dados na memória RAM sempre que possível, o que o torna muito mais rápido
- **Distribuído:** os dados e o processamento são divididos entre vários nós de um cluster, permitindo escalar horizontalmente para volumes enormes de dados.
- **Multi-linguagem:** pode ser usado com Python (PySpark), Scala, Java, R e SQL.
- **Versátil:** não serve só para processamento batch — também lida com streaming (dados em tempo real), machine learning (MLlib), consultas SQL (Spark SQL) e processamento de grafos (GraphX).

**Componentes principais:**

- Spark Core: motor base, gerencia tarefas e memória.
- Spark SQL: permite rodar consultas SQL sobre dados estruturados.
- Spark Streaming / Structured Streaming: processa dados em tempo real.
- MLlib: biblioteca de machine learning.
- GraphX: processamento de grafos.

Conceito-chave: os dados são organizados em DataFrames (estruturas tabulares, parecidas com uma tabela SQL ou um DataFrame do pandas) ou RDDs (a estrutura mais antiga e de baixo nível), e o Spark otimiza automaticamente como distribuir e executar as operações sobre eles.

É a base tecnológica sobre a qual o Databricks foi construído — na verdade, o Databricks foi fundado pelos criadores do próprio Spark.