# 📊 VoeBem Analytics

> **Pipeline de Engenharia de Dados para Análise de Voos Regulares Brasileiros**

Projeto completo de análise de dados de voos regulares ativos (VRA) no Brasil, utilizando dados públicos da **ANAC (Agência Nacional de Aviação Civil)**. Implementa uma arquitetura moderna de **Data Lakehouse** com camadas **Bronze**, **Silver** e **Gold** no Databricks.

---

## 🎯 Objetivo

Construir um pipeline de dados end-to-end que:
- Ingere dados brutos de voos, aeródromos e empresas aéreas
- Transforma e valida os dados seguindo práticas de engenharia de dados
- Disponibiliza dados prontos para análise, BI e Machine Learning
- Demonstra conceitos essenciais de **Data Engineering** e **Data Governance**

---

## 🏗️ Arquitetura

### Medallion Architecture (Bronze → Silver → Gold)

```
┌─────────────────────────────────────────────────────────────────┐
│  FONTE DE DADOS                                                  │
│  ANAC - Dados Públicos de Aviação Civil                        │
│  • VRA (Voo Regular Ativo) - 12 CSVs mensais                   │
│  • Aeródromos Públicos                                          │
│  • Empresas Aéreas (Nacionais e Estrangeiras)                  │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  🥉 BRONZE - Dados Brutos                                        │
│  Unity Catalog: voebem.bronze                                   │
│  • Ingestão idempotente (full refresh)                          │
│  • Todas as colunas STRING (sem tipagem)                        │
│  • Nenhum filtro aplicado (100% dos dados)                      │
│  • Colunas de auditoria (_arquivo_origem, _ingerido_em)        │
│  Tabelas: vra (1M+ registros), aerodromos, empresas_*, codigos │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  🥈 SILVER - Dados Governados                                    │
│  Unity Catalog: voebem.silver                                   │
│  • Espelho do Bronze com governança aplicada                    │
│  • Tipagem forte (STRING → TIMESTAMP, INT, DATE)                │
│  • Tratamento de valores null e inconsistências                 │
│  • Colunas calculadas (atrasos, recuperação)                    │
│  • Mesma contagem de linhas (sem filtros)                       │
│  • Metadados e comentários em todas as colunas                  │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  🥇 GOLD - Dados Analíticos                                      │
│  Unity Catalog: voebem.gold (futuro)                            │
│  • Agregações e métricas de negócio                             │
│  • Modelos dimensionais (star schema)                           │
│  • Dados otimizados para BI e ML                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📂 Estrutura do Repositório

```
proj eto-voebem-analytics/
├── README.md                    # Documentação principal (este arquivo)
├── notebooks/                   # Notebooks Databricks
│   ├── 03_bronze_vra.ipynb     # Ingestão VRA para Bronze
│   ├── 04_bronze_referencias.ipynb  # Ingestão tabelas de referência
│   ├── 05_silver_espelho.ipynb # Transformação Bronze → Silver
│   ├── desafio-01.ipynb        # Validação da camada Bronze
│   └── desafio-02.ipynb        # Comparação Bronze vs Silver
└── docs/                        # Documentação adicional
```

---

## 📊 Dados e Tabelas

### 🥉 Camada Bronze (`voebem.bronze`)

| Tabela | Registros | Descrição | Fonte |
|--------|-----------|-----------|-------|
| **vra** | 1.014.705 | Voos regulares ativos (12 meses: ago/2025-jul/2026) | VRA_*.csv |
| **aerodromos** | 496 | Aeródromos públicos brasileiros com coordenadas | AerodromosPublicos.csv |
| **empresas_nacionais** | 729 | Cadastro de empresas aéreas nacionais | pda_empresas_aereas_nacionais.csv |
| **empresas_estrangeiras** | 148 | Cadastro de empresas aéreas estrangeiras | pda_empresas_aereas_estrangeiros.csv |
| **codigos_operacao** | 13 | Seed table: códigos DI e tipo de linha | Criado manualmente |

**Total: 1.016.091 registros**

#### Características da Camada Bronze:
- ✅ **Tipagem zero**: todas as colunas são `STRING`
- ✅ **Filtros zero**: nenhuma linha é descartada
- ✅ **Auditoria**: colunas `_arquivo_origem` e `_ingerido_em`
- ✅ **Idempotência**: executar 2x não duplica dados
- ✅ **Volume UC**: `/Volumes/voebem/bronze/arquivos/`

### 🥈 Camada Silver (`voebem.silver`)

| Tabela | Registros | Mudanças vs Bronze |
|--------|-----------|--------------------|
| **vra** | 1.014.705 | Tipagem forte, tratamento de nulls, colunas calculadas |
| **aerodromos** | 496 | Em desenvolvimento |
| **empresas** | 877 | Unificação de nacionais + estrangeiras |

#### Características da Camada Silver:
- ✅ **Espelho governado**: mesma contagem de linhas do Bronze
- ✅ **Tipagem aplicada**: `TIMESTAMP`, `INT`, `DATE`
- ✅ **Tratamento de inconsistências**:
  - Strings `'null'` → `NULL` SQL
  - Timestamps com/sem fração de segundo
- ✅ **Colunas calculadas**:
  - `atraso_partida_min`, `atraso_chegada_min`
  - `minutos_recuperados` (recuperação em voo)
- ✅ **Governança**: comentários, metadados, tags

---

## 🔧 Notebooks e Pipeline

### 1️⃣ Ingestão Bronze

#### **`03_bronze_vra`** - Ingestão VRA
**Objetivo**: Ler 12 CSVs mensais e criar `voebem.bronze.vra`

**Desafios resolvidos**:
- ✅ Separador `;` brasileiro (não `,`)
- ✅ BOM UTF-8 (`EF BB BF`) na primeira linha
- ✅ Linha de cabeçalho "Atualizado em:" antes do header real
- ✅ Nomes de coluna com espaços (inválidos em Delta)
- ✅ Estratégia full-refresh idempotente

```python
# Leitura tratando peculiaridades do arquivo
bruto = spark.read.format("csv") \
    .option("sep", ";") \
    .option("skipRows", 1) \
    .option("header", "true") \
    .load("/Volumes/voebem/bronze/arquivos/vra/*.csv")
