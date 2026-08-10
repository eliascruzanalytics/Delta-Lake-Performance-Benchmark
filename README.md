# 🚀 Delta Lake Performance

## Partitioning vs Z-ORDER vs Liquid Clustering

Este projeto apresenta um benchmark prático para investigar como diferentes estratégias de organização física de dados influenciam a performance de consultas no **Delta Lake / Databricks**.

O objetivo não é apenas comparar tempos de execução, mas analisar os mecanismos que influenciam a performance de workloads analíticos:

* Data Skipping
* File Pruning
* Small Files
* Shuffle
* Scan
* Organização física dos arquivos
* OPTIMIZE
* Padrão de acesso aos dados
* Cardinalidade e seletividade
* Plano de execução do Spark

## 🎯 Pergunta principal

> **Qual estratégia de organização física apresenta melhor comportamento para diferentes padrões de consulta em uma tabela Delta?**

Para responder à pergunta, o projeto compara:

```text
                 Delta Lake
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
  Partitioning    Z-ORDER    Liquid Clustering
        │            │            │
        └────────────┼────────────┘
                     ▼
                 Benchmark
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Scan          Join      Data Skipping
```

## 🧪 Experimentos

### 1. Agregação

Dataset com milhões de registros submetido a consultas de agregação.

```sql
SELECT
    categoria,
    AVG(valor)
FROM vendas
GROUP BY categoria;
```

### 2. Join

Workload envolvendo uma tabela de vendas e uma dimensão de clientes.

```sql
SELECT
    c.cliente_id,
    SUM(v.valor)
FROM vendas v
JOIN clientes c
    ON v.cliente_id = c.cliente_id
GROUP BY c.cliente_id;
```

### 3. Data Skipping

Consultas seletivas utilizando as colunas utilizadas como estratégia de organização física.

```sql
SELECT *
FROM vendas
WHERE cliente_id = 12345
  AND data BETWEEN 100 AND 120;
```

## 📊 Métricas analisadas

Os benchmarks coletam métricas de execução em vez de depender de valores previamente definidos:

* tempo de execução;
* quantidade de arquivos;
* quantidade de bytes lidos;
* quantidade de dados processados;
* shuffle;
* tempo de scan;
* número de tarefas;
* plano físico de execução;
* métricas do Delta Lake;
* comportamento antes e depois do `OPTIMIZE`.

## 🧠 Principais conceitos demonstrados

### Partitioning

Organização física baseada em uma chave fixa de particionamento.

### Z-ORDER

Organização multidimensional utilizada para melhorar o data skipping em determinados padrões de consulta.

### Liquid Clustering

Estratégia de organização física que permite definir chaves de clustering sem depender do modelo tradicional de particionamento.

### Small Files

Investigação do impacto da existência de grande quantidade de arquivos pequenos sobre operações de leitura.

### Data Skipping

Análise de como as estatísticas dos arquivos podem permitir que o mecanismo evite a leitura de dados irrelevantes.

## ⚠️ Importante

Os resultados de performance apresentados neste projeto devem ser interpretados como **resultados experimentais do ambiente utilizado**, e não como garantia de ganho percentual em ambientes produtivos.

Performance distribuída depende de diversos fatores, incluindo:

* volume de dados;
* cardinalidade;
* distribuição das chaves;
* seletividade das consultas;
* tamanho dos arquivos;
* configuração do cluster;
* cache;
* runtime;
* Photon;
* concorrência;
* padrão de workload.

Por isso, o objetivo principal deste projeto é demonstrar **metodologia de benchmark e raciocínio de otimização**, e não estabelecer um percentual universal de ganho.

## 🛠️ Tecnologias

* Databricks
* Delta Lake
* Apache Spark
* PySpark
* SQL
* Liquid Clustering
* Z-ORDER
* OPTIMIZE

## 📁 Estrutura

```text
Delta-Lake-Performance-Benchmark/
│
├── README.md
├── notebooks/
│   └── Delta_Lake_Performance_Benchmark_Free_Edition.ipynb
│   └── resultados_benchmark.md
├── results/
│   └── aggregation
│   │   └── liquid.txt
│   │   └── partitioned.txt
│   │   └── zorder.txt
│   └── join
│   │   └── liquid.txt
│   │   └── partitioned.txt
│   │   └── zorder.txt
│   └── selective_filter
│   │   └── liquid.txt
│   │   └── partitioned.txt
│   │   └── zorder.txt
```

## 🚀 Como reproduzir

1. Abra o notebook em um ambiente Databricks compatível.
2. Execute a seção de preparação dos dados.
3. Execute cada benchmark separadamente.
4. Realize warm-up das queries.
5. Execute múltiplas vezes.
6. Compare a mediana dos resultados.
7. Analise as métricas de execução.
8. Compare os planos físicos do Spark.
9. Analise o estado da tabela Delta antes e depois da otimização.

## 📌 Conclusão

O principal objetivo deste projeto é demonstrar que **otimização de performance em Data Engineering não significa apenas escrever uma query mais eficiente**.

A performance também é consequência da forma como os dados são:

**armazenados → organizados → filtrados → lidos → processados → movimentados entre os nós do cluster.**

# 📚 Literature & References

This project is supported by academic and technical references related to **Delta Lake, Lakehouse architecture, data organization, and distributed query performance**.

## Academic References

1. **Armbrust, M. et al. (2020).**
   *Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores.*
   PVLDB, 13(12), 3411–3424.
   [DOI](https://doi.org/10.14778/3415478.3415560)

2. **Armbrust, M. et al. (2021).**
   *Lakehouse: A New Generation of Open Platforms that Unify Data Warehousing and Advanced Analytics.*
   CIDR 2021.
   [Paper](https://www.vldb.org/cidrdb/2021/lakehouse-a-new-generation-of-open-platforms-that-unify-data-warehousing-and-advanced-analytics.html)

3. **Jain, A. et al. (2023).**
   *Analyzing and Comparing Lakehouse Storage Systems.*
   CIDR 2023.
   [Paper](https://www.vldb.org/cidrdb/papers/2023/p92-jain.pdf)

## Technical References

* [Delta Lake Documentation](https://docs.delta.io/)
* [Databricks — OPTIMIZE](https://docs.databricks.com/aws/en/sql/language-manual/delta-optimize)
* [Databricks — Data Skipping](https://docs.databricks.com/aws/en/tables/data-skipping)
* [Databricks — Liquid Clustering](https://docs.databricks.com/aws/en/delta/clustering)
* [Apache Spark — Performance Tuning](https://spark.apache.org/docs/latest/sql-performance-tuning.html)

> These references provide the theoretical and technical foundation for the benchmark experiments. Results should be interpreted as experimental observations rather than universal performance guarantees.



