# 🏗️ Arquitetura do Projeto VoeBem Analytics

## Visão Geral

O **VoeBem Analytics** implementa uma arquitetura moderna de **Data Lakehouse** seguindo o padrão **Medallion Architecture** (Bronze → Silver → Gold) no Databricks.

---

## 1. Arquitetura de Camadas

### 🥉 Camada Bronze - Raw Data Layer

**Propósito**: Preservar a fidelidade absoluta aos dados de origem.

#### Princípios de Design:
- ✅ Aceitar tudo / ❌ Recusar nada
- ✅ Preservar granularidade / ❌ Agregar
- ✅ Auditabilidade / ❌ Transformar semântica

#### Regras Invioláveis:
1. **Zero Tipagem**: Todas as colunas são STRING
2. **Zero Filtros**: Nenhuma linha é descartada
3. **Auditoria Completa**: Rastreamento de origem e timestamp
4. **Idempotência**: Execução múltipla = mesmo resultado

#### Estratégia de Carga:
```python
# Full Refresh Determinístico
.write.mode("overwrite")
.option("overwriteSchema", "true")
.saveAsTable(tabela)
```

**Por que Full Refresh e não Append + Dedup?**
- Fonte imutável e completa (12 arquivos mensais)
- VRA não tem chave natural única
- Deduplicação no Bronze = perda de informação
- Estado final é derivável da entrada completa

---

### 🥈 Camada Silver - Curated Data Layer

**Propósito**: Espelho governado do Bronze com tipagem e metadados.

#### Regra Fundamental:
> A Silver é espelho: mesma tabela, mesmo grão, mesma contagem de linhas.

#### Transformações Permitidas:
- ✅ Tipagem (STRING → TIMESTAMP, INT, DATE)
- ✅ Tratamento de nulls e inconsistências
- ✅ Colunas calculadas (dentro da linha)
- ✅ Unificação de cadastros
- ✅ Governança (metadados, comments, tags)

#### Transformações Proibidas:
- ❌ Filtros (WHERE) que removem linhas
- ❌ Agregações (GROUP BY)
- ❌ Decisões de negócio (limiares)
- ❌ JOINs que mudam granularidade

---

### 🥇 Camada Gold - Business Data Layer

**Propósito**: Dados otimizados para consumo (BI, ML, APIs).

**Características**:
- ✅ Agregações e métricas de negócio
- ✅ Modelos dimensionais (Star Schema)
- ✅ Denormalização para performance
- ✅ Features para Machine Learning

---

## 2. Unity Catalog Structure

```
voebem                           # Catalog
├── bronze                       # Schema: Raw data
│   ├── vra                      # 1.014.705 registros
│   ├── aerodromos               # 496 registros
│   ├── empresas_nacionais       # 729 registros
│   ├── empresas_estrangeiras    # 148 registros
│   └── codigos_operacao         # 13 registros
├── silver                       # Schema: Curated data
│   ├── vra                      # 1.014.705 registros
│   └── empresas                 # 877 registros
└── gold                         # Schema: Business data (futuro)
```

---

## 3. Data Storage Architecture

### Unity Catalog Volumes

```
/Volumes/voebem/bronze/arquivos/
├── vra/
│   ├── VRA_20258.csv           # Agosto 2025
│   ├── VRA_20259.csv           # Setembro 2025
│   ├── VRA_202510.csv          # Outubro 2025
│   ├── VRA_202511.csv          # Novembro 2025
│   ├── VRA_202512.csv          # Dezembro 2025
│   ├── VRA_20261.csv           # Janeiro 2026
│   ├── VRA_20262.csv           # Fevereiro 2026
│   ├── VRA_20263.csv           # Março 2026
│   ├── VRA_20264.csv           # Abril 2026
│   ├── VRA_20265.csv           # Maio 2026
│   ├── VRA_20266.csv           # Junho 2026
│   └── VRA_20267.csv           # Julho 2026
└── referencias/
    ├── AerodromosPublicos.csv
    ├── pda_empresas_aereas_nacionais.csv
    └── pda_empresas_aereas_estrangeiros.csv
```

---

## 4. Data Flow Pipeline

```
┌────────────────────────────────────┐
│  ANAC Data Sources (CSV Files)     │
└──────────────┬─────────────────────┘
               ▼
┌────────────────────────────────────┐
│  Unity Catalog Volume              │
│  /Volumes/voebem/bronze/arquivos/  │
└──────────────┬─────────────────────┘
               ▼
┌────────────────────────────────────┐
│  Notebook: 03_bronze_vra           │
│  • Read CSV + Handle encodings     │
│  • Normalize column names          │
│  • Add audit columns               │
└──────────────┬─────────────────────┘
               ▼
┌────────────────────────────────────┐
│  Bronze Layer (voebem.bronze)      │
│  • All columns STRING              │
│  • No filters, no aggregations     │
└──────────────┬─────────────────────┘
               ▼
┌────────────────────────────────────┐
│  Notebook: 05_silver_espelho       │
│  • Type casting                    │
│  • Handle nulls                    │
│  • Calculate derived columns       │
└──────────────┬─────────────────────┘
               ▼
┌────────────────────────────────────┐
│  Silver Layer (voebem.silver)      │
│  • Strong typing                   │
│  • Same granularity as Bronze      │
└────────────────────────────────────┘
```

---

## 5. Idempotência e Reprocessamento

### Estratégia de Idempotência

**Bronze Layer:**
```python
bronze.write.mode("overwrite").option("overwriteSchema", "true").saveAsTable(TABELA)
```

**Silver Layer:**
```sql
CREATE OR REPLACE TABLE silver.vra AS SELECT ... FROM bronze.vra
```

---

## 6. Data Governance

### Auditoria Automática
```python
.withColumn("_arquivo_origem", F.col("_metadata.file_name"))
.withColumn("_ingerido_em", F.current_timestamp())
```

### Unity Catalog Features:
- **Access Control**: Row/column-level permissions
- **Data Lineage**: Rastreamento automático
- **Schema Evolution**: Versionamento de schemas
- **Time Travel**: Delta Lake histórico
- **Audit Logs**: Todas as operações registradas

---

## 7. Decisões Arquiteturais Importantes

### 1. Por que Bronze preserva tudo?
**Decisão**: Bronze = fonte de verdade imutável  
**Razão**: Permite reprocessamento com novas regras sem re-ingestão

### 2. Por que Silver não filtra?
**Decisão**: Silver = espelho governado, não subconjunto  
**Razão**: Evita perguntas que a camada não pode mais responder

### 3. Por que duas tabelas de empresas no Bronze?
**Decisão**: Manter cadastros separados (nacionais/estrangeiras)  
**Razão**: São fontes distintas na ANAC, união só faz sentido no Silver

### 4. Por que full refresh e não CDC?
**Decisão**: Sobrescrever 12 arquivos mensais a cada carga  
**Razão**: Fonte imutável, sem chave única confiável, simplicidade operacional

### 5. Por que seed table de códigos?
**Decisão**: Tabela manual de códigos_operacao  
**Razão**: Dados vêm de HTML da ANAC, não de CSV estruturado

---

**Documentação técnica do projeto VoeBem Analytics**  
Última atualização: 2026-09-17