```

#### **`04_bronze_referencias`** - Tabelas de Referência
**Objetivo**: Ingerir aeródromos, empresas e códigos

**Desafios resolvidos**:
- ✅ **Aeródromos**: encoding `ISO-8859-1` (não UTF-8), aspas em coordenadas
- ✅ **Empresas**: 2 cadastros separados (nacionais e estrangeiras mantidos distintos no Bronze)
- ✅ **Códigos**: seed table criada manualmente (dados da página HTML da ANAC)

### 2️⃣ Transformação Silver

#### **`05_silver_espelho`** - Bronze → Silver
**Objetivo**: Criar espelho governado com tipagem e validação

**Transformações aplicadas**:
```sql
try_cast(nullif(partida_real, 'null') AS TIMESTAMP) AS partida_real
```
- ✅ Conversão de strings `'null'` para NULL
- ✅ `try_cast` para aceitar múltiplos formatos de timestamp
- ✅ Cálculo de atrasos e recuperação
- ✅ Validação: `COUNT(bronze) = COUNT(silver)` ✅

### 3️⃣ Análise e Validação

#### **`desafio-01`** - Validação Bronze
- Contagem de registros por tabela
- Estrutura de colunas
- Amostras de dados

#### **`desafio-02`** - Comparação Bronze vs Silver
- Colunas transformadas
- Mudanças de tipo
- Valores padronizados
- Volumetria (diferença = 0 ✅)

---

## 🚀 Como Usar

### Pré-requisitos
- Databricks Workspace (AWS, Azure ou GCP)
- Unity Catalog habilitado
- Catálogo `voebem` criado
- Compute Serverless ou cluster Spark

### Execução do Pipeline

```bash
# 1. Ingestão Bronze
Executar notebooks na ordem:
  → 03_bronze_vra
  → 04_bronze_referencias

