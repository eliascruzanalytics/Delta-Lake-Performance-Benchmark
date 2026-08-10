# Benchmark: Partitioning vs Z-ORDER vs Liquid Clustering no Databricks

## 📌 Visão Geral

Este projeto apresenta um experimento de **performance no Databricks Free Edition**, comparando três estratégias de organização e otimização de dados em tabelas Delta:

* **Partitioning**
* **Z-ORDER**
* **Liquid Clustering**

O objetivo foi avaliar o comportamento dessas estratégias em diferentes workloads, observando:

* Tempo de execução;
* Quantidade de arquivos;
* File Pruning / Data Skipping;
* Shuffle / Exchange;
* Performance de agregações;
* Performance de filtros seletivos;
* Performance de joins;
* Impacto de Small Files;
* Comportamento do Liquid Clustering.

O experimento foi executado utilizando **Databricks Free Edition**, permitindo observar na prática como diferentes estratégias de organização física dos dados podem impactar a execução de workloads analíticos.

---

## 🏗️ Ambiente

### Databricks

* **Databricks Free Edition**
* **Serverless CPU**
* **Photon habilitado**

A presença de operações como:

```text
PhotonScan
PhotonAgg
PhotonResultStage
PhotonShuffleExchangeSink
```

no plano físico confirma a utilização do **Photon Engine**.

---

# 🔬 Estratégias Avaliadas

Foram avaliadas três estratégias principais.

## 1. Partitioning

A tabela foi fisicamente particionada para organizar os dados.

Essa estratégia pode reduzir a quantidade de dados lidos quando os filtros utilizam a coluna de particionamento.

Por outro lado, uma estratégia inadequada de particionamento pode provocar:

* excesso de arquivos;
* small files;
* maior overhead de leitura;
* maior quantidade de operações de coordenação;
* problemas relacionados à cardinalidade da coluna utilizada no particionamento.

---

## 2. Z-ORDER

Foi utilizado:

```sql
OPTIMIZE vendas_zorder
ZORDER BY (cliente_id, data_id);
```

O Z-ORDER reorganiza fisicamente os dados para melhorar a **localidade dos valores utilizados nos filtros**.

Neste experimento, as colunas utilizadas foram:

```text
cliente_id
data_id
```

---

## 3. Liquid Clustering

Foi utilizada uma tabela com:

```text
clusteringColumns: [cliente_id, data_id]
```

O processo de otimização foi executado através de:

```sql
OPTIMIZE vendas_liquid;
```

A tabela apresentou:

```text
1 arquivo
2.168.422 bytes
```

indicando que o processo de clustering foi executado corretamente.

---

# ⚙️ Execução do OPTIMIZE

Os dois processos de otimização foram executados com sucesso:

```sql
OPTIMIZE vendas_zorder
ZORDER BY (cliente_id, data_id);
```

✅ Executado com sucesso.

E:

```sql
OPTIMIZE vendas_liquid;
```

✅ Executado com sucesso.

> **Observação:** o notebook não capturou os tempos individuais de execução dos comandos `OPTIMIZE`. Portanto, a análise de performance abaixo considera principalmente o tempo dos workloads após a otimização.

---

# 📁 Quantidade de Arquivos

Um dos pontos analisados foi o número de arquivos produzidos por cada estratégia.

| Estratégia        | Arquivos |         Tamanho |
| ----------------- | -------: | --------------: |
| Partitioned       |       20 | 2.141.423 bytes |
| Z-ORDER           |        1 | 2.184.508 bytes |
| Liquid Clustering |        1 | 2.168.422 bytes |

### Small Files

Também foi realizado um experimento específico para avaliar o problema de **Small Files**.

| Situação           | Arquivos |         Tamanho |
| ------------------ | -------: | --------------: |
| Antes do OPTIMIZE  |       80 | 2.438.601 bytes |
| Depois do OPTIMIZE |        1 | 2.188.768 bytes |

O resultado demonstra claramente o efeito da compactação dos arquivos.

```text
80 arquivos
     ↓
   OPTIMIZE
     ↓
1 arquivo
```

Isso reduz o overhead associado ao gerenciamento e leitura de uma grande quantidade de arquivos pequenos.

---

# 📊 Resultados do Benchmark

