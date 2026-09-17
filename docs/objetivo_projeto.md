# Objetivo do Projeto VoeBem Analytics

## Visão Geral

A **VoeBem** é uma empresa de analytics especializada em aviação civil brasileira, com foco em fornecer insights baseados em dados para responder perguntas estratégicas do setor aéreo.

## Fonte de Dados

Trabalhamos com datasets públicos da **ANAC (Agência Nacional de Aviação Cívil)**, especificamente:

* **VRA (Voo Regular Ativo)**: Aproximadamente 1 milhão de registros cobrindo 12 meses de operações (agosto/2025 a julho/2026)
* **Tabelas de Referência**:
  * Aeródromos Públicos (~496 aeroportos)
  * Empresas Aéreas Nacionais (~729 registros)
  * Empresas Aéreas Estrangeiras (~148 registros)
  * Códigos de Operação e Justificativas

## Escopo de Análise

Nosso foco principal está em analisar e responder perguntas sobre:

* ✈️ **Cancelamentos de voos**: Padrões, causas e empresas mais afetadas
* ⏱️ **Atrasos**: Partidas e chegadas (previstas vs. reais)
* 🏢 **Desempenho por companhia aérea**: GOL, AZUL, LATAM e outras
* 🛫 **Análise por aeródromo**: Origem e destino, aeroportos mais movimentados
* 📊 **Situações operacionais**: Voos realizados, cancelados, atrasados
* 🔍 **Justificativas de irregularidades**: Motivos técnicos, climáticos, operacionais

## Arquitetura Técnica

### Plataforma
* **Databricks**: Processamento de dados distribuído com Apache Spark
* **Delta Lake**: Formato de tabelas confiável e ACID
* **Unity Catalog**: Governança de dados centralizada (catálogo `voebem`)

### Arquitetura Medalhão Implementada

**🥉 Camada Bronze** (implementada):
* Dados brutos preservados exatamente como fornecidos pela ANAC
* Todas as colunas como strings (sem tipagem prematura)
* Colunas de auditoria: `_arquivo_origem` e `_ingerido_em`
* Tratamento de encoding (ISO-8859-1 e UTF-8)
* Carga idempotente com full refresh
* Tabelas: `voebem.bronze.vra`, `voebem.bronze.aerodromos`, `voebem.bronze.empresas_nacionais`, `voebem.bronze.empresas_estrangeiras`

**🥈 Camada Silver** (planejada):
* Dados limpos, tipados e validados
* Joins entre VRA e tabelas de referência
* Tradução de códigos para descrições legíveis
* Cálculo de atrasos (diferença entre previsto e real)

**🥇 Camada Gold** (planejada):
* Agregações por empresa, aeródromo, período
* Métricas de negócio: taxa de pontualidade, tempo médio de atraso
* Dados prontos para dashboards e relatórios executivos

