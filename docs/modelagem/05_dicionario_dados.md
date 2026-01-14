# 📖 Dicionário de Dados - DW E-commerce

> Catálogo completo de todos os campos, tipos e significados

## 📋 Índice

- [Como Usar Este Documento](#como-usar-este-documento)
- [Convenções e Padrões](#convenções-e-padrões)
- [Dimensões](#dimensões)
- [Tabelas Fato](#tabelas-fato)
- [Views Auxiliares](#views-auxiliares)
- [Glossário de Termos](#glossário-de-termos)

---

## 📚 Como Usar Este Documento

### Estrutura das Entradas

Cada campo está documentado com:

| Elemento | Descrição |
|----------|-----------|
| **Nome** | Nome técnico do campo |
| **Tipo** | Tipo de dados SQL Server |
| **Obrigatório** | NULL ou NOT NULL |
| **Descrição** | O que o campo representa |
| **Valores** | Valores válidos ou exemplo |
| **Regras** | Constraints e validações |
| **Origem** | Sistema fonte (quando aplicável) |

### Navegação Rápida

- 🔑 = Primary Key
- 🔗 = Foreign Key
- 📊 = Métrica (medida)
- 📝 = Atributo descritivo
- 🏷️ = Flag (booleano)
- 🗓️ = Campo temporal

---

## 📐 Convenções e Padrões

### Nomenclatura

```
Padrão de Nomes:
├─ Tabelas: MAIÚSCULAS com prefixo (DIM_, FACT_)
├─ Campos: snake_case (minúsculas com underscore)
├─ PKs: [tabela]_id (ex: cliente_id)
├─ FKs: mesmo nome da PK referenciada
└─ Views: prefixo VW_
```

### Tipos de Dados

| Tipo SQL Server | Uso | Exemplo |
|----------------|-----|---------|
| `INT` | IDs, contadores | `cliente_id INT` |
| `BIGINT` | IDs de facts (grande volume) | `venda_id BIGINT` |
| `VARCHAR(n)` | Textos variáveis | `nome_cliente VARCHAR(200)` |
| `CHAR(n)` | Textos fixos | `estado CHAR(2)` |
| `DECIMAL(p,s)` | Valores monetários | `valor_total DECIMAL(15,2)` |
| `DATE` | Datas | `data_cadastro DATE` |
| `DATETIME` | Data+hora | `data_inclusao DATETIME` |
| `BIT` | Booleanos | `eh_ativo BIT` |

### Surrogate Keys

**Padrão:** INT IDENTITY(1,1)

- Todas dimensões: `[tabela]_id INT`
- Todas facts: `[tabela]_id BIGINT`
- Sempre incremento automático
- Sempre NOT NULL PRIMARY KEY

---

## 📐 DIMENSÕES

## DIM_DATA - Dimensão Temporal

**Schema:** `dim.DIM_DATA`  
**Registros:** ~3.650 (10 anos: 2020-2030)  
**Crescimento:** Planejado (adição manual de anos futuros)

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **data_id** | INT | ✓ | PK - Formato YYYYMMDD | `20241231` | PRIMARY KEY, formato date integer |
| 📝 **data_completa** | DATE | ✓ | Data no formato padrão | `2024-12-31` | UNIQUE |
| 📝 **ano** | INT | ✓ | Ano (4 dígitos) | `2024` | `>= 2020 AND <= 2030` |
| 📝 **trimestre** | INT | ✓ | Trimestre do ano | `4` | `BETWEEN 1 AND 4` |
| 📝 **mes** | INT | ✓ | Mês (número) | `12` | `BETWEEN 1 AND 12` |
| 📝 **nome_mes** | VARCHAR(20) | ✓ | Nome do mês por extenso | `"Dezembro"` | Lista fixa de 12 meses |
| 📝 **dia_mes** | INT | ✓ | Dia do mês | `31` | `BETWEEN 1 AND 31` |
| 📝 **dia_ano** | INT | ✓ | Dia do ano (ordinal) | `365` | `BETWEEN 1 AND 366` |
| 📝 **dia_semana** | INT | ✓ | Dia da semana (1=Dom) | `7` | `BETWEEN 1 AND 7` |
| 📝 **nome_dia_semana** | VARCHAR(20) | ✓ | Nome do dia por extenso | `"Sábado"` | Lista fixa de 7 dias |
| 🏷️ **eh_fim_de_semana** | BIT | ✓ | 1=Sáb/Dom, 0=Útil | `1` | Calculado: dia_semana IN (1,7) |
| 🏷️ **eh_feriado** | BIT | ✓ | 1=Feriado nacional | `1` | Lista de feriados brasileiros |
| 📝 **nome_feriado** | VARCHAR(50) | ✗ | Nome do feriado | `"Natal"` | NULL se não é feriado |

**Hierarquia Temporal:**
```
ano → trimestre → mes → dia_mes
                      → dia_semana
```

**Origem:** Gerada pelo script (não vem de sistema fonte)

---

## DIM_CLIENTE - Dimensão Cliente

**Schema:** `dim.DIM_CLIENTE`  
**Registros Estimados:** 10.000 - 1.000.000  
**Crescimento:** Alto (novos clientes diariamente)  
**SCD Type:** Type 1 (sobrescreve)

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **cliente_id** | INT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **cliente_original_id** | INT | ✓ | Natural Key (sistema CRM) | `45123` | UNIQUE, origem: CRM |
| 📝 **nome_cliente** | VARCHAR(200) | ✓ | Nome completo ou razão social | `"João Silva"` | `LEN >= 3` |
| 📝 **email** | VARCHAR(255) | ✓ | Email principal | `"joao@email.com"` | UNIQUE, formato email |
| 📝 **tipo_cliente** | VARCHAR(20) | ✓ | Pessoa Física ou Jurídica | `"PF"` | `IN ('PF', 'PJ')` |
| 📝 **segmento** | VARCHAR(30) | ✗ | Classificação de valor | `"Ouro"` | `IN ('Bronze','Prata','Ouro','Platinum','Corporativo','Enterprise')` |
| 📝 **pais** | VARCHAR(50) | ✓ | País de origem | `"Brasil"` | Default: 'Brasil' |
| 📝 **estado** | CHAR(2) | ✗ | UF do cliente | `"SP"` | `LEN = 2`, códigos IBGE |
| 📝 **cidade** | VARCHAR(100) | ✗ | Cidade do cliente | `"São Paulo"` | - |
| 🗓️ **data_cadastro** | DATE | ✓ | Data de registro no sistema | `2024-01-15` | `<= GETDATE()` |
| 🗓️ **data_ultima_compra** | DATE | ✗ | Última transação | `2024-12-10` | Atualizado por ETL |
| 🏷️ **eh_ativo** | BIT | ✓ | Status do cliente | `1` | Default: 1, 0=Inativo |

**Origem:** Sistema CRM (Salesforce/Dynamics)

**Segmentação por Valor (Regra de Negócio):**
- Bronze: < R$ 1.000 lifetime value
- Prata: R$ 1.000 - R$ 10.000
- Ouro: R$ 10.000 - R$ 50.000
- Platinum: > R$ 50.000
- Corporativo: PJ pequeno/médio porte
- Enterprise: PJ grande porte

---

## DIM_PRODUTO - Dimensão Produto

**Schema:** `dim.DIM_PRODUTO`  
**Registros Estimados:** 1.000 - 100.000  
**Crescimento:** Médio (novos produtos mensalmente)  
**SCD Type:** Type 1

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **produto_id** | INT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **produto_original_id** | INT | ✓ | Natural Key (sistema ERP) | `78945` | UNIQUE, origem: ERP |
| 📝 **codigo_sku** | VARCHAR(50) | ✓ | Stock Keeping Unit | `"DELL-INSP-15"` | UNIQUE |
| 📝 **nome_produto** | VARCHAR(200) | ✓ | Nome descritivo completo | `"Notebook Dell Inspiron 15"` | - |
| 📝 **categoria** | VARCHAR(50) | ✓ | Categoria principal (nível 1) | `"Eletrônicos"` | - |
| 📝 **subcategoria** | VARCHAR(50) | ✗ | Subcategoria (nível 2) | `"Notebooks"` | - |
| 📝 **marca** | VARCHAR(50) | ✗ | Marca do produto | `"Dell"` | - |
| 🔗 **fornecedor_id** | INT | ✗ | ID do fornecedor | `123` | Origem: ERP |
| 📝 **nome_fornecedor** | VARCHAR(100) | ✗ | Nome do fornecedor (desnorm.) | `"Dell Inc."` | Desnormalizado |
| 📊 **peso_kg** | DECIMAL(10,2) | ✗ | Peso em quilogramas | `2.50` | `>= 0` |
| 📝 **dimensoes** | VARCHAR(50) | ✗ | Dimensões físicas | `"35x25x2 cm"` | Formato livre |
| 📊 **preco_sugerido** | DECIMAL(10,2) | ✗ | Preço de tabela atual | `3500.00` | `> 0` |
| 📊 **custo_medio** | DECIMAL(10,2) | ✗ | Custo médio unitário | `2000.00` | `> 0` |
| 🏷️ **eh_ativo** | BIT | ✓ | Produto ativo no catálogo | `1` | Default: 1 |

**Hierarquia de Categorização:**
```
categoria → subcategoria → marca → produto → SKU
```

**Origem:** Sistema ERP (SAP/TOTVS)

**Regra de Margem:**
```sql
margem = (preco_sugerido - custo_medio) / preco_sugerido * 100
```

---

## DIM_REGIAO - Dimensão Geográfica

**Schema:** `dim.DIM_REGIAO`  
**Registros Estimados:** 100 - 5.000 (municípios brasileiros)  
**Crescimento:** Muito baixo (raramente adiciona cidades)  
**SCD Type:** Type 1

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **regiao_id** | INT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **regiao_original_id** | INT | ✓ | Natural Key | `3550308` | UNIQUE, código IBGE |
| 📝 **pais** | VARCHAR(50) | ✓ | País | `"Brasil"` | Default: 'Brasil' |
| 📝 **regiao_pais** | VARCHAR(30) | ✗ | Região do país | `"Sudeste"` | `IN ('Norte','Nordeste','Centro-Oeste','Sudeste','Sul')` |
| 📝 **estado** | CHAR(2) | ✓ | Sigla UF | `"SP"` | `LEN = 2` |
| 📝 **nome_estado** | VARCHAR(50) | ✓ | Nome completo do estado | `"São Paulo"` | - |
| 📝 **cidade** | VARCHAR(100) | ✓ | Nome do município | `"São Paulo"` | - |
| 📝 **codigo_ibge** | VARCHAR(10) | ✗ | Código IBGE de 7 dígitos | `"3550308"` | Formato: XXXXXXX |
| 📝 **cep_inicial** | VARCHAR(10) | ✗ | CEP inicial da região | `"01000-000"` | Formato: XXXXX-XXX |
| 📝 **cep_final** | VARCHAR(10) | ✗ | CEP final da região | `"05999-999"` | Formato: XXXXX-XXX |
| 📝 **ddd** | CHAR(2) | ✗ | Código DDD telefônico | `"11"` | `LEN = 2` |
| 📊 **populacao_estimada** | INT | ✗ | População do município | `12325232` | `> 0`, fonte: IBGE |
| 📊 **area_km2** | DECIMAL(10,2) | ✗ | Área em km² | `1521.11` | `> 0` |
| 📊 **densidade_demografica** | DECIMAL(10,2) | ✗ | Habitantes por km² | `8097.99` | Calculado: pop/área |
| 📝 **tipo_municipio** | VARCHAR(30) | ✗ | Classificação | `"Capital"` | `IN ('Capital','Interior','Região Metropolitana')` |
| 📝 **porte_municipio** | VARCHAR(20) | ✗ | Porte por população | `"Grande"` | `IN ('Grande','Médio','Pequeno')` |
| 📊 **pib_per_capita** | DECIMAL(10,2) | ✗ | PIB per capita em R$ | `52796.00` | Fonte: IBGE |
| 📊 **idh** | DECIMAL(4,3) | ✗ | Índice Desenv. Humano | `0.805` | `BETWEEN 0 AND 1` |
| 📊 **latitude** | DECIMAL(10,7) | ✗ | Coordenada geográfica | `-23.5505199` | Formato decimal |
| 📊 **longitude** | DECIMAL(10,7) | ✗ | Coordenada geográfica | `-46.6333094` | Formato decimal |
| 📝 **fuso_horario** | VARCHAR(50) | ✗ | Timezone IANA | `"America/Sao_Paulo"` | - |
| 🗓️ **data_cadastro** | DATETIME | ✓ | Data de criação do registro | `2024-01-01 00:00:00` | Default: GETDATE() |
| 🗓️ **data_ultima_atualizacao** | DATETIME | ✓ | Última modificação | `2024-12-15 10:30:00` | Atualizado em UPDATE |
| 🏷️ **eh_ativo** | BIT | ✓ | Região ativa | `1` | Default: 1 |

**Hierarquia Geográfica:**
```
pais → regiao_pais → estado → cidade
```

**Origem:** Base de dados IBGE + enriquecimento demográfico

**Unique Constraint:**
```sql
UNIQUE (pais, estado, cidade)
```

---

## DIM_EQUIPE - Dimensão Equipe

**Schema:** `dim.DIM_EQUIPE`  
**Registros Estimados:** 10 - 100  
**Crescimento:** Baixo (reorganizações ocasionais)  
**SCD Type:** Type 1

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **equipe_id** | INT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **equipe_original_id** | INT | ✓ | Natural Key (RH/CRM) | `501` | UNIQUE |
| 📝 **nome_equipe** | VARCHAR(100) | ✓ | Nome da equipe | `"Equipe Alpha SP"` | UNIQUE |
| 📝 **codigo_equipe** | VARCHAR(20) | ✗ | Código interno | `"EQ-SP-01"` | - |
| 📝 **tipo_equipe** | VARCHAR(30) | ✗ | Tipo de atuação | `"Vendas Diretas"` | `IN ('Vendas Diretas','Inside Sales','Key Accounts','Varejo','E-commerce')` |
| 📝 **categoria_equipe** | VARCHAR(30) | ✗ | Classificação performance | `"Elite"` | `IN ('Elite','Avançado','Intermediário','Iniciante')` |
| 📝 **regional** | VARCHAR(50) | ✗ | Região de atuação | `"Sudeste"` | - |
| 📝 **estado_sede** | CHAR(2) | ✗ | UF da sede | `"SP"` | `LEN = 2` |
| 📝 **cidade_sede** | VARCHAR(100) | ✗ | Cidade da sede | `"São Paulo"` | - |
| 🔗 **lider_equipe_id** | INT | ✗ | FK → DIM_VENDEDOR | `1` | Circular reference |
| 📝 **nome_lider** | VARCHAR(150) | ✗ | Nome do líder (desnorm.) | `"Carlos Silva"` | Atualizado com ETL |
| 📝 **email_lider** | VARCHAR(255) | ✗ | Email do líder | `"carlos@empresa.com"` | - |
| 📊 **meta_mensal_equipe** | DECIMAL(15,2) | ✗ | Meta de vendas mensal | `500000.00` | `>= 0` |
| 📊 **meta_trimestral_equipe** | DECIMAL(15,2) | ✗ | Meta trimestral | `1500000.00` | Geralmente meta_mensal * 3 |
| 📊 **meta_anual_equipe** | DECIMAL(15,2) | ✗ | Meta anual | `6000000.00` | - |
| 📊 **qtd_meta_vendas_mes** | INT | ✗ | Meta de quantidade mensal | `150` | Número de transações |
| 📊 **qtd_membros_atual** | INT | ✗ | Vendedores atuais | `8` | Atualizado por ETL |
| 📊 **qtd_membros_ideal** | INT | ✗ | Tamanho ideal da equipe | `10` | Planejamento RH |
| 📊 **total_vendas_mes_anterior** | DECIMAL(15,2) | ✗ | Vendas do último mês | `520000.00` | Snapshot |
| 📊 **percentual_meta_mes_anterior** | DECIMAL(5,2) | ✗ | % meta atingida | `104.00` | Calculado |
| 📊 **ranking_ultimo_mes** | INT | ✗ | Posição no ranking | `2` | 1 = melhor equipe |
| 🗓️ **data_criacao** | DATE | ✓ | Data de formação | `2023-01-15` | - |
| 🗓️ **data_ultima_atualizacao** | DATETIME | ✓ | Última modificação | `2024-12-15 10:00:00` | Default: GETDATE() |
| 🗓️ **data_inativacao** | DATE | ✗ | Data de desativação | `NULL` | NULL se ativa |
| 📝 **situacao** | VARCHAR(20) | ✓ | Status da equipe | `"Ativa"` | `IN ('Ativa','Inativa','Suspensa','Em Formação')` |
| 🏷️ **eh_ativa** | BIT | ✓ | Flag booleana | `1` | Default: 1 |
| 📝 **observacoes** | VARCHAR(500) | ✗ | Notas | `"Especializada em B2B"` | Texto livre |

**Origem:** Sistema RH + CRM

**Relacionamento Circular:**
- `DIM_EQUIPE.lider_equipe_id` → `DIM_VENDEDOR.vendedor_id`
- `DIM_VENDEDOR.equipe_id` → `DIM_EQUIPE.equipe_id`

**Solução:** Criar DIM_EQUIPE primeiro, popular DIM_VENDEDOR, depois atualizar líderes

---

## DIM_VENDEDOR - Dimensão Vendedor

**Schema:** `dim.DIM_VENDEDOR`  
**Registros Estimados:** 50 - 1.000  
**Crescimento:** Médio (contratações e desligamentos)  
**SCD Type:** Type 1

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **vendedor_id** | INT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **vendedor_original_id** | INT | ✓ | Natural Key (RH) | `10234` | UNIQUE |
| 📝 **nome_vendedor** | VARCHAR(150) | ✓ | Nome completo | `"João da Silva"` | - |
| 📝 **nome_exibicao** | VARCHAR(50) | ✗ | Nome curto | `"João S."` | Para dashboards |
| 📝 **matricula** | VARCHAR(20) | ✗ | Matrícula funcional | `"VND2024001"` | UNIQUE |
| 📝 **cpf** | VARCHAR(14) | ✗ | CPF do vendedor | `"123.456.789-00"` | UNIQUE, formato com pontuação |
| 📝 **email** | VARCHAR(255) | ✓ | Email corporativo | `"joao.silva@empresa.com"` | UNIQUE |
| 📝 **email_pessoal** | VARCHAR(255) | ✗ | Email pessoal | `"joao@gmail.com"` | Backup |
| 📝 **telefone_celular** | VARCHAR(20) | ✗ | Telefone móvel | `"(11) 99999-9999"` | - |
| 📝 **telefone_comercial** | VARCHAR(20) | ✗ | Ramal | `"(11) 3333-4444 R:123"` | - |
| 📝 **cargo** | VARCHAR(50) | ✓ | Cargo atual | `"Vendedor Pleno"` | - |
| 📝 **nivel_senioridade** | VARCHAR(20) | ✗ | Nível | `"Pleno"` | `IN ('Júnior','Pleno','Sênior','Especialista','Gerente')` |
| 📝 **departamento** | VARCHAR(50) | ✗ | Departamento | `"Vendas"` | - |
| 📝 **area** | VARCHAR(50) | ✗ | Área específica | `"B2B"` | - |
| 🔗 **equipe_id** | INT | ✗ | FK → DIM_EQUIPE | `1` | NULL = sem equipe |
| 📝 **nome_equipe** | VARCHAR(100) | ✗ | Nome da equipe (desnorm.) | `"Equipe Alpha SP"` | - |
| 🔗 **gerente_id** | INT | ✗ | FK → DIM_VENDEDOR (self) | `5` | NULL = sem gerente |
| 📝 **nome_gerente** | VARCHAR(150) | ✗ | Nome do gerente (desnorm.) | `"Carlos Silva"` | - |
| 📝 **estado_atuacao** | CHAR(2) | ✗ | UF principal | `"SP"` | - |
| 📝 **cidade_atuacao** | VARCHAR(100) | ✗ | Cidade base | `"São Paulo"` | - |
| 📝 **territorio_vendas** | VARCHAR(100) | ✗ | Território | `"Grande SP"` | - |
| 📝 **tipo_vendedor** | VARCHAR(30) | ✗ | Tipo de atuação | `"Externo"` | `IN ('Interno','Externo','Híbrido','Remoto')` |
| 📊 **meta_mensal_base** | DECIMAL(15,2) | ✗ | Meta padrão mensal | `50000.00` | Base para FACT_METAS |
| 📊 **meta_trimestral_base** | DECIMAL(15,2) | ✗ | Meta trimestral | `150000.00` | - |
| 📊 **percentual_comissao_padrao** | DECIMAL(5,2) | ✗ | % comissão | `3.50` | `BETWEEN 0 AND 100` |
| 📝 **tipo_comissao** | VARCHAR(30) | ✗ | Tipo | `"Variável"` | `IN ('Fixa','Variável','Escalonada')` |
| 📊 **total_vendas_mes_atual** | DECIMAL(15,2) | ✗ | Vendas do mês corrente | `45000.00` | Snapshot, atualizado |
| 📊 **total_vendas_mes_anterior** | DECIMAL(15,2) | ✗ | Vendas do mês passado | `52000.00` | Snapshot |
| 📊 **percentual_meta_mes_anterior** | DECIMAL(5,2) | ✗ | % meta atingida | `104.00` | - |
| 📊 **ranking_mes_anterior** | INT | ✗ | Posição no ranking | `3` | 1 = melhor |
| 📊 **total_vendas_acumulado_ano** | DECIMAL(15,2) | ✗ | Total no ano | `600000.00` | Year-to-date |
| 🗓️ **data_contratacao** | DATE | ✓ | Data de admissão | `2023-01-15` | - |
| 🗓️ **data_primeira_venda** | DATE | ✗ | Primeira transação | `2023-02-01` | Marco |
| 🗓️ **data_ultima_venda** | DATE | ✗ | Última transação | `2024-12-14` | Atualizado |
| 🗓️ **data_desligamento** | DATE | ✗ | Data de saída | `NULL` | NULL = ativo |
| 🗓️ **data_ultima_atualizacao** | DATETIME | ✓ | Última modificação | `2024-12-15 09:00:00` | - |
| 📝 **situacao** | VARCHAR(20) | ✓ | Status | `"Ativo"` | `IN ('Ativo','Afastado','Suspenso','Desligado')` |
| 🏷️ **eh_ativo** | BIT | ✓ | Flag booleana | `1` | Default: 1 |
| 🏷️ **eh_lider** | BIT | ✓ | É líder de equipe? | `0` | 0=Não, 1=Sim |
| 🏷️ **aceita_novos_clientes** | BIT | ✓ | Aceita leads? | `1` | Controle de distribuição |
| 📝 **observacoes** | VARCHAR(500) | ✗ | Notas | `"Especialista B2B"` | - |
| 📝 **motivo_desligamento** | VARCHAR(200) | ✗ | Motivo | `"Pedido de demissão"` | Se desligado |

**Origem:** Sistema RH (ADP/Workday)

**Self-Join Hierarchy:**
```sql
-- Exemplo de hierarquia
vendedor.gerente_id → vendedor.vendedor_id
```

---

## DIM_DESCONTO - Dimensão Desconto

**Schema:** `dim.DIM_DESCONTO`  
**Registros Estimados:** 100 - 1.000  
**Crescimento:** Médio (novas campanhas)  
**SCD Type:** Type 1

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **desconto_id** | INT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **desconto_original_id** | INT | ✓ | Natural Key (Marketing) | `7890` | UNIQUE |
| 📝 **codigo_desconto** | VARCHAR(50) | ✓ | Código do cupom | `"BLACKFRIDAY"` | UNIQUE |
| 📝 **nome_campanha** | VARCHAR(100) | ✓ | Nome da campanha | `"Black Friday 2024"` | - |
| 📝 **tipo_desconto** | VARCHAR(30) | ✓ | Tipo de desconto | `"Percentual"` | `IN ('Percentual','Valor Fixo','Frete Grátis','Brinde')` |
| 📝 **metodo_desconto** | VARCHAR(30) | ✓ | Método de aplicação | `"Cupom"` | `IN ('Cupom','Automático','Negociado','Volume')` |
| 📊 **valor_desconto** | DECIMAL(10,2) | ✓ | Valor (R$ ou %) | `10.00` | `> 0`, interpretação depende do tipo |
| 📊 **min_valor_compra_regra** | DECIMAL(10,2) | ✗ | Valor mínimo para aplicar | `100.00` | NULL = sem mínimo |
| 📊 **max_valor_desconto_regra** | DECIMAL(10,2) | ✗ | Teto do desconto | `50.00` | NULL = sem teto |
| 📝 **aplica_em** | VARCHAR(30) | ✓ | Nível de aplicação | `"Carrinho"` | `IN ('Produto','Categoria','Carrinho','Frete')` |
| 🗓️ **data_inicio_validade** | DATE | ✓ | Início da vigência | `2024-11-25` | - |
| 🗓️ **data_fim_validade** | DATE | ✗ | Fim da vigência | `2024-11-30` | NULL = sem expiração |
| 📝 **situacao** | VARCHAR(20) | ✓ | Status | `"Ativo"` | `IN ('Ativo','Inativo','Expirado','Pausado')` |

**Origem:** Sistema de Marketing/Promoções

**Vigência:**
```sql
-- Cupom está vigente se:
GETDATE() BETWEEN data_inicio_validade AND ISNULL(data_fim_validade, '9999-12-31')
AND situacao = 'Ativo'
```

---

## 📊 TABELAS FATO

## FACT_VENDAS - Fato Transacional

**Schema:** `fact.FACT_VENDAS`  
**Registros Estimados:** Milhões (cresce continuamente)  
**Crescimento:** Alto (centenas/milhares por dia)  
**Tipo:** Transaction Fact Table

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **venda_id** | BIGINT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **data_id** | INT | ✓ | FK → DIM_DATA | `20241215` | NOT NULL |
| 🔗 **cliente_id** | INT | ✓ | FK → DIM_CLIENTE | `5` | NOT NULL |
| 🔗 **produto_id** | INT | ✓ | FK → DIM_PRODUTO | `10` | NOT NULL |
| 🔗 **regiao_id** | INT | ✓ | FK → DIM_REGIAO | `1` | NOT NULL |
| 🔗 **vendedor_id** | INT | ✗ | FK → DIM_VENDEDOR | `3` | NULL = venda direta |
| 📊 **quantidade_vendida** | INT | ✓ | Unidades vendidas | `2` | `> 0` |
| 📊 **preco_unitario_tabela** | DECIMAL(10,2) | ✓ | Preço de tabela | `3500.00` | `> 0` |
| 📊 **valor_total_bruto** | DECIMAL(15,2) | ✓ | Valor antes de descontos | `7000.00` | `>= 0` |
| 📊 **valor_total_descontos** | DECIMAL(15,2) | ✓ | Total de descontos | `700.00` | `>= 0` |
| 📊 **valor_total_liquido** | DECIMAL(15,2) | ✓ | Valor pago pelo cliente | `6300.00` | `>= 0` |
| 📊 **custo_total** | DECIMAL(15,2) | ✓ | Custo dos produtos | `4000.00` | `>= 0` |
| 📊 **quantidade_devolvida** | INT | ✓ | Unidades devolvidas | `0` | `>= 0`, `<= quantidade_vendida` |
| 📊 **valor_devolvido** | DECIMAL(15,2) | ✓ | Valor reembolsado | `0.00` | `>= 0` |
| 📊 **percentual_comissao** | DECIMAL(5,2) | ✗ | % comissão vendedor | `3.50` | `BETWEEN 0 AND 100` |
| 📊 **valor_comissao** | DECIMAL(15,2) | ✗ | Valor da comissão | `220.50` | `>= 0` |
| 📝 **numero_pedido** | VARCHAR(20) | ✓ | Número do pedido (DD) | `"PED-2024-123456"` | Degenerate Dimension |
| 🏷️ **teve_desconto** | BIT | ✓ | Flag de desconto | `1` | 0=Não, 1=Sim |
| 🗓️ **data_inclusao** | DATETIME | ✓ | Quando foi inserido | `2024-12-15 10:30:00` | Default: GETDATE() |
| 🗓️ **data_atualizacao** | DATETIME | ✓ | Última atualização | `2024-12-15 10:30:00` | Default: GETDATE() |

**Granularidade:** 1 item vendido em 1 pedido

**Constraints Críticos:**
```sql
-- Valor líquido = bruto - descontos
CHECK (valor_total_liquido = valor_total_bruto - valor_total_descontos)

-- Quantidade devolvida <= vendida
CHECK (quantidade_devolvida <= quantidade_vendida)
```

**Métricas Calculadas (em queries):**
```sql
-- Margem
(valor_total_liquido - custo_total) AS lucro_bruto
(valor_total_liquido - custo_total) / valor_total_liquido * 100 AS margem_percentual

-- Ticket médio
AVG(valor_total_liquido) AS ticket_medio
```

---

## FACT_METAS - Snapshot Periódico

**Schema:** `fact.FACT_METAS`  
**Registros Estimados:** Milhares (controlado)  
**Crescimento:** Baixo (número vendedores × períodos)  
**Tipo:** Periodic Snapshot Fact Table

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **meta_id** | BIGINT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **vendedor_id** | INT | ✓ | FK → DIM_VENDEDOR | `3` | NOT NULL |
| 🔗 **data_id** | INT | ✓ | FK → DIM_DATA | `20241201` | NOT NULL (1º dia do mês) |
| 📊 **valor_meta** | DECIMAL(15,2) | ✓ | Meta em R$ | `50000.00` | `> 0` |
| 📊 **quantidade_meta** | INT | ✗ | Meta em quantidade | `20` | `> 0` |
| 📊 **valor_realizado** | DECIMAL(15,2) | ✓ | Vendas reais | `52500.00` | `>= 0` |
| 📊 **quantidade_realizada** | INT | ✓ | Vendas reais (qtd) | `22` | `>= 0` |
| 📊 **percentual_atingido** | DECIMAL(5,2) | ✓ | % da meta | `105.00` | `>= 0` |
| 📊 **gap_meta** | DECIMAL(15,2) | ✓ | Diferença | `2500.00` | Pode ser negativo |
| 📊 **ticket_medio_realizado** | DECIMAL(10,2) | ✗ | Ticket médio | `2386.36` | Calculado |
| 📊 **ranking_periodo** | INT | ✗ | Posição no ranking | `3` | 1 = melhor |
| 📝 **quartil_performance** | VARCHAR(10) | ✗ | Quartil | `"Q1"` | `IN ('Q1','Q2','Q3','Q4')` |
| 🏷️ **meta_batida** | BIT | ✓ | Atingiu meta? | `1` | 0=Não, 1=Sim |
| 🏷️ **meta_superada** | BIT | ✓ | Superou meta? | `1` | 0=Não, 1=Sim (>100%) |
| 🏷️ **eh_periodo_fechado** | BIT | ✓ | Período encerrado? | `1` | 0=Em andamento, 1=Fechado |
| 📝 **tipo_periodo** | VARCHAR(20) | ✓ | Tipo | `"Mensal"` | `IN ('Mensal','Trimestral','Anual')` |
| 📝 **observacoes** | VARCHAR(500) | ✗ | Notas | `"Meta ajustada devido férias"` | - |
| 🗓️ **data_inclusao** | DATETIME | ✓ | Quando criado | `2024-12-01 00:00:00` | Default: GETDATE() |
| 🗓️ **data_ultima_atualizacao** | DATETIME | ✓ | Última atualização | `2024-12-31 23:59:59` | Atualizado no ETL |

**Granularidade:** 1 meta de 1 vendedor em 1 período

**Unique Constraint:**
```sql
UNIQUE (vendedor_id, data_id, tipo_periodo)
-- Garante: vendedor não pode ter 2 metas no mesmo período
```

**Constraint de Coerência:**
```sql
CHECK (
    (meta_batida = 0 AND percentual_atingido < 100) OR
    (meta_batida = 1 AND percentual_atingido >= 100)
)
```

---

## FACT_DESCONTOS - Fato Transacional

**Schema:** `fact.FACT_DESCONTOS`  
**Registros Estimados:** Variável (depende de campanhas)  
**Crescimento:** Médio (múltiplos descontos por venda)  
**Tipo:** Transaction Fact Table

### Campos

| Campo | Tipo | Obr. | Descrição | Exemplo | Regras |
|-------|------|------|-----------|---------|--------|
| 🔑 **desconto_aplicado_id** | BIGINT | ✓ | PK - Surrogate Key | `1` | PRIMARY KEY IDENTITY |
| 🔗 **desconto_id** | INT | ✓ | FK → DIM_DESCONTO | `10` | NOT NULL |
| 🔗 **venda_id** | BIGINT | ✓ | FK → FACT_VENDAS | `123` | NOT NULL |
| 🔗 **data_aplicacao_id** | INT | ✓ | FK → DIM_DATA | `20241215` | NOT NULL |
| 🔗 **cliente_id** | INT | ✓ | FK → DIM_CLIENTE | `5` | NOT NULL (desnorm.) |
| 🔗 **produto_id** | INT | ✗ | FK → DIM_PRODUTO | `10` | NULL se desconto no pedido |
| 📝 **nivel_aplicacao** | VARCHAR(20) | ✓ | Nível | `"Produto"` | `IN ('Produto','Pedido','Frete')` |
| 📊 **valor_desconto_aplicado** | DECIMAL(10,2) | ✓ | Valor do desconto | `350.00` | `>= 0` |
| 📊 **valor_sem_desconto** | DECIMAL(10,2) | ✓ | Valor original | `3500.00` | `>= 0` |
| 📊 **valor_com_desconto** | DECIMAL(10,2) | ✓ | Valor final | `3150.00` | `>= 0` |
| 📊 **margem_antes_desconto** | DECIMAL(10,2) | ✓ | Margem original | `1500.00` | Pode ser negativo |
| 📊 **margem_apos_desconto** | DECIMAL(10,2) | ✓ | Margem final | `1150.00` | Pode ser negativo |
| 📊 **impacto_margem** | DECIMAL(10,2) | ✓ | Redução | `-350.00` | Negativo = perda |
| 📝 **numero_pedido** | VARCHAR(20) | ✓ | Número do pedido (DD) | `"PED-2024-123456"` | Degenerate Dimension |
| 🏷️ **desconto_aprovado** | BIT | ✓ | Foi aprovado? | `1` | 0=Não, 1=Sim |
| 🗓️ **data_inclusao** | DATETIME | ✓ | Quando registrado | `2024-12-15 10:30:00` | Default: GETDATE() |

**Granularidade:** 1 desconto aplicado em 1 venda

**Relacionamento Fact-to-Fact:**
```sql
-- Um pedido pode ter múltiplos descontos
-- Exemplo: cupom + volume + frete grátis
```

**Constraints:**
```sql
-- Valor com desconto = sem desconto - desconto aplicado
CHECK (valor_com_desconto = valor_sem_desconto - valor_desconto_aplicado)
```

---

## 🔍 VIEWS AUXILIARES

### Views Dimensionais

| View | Descrição | Base |
|------|-----------|------|
| **VW_CALENDARIO_COMPLETO** | Calendário + campos calculados | DIM_DATA |
| **VW_PRODUTOS_ATIVOS** | Produtos ativos + margem | DIM_PRODUTO |
| **VW_HIERARQUIA_GEOGRAFICA** | Hierarquia geográfica | DIM_REGIAO |
| **VW_DESCONTOS_ATIVOS** | Descontos vigentes | DIM_DESCONTO |
| **VW_VENDEDORES_ATIVOS** | Vendedores + tempo casa | DIM_VENDEDOR |
| **VW_HIERARQUIA_VENDEDORES** | Hierarquia gerencial | DIM_VENDEDOR (self-join) |

### Views de Equipes

| View | Descrição | Base |
|------|-----------|------|
| **VW_ANALISE_EQUIPE_VENDEDORES** | Análise de composição | DIM_EQUIPE + DIM_VENDEDOR |
| **VW_EQUIPES_ATIVAS** | Equipes operacionais | DIM_EQUIPE |
| **VW_RANKING_EQUIPES_META** | Ranking por meta | DIM_EQUIPE |
| **VW_ANALISE_REGIONAL_EQUIPES** | Agregação regional | DIM_EQUIPE |

### Views Mestres

| View | Descrição | Base |
|------|-----------|------|
| **VW_VENDAS_COMPLETA** | Vendas + todas dimensões | FACT_VENDAS + JOINs |
| **VW_METAS_COMPLETA** | Metas + contexto completo | FACT_METAS + JOINs |

**Documentação completa:** Ver `sql/04_views/README.md`

---

## 📚 Glossário de Termos

### Termos de Modelagem Dimensional

| Termo | Definição |
|-------|-----------|
| **Star Schema** | Modelo com fact no centro e dimensions ao redor (estrela) |
| **Snowflake Schema** | Star schema com dimensões normalizadas |
| **Surrogate Key** | Chave artificial (1,2,3...) gerada pelo DW |
| **Natural Key** | Chave do sistema fonte (codigo_sku, cpf) |
| **Granularidade** | Nível de detalhe: o que é 1 linha da fact? |
| **SCD Type 1** | Sobrescreve: valor antigo perdido |
| **SCD Type 2** | Novo registro: histórico completo mantido |
| **Degenerate Dimension (DD)** | Atributo descritivo que fica na fact (numero_pedido) |
| **Conformed Dimension** | Dimensão compartilhada entre múltiplas facts |

### Tipos de Métricas

| Termo | Definição |
|-------|-----------|
| **Additive Measure** | Métrica somável em todas dimensões (quantidade) |
| **Semi-Additive** | Somável em algumas dimensões (saldo_conta) |
| **Non-Additive** | Não somável, deve ser calculada (percentual) |

### Operações Analíticas

| Termo | Definição |
|-------|-----------|
| **Drill-Down** | Detalhar: ano → trimestre → mês |
| **Roll-Up** | Agregar: dia → mês → ano |
| **Slice** | Filtrar uma dimensão: "apenas 2024" |
| **Dice** | Filtrar múltiplas dimensões: "2024 + SP + Eletrônicos" |

### Tipos de Facts

| Termo | Definição |
|-------|-----------|
| **Transaction Fact** | Cada linha = evento individual (FACT_VENDAS) |
| **Periodic Snapshot** | Foto periódica do estado (FACT_METAS) |
| **Accumulating Snapshot** | Processo com múltiplas etapas (não implementado) |

---

## 📊 Resumo Estatístico

### Contagem de Campos por Tabela

| Tabela | Total Campos | PKs | FKs | Métricas | Descritivos | Flags | Temporais |
|--------|--------------|-----|-----|----------|-------------|-------|-----------|
| DIM_DATA | 13 | 1 | 0 | 0 | 10 | 2 | 0 |
| DIM_CLIENTE | 12 | 1 | 1 | 0 | 7 | 1 | 2 |
| DIM_PRODUTO | 14 | 1 | 2 | 3 | 7 | 1 | 0 |
| DIM_REGIAO | 21 | 1 | 1 | 5 | 11 | 1 | 2 |
| DIM_EQUIPE | 22 | 1 | 1 | 9 | 8 | 1 | 3 |
| DIM_VENDEDOR | 38 | 1 | 3 | 7 | 19 | 3 | 5 |
| DIM_DESCONTO | 12 | 1 | 1 | 3 | 5 | 0 | 2 |
| FACT_VENDAS | 18 | 1 | 5 | 9 | 1 | 1 | 2 |
| FACT_METAS | 19 | 1 | 2 | 9 | 2 | 3 | 2 |
| FACT_DESCONTOS | 16 | 1 | 5 | 6 | 2 | 1 | 1 |
| **TOTAL** | **185** | **10** | **21** | **51** | **72** | **14** | **19** |

### Tipos de Dados Mais Usados

| Tipo | Frequência | Uso Principal |
|------|------------|---------------|
| VARCHAR | 42% | Textos descritivos |
| DECIMAL | 18% | Valores monetários e percentuais |
| INT | 15% | IDs e contadores |
| BIT | 8% | Flags booleanas |
| DATE/DATETIME | 10% | Campos temporais |
| BIGINT | 2% | PKs de facts |
| CHAR | 5% | Códigos fixos (UF, DDD) |

---

## 🔍 Índice Alfabético de Campos

<details>
<summary>Clique para expandir lista completa (185 campos)</summary>

| Campo | Tabelas |
|-------|---------|
| aceita_novos_clientes | DIM_VENDEDOR |
| ano | DIM_DATA |
| aplica_em | DIM_DESCONTO |
| area | DIM_VENDEDOR |
| area_km2 | DIM_REGIAO |
| cargo | DIM_VENDEDOR |
| categoria | DIM_PRODUTO |
| categoria_equipe | DIM_EQUIPE |
| cep_final | DIM_REGIAO |
| cep_inicial | DIM_REGIAO |
| cidade | DIM_CLIENTE, DIM_REGIAO |
| cidade_atuacao | DIM_VENDEDOR |
| cidade_sede | DIM_EQUIPE |
| cliente_id | DIM_CLIENTE (PK), FACT_VENDAS, FACT_DESCONTOS |
| cliente_original_id | DIM_CLIENTE |
| codigo_desconto | DIM_DESCONTO |
| codigo_equipe | DIM_EQUIPE |
| codigo_ibge | DIM_REGIAO |
| codigo_sku | DIM_PRODUTO |
| cpf | DIM_VENDEDOR |
| custo_medio | DIM_PRODUTO |
| custo_total | FACT_VENDAS |
| data_aplicacao_id | FACT_DESCONTOS |
| data_atualizacao | FACT_VENDAS |
| data_cadastro | DIM_CLIENTE, DIM_REGIAO |
| data_completa | DIM_DATA |
| data_contratacao | DIM_VENDEDOR |
| data_criacao | DIM_EQUIPE |
| data_desligamento | DIM_VENDEDOR |
| data_fim_validade | DIM_DESCONTO |
| data_id | DIM_DATA (PK), FACT_VENDAS, FACT_METAS |
| data_inativacao | DIM_EQUIPE |
| data_inclusao | FACT_VENDAS, FACT_METAS, FACT_DESCONTOS |
| data_inicio_validade | DIM_DESCONTO |
| data_primeira_venda | DIM_VENDEDOR |
| data_ultima_atualizacao | DIM_REGIAO, DIM_EQUIPE, DIM_VENDEDOR, FACT_METAS |
| data_ultima_compra | DIM_CLIENTE |
| data_ultima_venda | DIM_VENDEDOR |
| ddd | DIM_REGIAO |
| densidade_demografica | DIM_REGIAO |
| departamento | DIM_VENDEDOR |
| desconto_aprovado | FACT_DESCONTOS |
| desconto_aplicado_id | FACT_DESCONTOS (PK) |
| desconto_id | DIM_DESCONTO (PK), FACT_DESCONTOS |
| desconto_original_id | DIM_DESCONTO |
| dia_ano | DIM_DATA |
| dia_mes | DIM_DATA |
| dia_semana | DIM_DATA |
| dimensoes | DIM_PRODUTO |
| eh_ativo | DIM_CLIENTE, DIM_PRODUTO, DIM_REGIAO, DIM_EQUIPE, DIM_VENDEDOR |
| eh_ativa | DIM_EQUIPE |
| eh_feriado | DIM_DATA |
| eh_fim_de_semana | DIM_DATA |
| eh_lider | DIM_VENDEDOR |
| eh_periodo_fechado | FACT_METAS |
| email | DIM_CLIENTE, DIM_VENDEDOR |
| email_lider | DIM_EQUIPE |
| email_pessoal | DIM_VENDEDOR |
| equipe_id | DIM_EQUIPE (PK), DIM_VENDEDOR |
| equipe_original_id | DIM_EQUIPE |
| estado | DIM_CLIENTE, DIM_REGIAO |
| estado_atuacao | DIM_VENDEDOR |
| estado_sede | DIM_EQUIPE |
| fornecedor_id | DIM_PRODUTO |
| fuso_horario | DIM_REGIAO |
| gap_meta | FACT_METAS |
| gerente_id | DIM_VENDEDOR |
| idh | DIM_REGIAO |
| impacto_margem | FACT_DESCONTOS |
| latitude | DIM_REGIAO |
| lider_equipe_id | DIM_EQUIPE |
| longitude | DIM_REGIAO |
| marca | DIM_PRODUTO |
| margem_antes_desconto | FACT_DESCONTOS |
| margem_apos_desconto | FACT_DESCONTOS |
| matricula | DIM_VENDEDOR |
| max_valor_desconto_regra | DIM_DESCONTO |
| mes | DIM_DATA |
| meta_anual_equipe | DIM_EQUIPE |
| meta_batida | FACT_METAS |
| meta_id | FACT_METAS (PK) |
| meta_mensal_base | DIM_VENDEDOR |
| meta_mensal_equipe | DIM_EQUIPE |
| meta_superada | FACT_METAS |
| meta_trimestral_base | DIM_VENDEDOR |
| meta_trimestral_equipe | DIM_EQUIPE |
| metodo_desconto | DIM_DESCONTO |
| min_valor_compra_regra | DIM_DESCONTO |
| motivo_desligamento | DIM_VENDEDOR |
| nivel_aplicacao | FACT_DESCONTOS |
| nivel_senioridade | DIM_VENDEDOR |
| nome_campanha | DIM_DESCONTO |
| nome_cliente | DIM_CLIENTE |
| nome_dia_semana | DIM_DATA |
| nome_equipe | DIM_EQUIPE, DIM_VENDEDOR |
| nome_estado | DIM_REGIAO |
| nome_feriado | DIM_DATA |
| nome_fornecedor | DIM_PRODUTO |
| nome_gerente | DIM_VENDEDOR |
| nome_lider | DIM_EQUIPE |
| nome_mes | DIM_DATA |
| nome_produto | DIM_PRODUTO |
| nome_vendedor | DIM_VENDEDOR |
| numero_pedido | FACT_VENDAS, FACT_DESCONTOS |
| observacoes | DIM_EQUIPE, DIM_VENDEDOR, FACT_METAS |
| pais | DIM_CLIENTE, DIM_REGIAO |
| percentual_atingido | FACT_METAS |
| percentual_comissao | FACT_VENDAS |
| percentual_comissao_padrao | DIM_VENDEDOR |
| percentual_meta_mes_anterior | DIM_EQUIPE, DIM_VENDEDOR |
| peso_kg | DIM_PRODUTO |
| pib_per_capita | DIM_REGIAO |
| populacao_estimada | DIM_REGIAO |
| porte_municipio | DIM_REGIAO |
| preco_sugerido | DIM_PRODUTO |
| preco_unitario_tabela | FACT_VENDAS |
| produto_id | DIM_PRODUTO (PK), FACT_VENDAS, FACT_DESCONTOS |
| produto_original_id | DIM_PRODUTO |
| quantidade_devolvida | FACT_VENDAS |
| quantidade_meta | FACT_METAS |
| quantidade_realizada | FACT_METAS |
| quantidade_vendida | FACT_VENDAS |
| quartil_performance | FACT_METAS |
| qtd_membros_atual | DIM_EQUIPE |
| qtd_membros_ideal | DIM_EQUIPE |
| qtd_meta_vendas_mes | DIM_EQUIPE |
| ranking_mes_anterior | DIM_VENDEDOR |
| ranking_periodo | FACT_METAS |
| ranking_ultimo_mes | DIM_EQUIPE |
| regiao_id | DIM_REGIAO (PK), FACT_VENDAS |
| regiao_original_id | DIM_REGIAO |
| regiao_pais | DIM_REGIAO |
| regional | DIM_EQUIPE |
| segmento | DIM_CLIENTE |
| situacao | DIM_EQUIPE, DIM_VENDEDOR, DIM_DESCONTO |
| subcategoria | DIM_PRODUTO |
| telefone_celular | DIM_VENDEDOR |
| telefone_comercial | DIM_VENDEDOR |
| territorio_vendas | DIM_VENDEDOR |
| ticket_medio_realizado | FACT_METAS |
| tipo_cliente | DIM_CLIENTE |
| tipo_comissao | DIM_VENDEDOR |
| tipo_desconto | DIM_DESCONTO |
| tipo_equipe | DIM_EQUIPE |
| tipo_municipio | DIM_REGIAO |
| tipo_periodo | FACT_METAS |
| tipo_vendedor | DIM_VENDEDOR |
| total_vendas_acumulado_ano | DIM_VENDEDOR |
| total_vendas_mes_anterior | DIM_EQUIPE, DIM_VENDEDOR |
| total_vendas_mes_atual | DIM_VENDEDOR |
| teve_desconto | FACT_VENDAS |
| trimestre | DIM_DATA |
| valor_comissao | FACT_VENDAS |
| valor_com_desconto | FACT_DESCONTOS |
| valor_desconto | DIM_DESCONTO |
| valor_desconto_aplicado | FACT_DESCONTOS |
| valor_devolvido | FACT_VENDAS