## A. Aggregation

### Mediana do tempo de execução

| Estratégia  |       Tempo |
| ----------- | ----------: |
| 🥇 Z-ORDER  | **0.569 s** |
| Partitioned |     0.600 s |
| Liquid      |     0.608 s |

### Resultado

O **Z-ORDER apresentou o melhor resultado** para a agregação.

```text
Z-ORDER      0.569s  🥇
Partitioned  0.600s
Liquid       0.608s
```

A diferença é pequena, mas o Z-ORDER apresentou a menor mediana.

---

# 🔎 B. Selective Filter

Neste experimento foram utilizados filtros seletivos sobre:

```text
cliente_id
data_id
```

### Mediana do tempo de execução

| Estratégia  |       Tempo |
| ----------- | ----------: |
| 🥇 Z-ORDER  | **0.521 s** |
| Partitioned |     0.627 s |
| Liquid      |     0.724 s |

### Resultado

O **Z-ORDER apresentou novamente o melhor resultado**.

```text
Z-ORDER      0.521s  🥇
Partitioned  0.627s
Liquid       0.724s
```

Neste workload específico, o Liquid Clustering apresentou desempenho inferior ao Z-ORDER.

---

# 🔗 C. Join

O experimento de Join apresentou um comportamento diferente.

### Mediana do tempo de execução

| Estratégia  |       Tempo |
| ----------- | ----------: |
| 🥇 Liquid   | **0.816 s** |
| Z-ORDER     |     0.948 s |
| Partitioned |     1.390 s |

### Observação importante

A estratégia Partitioned apresentou um **outlier extremo de aproximadamente 497 segundos** em uma das execuções.

Apesar desse outlier, a mediana permaneceu em:

```text
1.390s
```

contra:

```text
Liquid      0.816s
Z-ORDER     0.948s
```

### Resultado

Neste workload:

```text
Liquid       0.816s  🥇
Z-ORDER      0.948s
Partitioned  1.390s
```

O **Liquid Clustering apresentou o melhor resultado**.

---

# ⚔️ D. Z-ORDER × Liquid Clustering

Foi realizada uma comparação direta entre as duas estratégias.

| Estratégia |       Tempo |
| ---------- | ----------: |
| Liquid     | **0.556 s** |
| Z-ORDER    |     0.565 s |

A diferença foi extremamente pequena:

```text
Liquid       0.556s
Z-ORDER      0.565s
```

Na prática, podemos considerar um:

> **Empate técnico para este workload.**

---

# 🔄 Shuffle / Exchange

A análise dos planos físicos demonstrou a presença de operações de:

```text
Exchange
Shuffle
Broadcast Hash Join
```

em diferentes workloads.

---

## Selective Filter

### Partitioned

Foi identificado:

```text
PhotonShuffleExchangeSink
        ↓
SinglePartition
        ↓
Aggregation
```

Ou seja, houve uma operação de **shuffle/exchange** antes da agregação.

---

### Z-ORDER

O plano apresentou:

```text
PhotonAgg
     ↓
PhotonResultStage
```

Sem a presença do mesmo shuffle observado na estratégia Partitioned.

---

### Liquid Clustering

Também apresentou:

```text
PhotonAgg
     ↓
PhotonResultStage
```

sem o shuffle observado no cenário Partitioned.

---

## Resultado

Para esse workload específico:

```text
Partitioned
     ↓
Shuffle
     ↓
Aggregation
```

enquanto:

```text
Z-ORDER
     ↓
Local Aggregation
```

e:

```text
Liquid
     ↓
Local Aggregation
```

Isso ajuda a explicar parte da diferença observada nos tempos.

---

# 🔗 Shuffle no Join

No workload de Join, todas as estratégias apresentaram:

```text
Broadcast Hash Join
```

com operações relacionadas a shuffle.

Foram identificadas duas operações principais:

### 1. Broadcast

```text
EXECUTOR_BROADCAST
```

A tabela `clientes` foi enviada para os executores.

### 2. Hash Partitioning

Foi identificado:

```text
hashpartitioning(segmento, 32)
```

indicando uma redistribuição dos dados para permitir a agregação final.

Representação simplificada:

```text
                    ┌──────────────┐
                    │   Clientes   │
                    └──────┬───────┘
                           │
                       Broadcast
                           │
                           ▼
┌─────────────┐     ┌───────────────┐
│   Vendas    │ ──► │ Broadcast Hash│
└─────────────┘     │     Join      │
                    └───────┬───────┘
                            │
                            ▼
                    hashpartitioning
                       (segmento)
                            │
                            ▼
                      Aggregation
```

---

# 🚀 Data Skipping / File Pruning

Foi identificada evidência de **Data Skipping / File Pruning** nas três estratégias.

No plano PhotonScan foram observados filtros como:

```text
DictionaryFilters:
[
  (cliente_id = 12345),
  ((data_id >= 100) AND (data_id <= 120))
]
```

Esses filtros estavam presentes em:

* ✅ Partitioned
* ✅ Z-ORDER
* ✅ Liquid Clustering

Portanto, todas as estratégias conseguiram utilizar mecanismos de filtragem em nível de arquivo.

---

# 📦 Diferença na Quantidade de Dados / Arquivos

Embora todas as estratégias apresentassem evidências de Data Skipping, a organização física dos arquivos era diferente.

### Partitioned

```text
20 arquivos
```

O mecanismo precisou trabalhar sobre múltiplos arquivos/partições.

### Z-ORDER

```text
1 arquivo
```

Após o `OPTIMIZE`, os dados estavam consolidados.

### Liquid Clustering

```text
1 arquivo
```

Também apresentou consolidação dos dados após o `OPTIMIZE`.

---

# 🐢 Principais Gargalos Identificados

## 1. Shuffle no Partitioned

No workload de filtro seletivo, o cenário Partitioned apresentou uma operação de:

```text
PhotonShuffleExchangeSink
```

enquanto Z-ORDER e Liquid conseguiram executar a agregação sem esse mesmo shuffle.

Esse comportamento pode contribuir para o aumento do custo computacional.

---

## 2. Join instável no Partitioned

O cenário Partitioned apresentou um outlier extremo:

```text
497 segundos
```

Embora seja necessário investigar o motivo específico desse outlier, a presença de múltiplos arquivos e o comportamento do ambiente Serverless podem contribuir para maior variabilidade.

A mediana também foi superior:

```text
Partitioned = 1.390s
Liquid      = 0.816s
Z-ORDER     = 0.948s
```

---

## 3. Liquid Clustering no Selective Filter

Neste experimento:

```text
Liquid = 0.724s
Z-ORDER = 0.521s
```

O Liquid apresentou desempenho inferior.

Uma hipótese levantada durante a análise foi a presença de informações estatísticas incompletas no plano:

```text
missing = vendas_liquid
```

Isso pode indicar que o ambiente Free Edition possui limitações ou diferenças na forma como algumas informações de otimização são expostas/gerenciadas.

> Essa observação deve ser tratada como hipótese de investigação, e não como causa definitivamente comprovada.

---

## 4. I/O e quantidade de arquivos

O ambiente Free Edition possui recursos computacionais limitados.

A diferença:

```text
Partitioned → 20 arquivos
Z-ORDER     → 1 arquivo
Liquid      → 1 arquivo
```

pode aumentar o overhead de I/O e coordenação no cenário Partitioned.

---

# 🧠 Interpretação dos Resultados

Os resultados demonstram que **não existe uma estratégia universalmente melhor**.

A escolha depende do:

* padrão de acesso;
* cardinalidade;
* seletividade dos filtros;
* volume de dados;
* frequência dos joins;
* distribuição dos dados;
* quantidade de arquivos;
* padrão de ingestão;
* necessidade de manutenção da tabela.

Neste benchmark específico:

| Workload         | Melhor estratégia |
| ---------------- | ----------------- |
| Aggregation      | 🥇 Z-ORDER        |
| Selective Filter | 🥇 Z-ORDER        |
| Join             | 🥇 Liquid         |
| Z-ORDER × Liquid | 🤝 Empate técnico |

---

# 🏆 Conclusão

Para o workload utilizado neste experimento:

### Z-ORDER

Foi a estratégia mais consistente.

Venceu:

```text
Aggregation
Selective Filter
```

Apresentando:

```text
0.569s
0.521s
```

respectivamente.

---

### Liquid Clustering

Apresentou o melhor resultado no:

```text
Join
```

com:

```text
0.816s
```

Além disso, apresentou praticamente empate com Z-ORDER em uma comparação direta:

```text
Liquid  = 0.556s
Z-ORDER = 0.565s
```

---

### Partitioning

Foi a estratégia com maior desvantagem neste benchmark.

Principais pontos observados:

* 20 arquivos após otimização;
* maior overhead de arquivos;
* presença de shuffle no filtro seletivo;
* maior mediana no Join;
* outlier extremo de aproximadamente 497 segundos.

Isso não significa que Partitioning seja uma estratégia ruim. Significa que **o particionamento utilizado neste experimento não apresentou vantagem para o padrão de acesso testado**.

---

# 🎯 Conclusão Prática

Para este volume de dados e padrão de acesso:

> **Z-ORDER apresentou o melhor custo-benefício geral no Databricks Free Edition.**

Resumo:

```text
                  BENCHMARK
                      │
       ┌──────────────┼──────────────┐
       │              │              │
   Partitioned     Z-ORDER        Liquid
       │              │              │
       ▼              ▼              ▼
  20 arquivos     1 arquivo      1 arquivo
       │              │              │
       ▼              ▼              ▼
  Mais overhead    Melhor        Melhor Join
                  geral
```

### Ranking geral observado

🥇 **Z-ORDER** — melhor consistência geral
🥈 **Liquid Clustering** — excelente em Join
🥉 **Partitioning** — menos eficiente neste workload

---

# ⚠️ Limitações do Experimento

Os resultados devem ser interpretados considerando que o benchmark foi executado no:

```text
Databricks Free Edition
Serverless CPU
Photon
```

Portanto, os resultados não devem ser extrapolados diretamente para clusters produtivos maiores.

Além disso:

* o volume de dados utilizado é relativamente pequeno;
* o ambiente é compartilhado/serverless;
* o comportamento de cache pode influenciar os resultados;
* o benchmark não representa necessariamente workloads de produção;
* os tempos de `OPTIMIZE` não foram capturados individualmente;
* o outlier de 497 segundos merece uma investigação específica.

Para um benchmark produtivo, seria recomendável testar:

* maior volume de dados;
* diferentes cardinalidades;
* diferentes níveis de seletividade;
* múltiplos tamanhos de cluster;
* Photon vs não-Photon;
* diferentes padrões de Join;
* Data Skipping;
* Spark UI;
* SQL Warehouse;
* métricas de I/O;
* métricas de shuffle;
* número de bytes lidos;
* número de arquivos efetivamente lidos.

---

# 📚 Principais Conceitos Demonstrados

Este experimento permite estudar na prática conceitos importantes de **Data Engineering e Databricks**:

* Delta Lake
* Delta Tables
* Partitioning
* Z-ORDER
* Liquid Clustering
* OPTIMIZE
* File Compaction
* Small Files Problem
* Data Skipping
* File Pruning
* Shuffle
* Exchange
* Broadcast Hash Join
* Photon Engine
* Query Execution Plan
* Performance Tuning
* I/O Optimization
* Data Layout

---

# 📂 Estrutura Sugerida do Projeto

```text
.
├── README.md
├── notebooks/
│   └── benchmark_partition_zorder_liquid.dbc
├── results/
│   ├── aggregation.csv
│   ├── selective_filter.csv
│   ├── join.csv
│   └── zorder_vs_liquid.csv
└── docs/
    └── execution_plans/
```

---

# 💡 Principal Insight

O principal aprendizado deste experimento é que **otimização física de dados deve ser orientada pelo workload**.

Não basta afirmar:

> "Z-ORDER é melhor que Partitioning"

ou:

> "Liquid Clustering é melhor que Z-ORDER".

A abordagem correta é analisar:

```text
Workload
   ↓
Padrão de acesso
   ↓
Filtros / Joins
   ↓
Data Layout
   ↓
Query Plan
   ↓
Shuffle / I/O
   ↓
Benchmark
   ↓
Escolha da estratégia
```

Neste experimento, essa abordagem levou à conclusão de que:

> **Z-ORDER apresentou a melhor performance geral para o padrão de acesso testado, enquanto Liquid Clustering demonstrou vantagem específica no workload de Join.**