# 2. Transformação Silver
  → 05_silver_espelho

# 3. Validação
  → desafio-01 (Bronze)
  → desafio-02 (Bronze vs Silver)
```

### Consultas SQL de Exemplo

```sql
-- Top 5 empresas com mais voos
SELECT e.razao_social, COUNT(*) AS total_voos
FROM voebem.silver.vra v
JOIN voebem.bronze.empresas_nacionais e
  ON v.icao_empresa = e.icao
GROUP BY e.razao_social
ORDER BY total_voos DESC
LIMIT 5;

-- Aeroportos com mais atrasos
SELECT a.nome, AVG(v.atraso_partida_min) AS atraso_medio
FROM voebem.silver.vra v
JOIN voebem.bronze.aerodromos a
  ON v.icao_origem = a.icao
WHERE v.atraso_partida_min > 0
GROUP BY a.nome
ORDER BY atraso_medio DESC
LIMIT 10;
```

---

## 💡 Conceitos de Engenharia de Dados Aplicados

### 1. **Medallion Architecture**
- Separação clara entre camadas Bronze (raw), Silver (curated) e Gold (business)
- Cada camada tem responsabilidades e regras específicas

### 2. **Idempotência**
- Pipeline pode ser executado múltiplas vezes sem duplicar dados
- Estratégia full-refresh determinística no Bronze

### 3. **Data Governance**
- Auditoria: rastreamento de origem e timestamp de ingestão
- Metadados: comentários em tabelas e colunas
- Unity Catalog: controle de acesso e linhagem

### 4. **Data Quality**
- Validação de contagem Bronze = Silver
- Tratamento explícito de valores null e inconsistências
- Tipagem forte na camada Silver

### 5. **Schema Evolution**
- Mapeamento explícito de renomeação de colunas
- Compatibilidade com múltiplos formatos de timestamp

### 6. **Separation of Concerns**
- Bronze preserva fidelidade à fonte
- Silver aplica governança sem agregar
- Gold (futuro) conterá métricas de negócio

---

## 📚 Fonte de Dados

**ANAC - Agência Nacional de Aviação Civil**
- [Portal de Dados Abertos da ANAC](https://www.gov.br/anac/pt-br/assuntos/dados-e-estatisticas)
- Conjunto: VRA (Voo Regular Ativo)
- Período: Agosto/2025 a Julho/2026 (12 meses)
- Licença: Dados públicos governamentais

---

## 🛠️ Tecnologias

- **Databricks**: Plataforma de Data Lakehouse
- **Apache Spark**: Engine de processamento distribuído
- **Delta Lake**: Storage layer ACID para Data Lakes
- **Unity Catalog**: Governança e metadados
- **SQL + Python (PySpark)**: Linguagens de desenvolvimento

---

## 📈 Próximos Passos

- [ ] Implementar camada Gold com métricas agregadas
- [ ] Criar modelo dimensional (star schema)
- [ ] Dashboard de BI com métricas de negócio
- [ ] Pipeline de ML para previsão de atrasos
- [ ] Automação com Databricks Workflows
- [ ] Data Quality Monitoring
- [ ] Documentação de linhagem completa

---

## 👤 Autor

**Jonas Gomes Xavier**
- Email: jonasgomesxavier0706@gmail.com
- Projeto: Engenharia de Dados com Databricks

---

## 📝 Licença

Este projeto é para fins educacionais. Os dados utilizados são de domínio público fornecidos pela ANAC.

---

**✨ Documentação criada com Databricks Assistant (Genie Code)**