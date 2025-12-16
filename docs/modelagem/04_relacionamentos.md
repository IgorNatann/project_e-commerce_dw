# 🔗 Relacionamentos e Integridade Referencial

> Mapa completo de Foreign Keys e relacionamentos do modelo

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Relacionamentos por Tabela Fato](#relacionamentos-por-tabela-fato)
- [Relacionamentos entre Dimensões](#relacionamentos-entre-dimensões)
- [Relacionamentos Especiais](#relacionamentos-especiais)
- [Integridade Referencial](#integridade-referencial)
- [Diagrama Completo](#diagrama-completo)

---

## 🎯 Visão Geral

### Tipos de Relacionamentos no Modelo

```
┌─────────────────────────────────────────────────────────────────┐
│ TIPOS DE RELACIONAMENTOS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. FACT → DIMENSION (N:1)                                       │
│    └─ Padrão Star Schema                                        │
│    └─ Exemplo: FACT_VENDAS.cliente_id → DIM_CLIENTE.cliente_id │
│                                                                 │
│ 2. DIMENSION → DIMENSION (N:1)                                  │
│    └─ Relacionamento transitivo                                 │
│    └─ Exemplo: DIM_VENDEDOR.equipe_id → DIM_EQUIPE.equipe_id   │
│                                                                 │
│ 3. DIMENSION → DIMENSION (Self-Join)                            │
│    └─ Hierarquia dentro da mesma tabela                         │
│    └─ Exemplo: DIM_VENDEDOR.gerente_id → DIM_VENDEDOR.vend_id  │
│                                                                 │
│ 4. FACT → FACT (1:N)                                            │
│    └─ Relacionamento entre facts                                │
│    └─ Exemplo: FACT_VENDAS.venda_id ← FACT_DESCONTOS.venda_id  │
└─────────────────────────────────────────────────────────────────┘
```

### Estatísticas do Modelo

| Métrica | Quantidade |
|---------|------------|
| **Total de FKs** | 15 |
| **FKs em FACT_VENDAS** | 5 |
| **FKs em FACT_METAS** | 2 |
| **FKs em FACT_DESCONTOS** | 6 |
| **FKs entre Dimensões** | 2 |
| **Self-Joins** | 1 |

---

## 📊 Relacionamentos por Tabela Fato

### FACT_VENDAS → Dimensões

```
FACT_VENDAS (1 item vendido)
│
├─► DIM_DATA (data_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: QUANDO foi vendido?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Exemplo: venda do dia 2024-12-15
│
├─► DIM_CLIENTE (cliente_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: QUEM comprou?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Exemplo: Cliente "João Silva"
│
├─► DIM_PRODUTO (produto_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: O QUE foi comprado?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Exemplo: Produto "Notebook Dell"
│
├─► DIM_REGIAO (regiao_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: ONDE foi entregue?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Exemplo: Região "São Paulo/SP"
│
└─► DIM_VENDEDOR (vendedor_id)
    └─ Cardinalidade: N:1
    └─ Descrição: QUEM vendeu?
    └─ Obrigatório: NÃO (NULL permitido)
    └─ NULL quando: Venda direta (e-commerce sem vendedor)
    └─ Exemplo: Vendedor "Carlos Silva"
```

**SQL das FKs:**

```sql
ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT FK_FACT_VENDAS_data 
    FOREIGN KEY (data_id) REFERENCES dim.DIM_DATA(data_id);

ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT FK_FACT_VENDAS_cliente 
    FOREIGN KEY (cliente_id) REFERENCES dim.DIM_CLIENTE(cliente_id);

ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT FK_FACT_VENDAS_produto 
    FOREIGN KEY (produto_id) REFERENCES dim.DIM_PRODUTO(produto_id);

ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT FK_FACT_VENDAS_regiao 
    FOREIGN KEY (regiao_id) REFERENCES dim.DIM_REGIAO(regiao_id);

ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT FK_FACT_VENDAS_vendedor 
    FOREIGN KEY (vendedor_id) REFERENCES dim.DIM_VENDEDOR(vendedor_id);
```

---

### FACT_METAS → Dimensões

```
FACT_METAS (1 meta por vendedor por período)
│
├─► DIM_VENDEDOR (vendedor_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: Meta de QUAL vendedor?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Exemplo: Vendedor "João da Silva"
│
└─► DIM_DATA (data_id)
    └─ Cardinalidade: N:1
    └─ Descrição: Meta de QUAL período?
    └─ Obrigatório: SIM (NOT NULL)
    └─ Observação: Sempre 1º dia do mês/trimestre
    └─ Exemplo: 2024-12-01 (meta de dezembro)
```

**SQL das FKs:**

```sql
ALTER TABLE fact.FACT_METAS
ADD CONSTRAINT FK_FACT_METAS_vendedor 
    FOREIGN KEY (vendedor_id) REFERENCES dim.DIM_VENDEDOR(vendedor_id);

ALTER TABLE fact.FACT_METAS
ADD CONSTRAINT FK_FACT_METAS_data 
    FOREIGN KEY (data_id) REFERENCES dim.DIM_DATA(data_id);
```

**Constraint Único:**

```sql
ALTER TABLE fact.FACT_METAS
ADD CONSTRAINT UK_FACT_METAS_vendedor_periodo 
    UNIQUE (vendedor_id, data_id, tipo_periodo);
-- Garante: 1 vendedor não pode ter 2 metas no mesmo período
```

---

### FACT_DESCONTOS → Dimensões e Facts

```
FACT_DESCONTOS (1 desconto aplicado)
│
├─► DIM_DESCONTO (desconto_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: QUAL cupom/campanha?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Exemplo: Cupom "BLACKFRIDAY"
│
├─► FACT_VENDAS (venda_id) ⚠️ FK para outra FACT!
│   └─ Cardinalidade: N:1
│   └─ Descrição: Desconto aplicado em QUAL venda?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Relacionamento: 1 venda pode ter N descontos
│   └─ Exemplo: Venda #12345
│
├─► DIM_DATA (data_aplicacao_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: QUANDO foi aplicado?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Pode diferir da data da venda
│
├─► DIM_CLIENTE (cliente_id)
│   └─ Cardinalidade: N:1
│   └─ Descrição: Desconto para QUAL cliente?
│   └─ Obrigatório: SIM (NOT NULL)
│   └─ Desnormalizado da FACT_VENDAS para performance
│
└─► DIM_PRODUTO (produto_id)
    └─ Cardinalidade: N:1
    └─ Descrição: Desconto em QUAL produto?
    └─ Obrigatório: SIM (NOT NULL)
    └─ Pode ser NULL se desconto for no pedido/frete
```

**SQL das FKs:**

```sql
ALTER TABLE fact.FACT_DESCONTOS
ADD CONSTRAINT FK_FACT_DESCONTOS_desconto 
    FOREIGN KEY (desconto_id) REFERENCES dim.DIM_DESCONTO(desconto_id);

-- ⚠️ FK PARA OUTRA FACT
ALTER TABLE fact.FACT_DESCONTOS
ADD CONSTRAINT FK_FACT_DESCONTOS_venda 
    FOREIGN KEY (venda_id) REFERENCES fact.FACT_VENDAS(venda_id);

ALTER TABLE fact.FACT_DESCONTOS
ADD CONSTRAINT FK_FACT_DESCONTOS_data 
    FOREIGN KEY (data_aplicacao_id) REFERENCES dim.DIM_DATA(data_id);

ALTER TABLE fact.FACT_DESCONTOS
ADD CONSTRAINT FK_FACT_DESCONTOS_cliente 
    FOREIGN KEY (cliente_id) REFERENCES dim.DIM_CLIENTE(cliente_id);

ALTER TABLE fact.FACT_DESCONTOS
ADD CONSTRAINT FK_FACT_DESCONTOS_produto 
    FOREIGN KEY (produto_id) REFERENCES dim.DIM_PRODUTO(produto_id);
```

---

## 📐 Relacionamentos entre Dimensões

### DIM_VENDEDOR → DIM_EQUIPE (Transitivo)

```
DIM_VENDEDOR
│
└─► DIM_EQUIPE (equipe_id)
    └─ Cardinalidade: N:1
    └─ Descrição: Vendedor pertence a QUAL equipe?
    └─ Obrigatório: NÃO (NULL permitido)
    └─ NULL quando: Vendedor sem equipe (novo/transição)
    └─ Exemplo: Vendedor "João" → Equipe "Alpha SP"
```

**Por que esse relacionamento?**

✅ **Permite análise transitiva:**

```sql
-- Vendas por equipe (sem FK redundante na FACT)
SELECT 
    e.nome_equipe,
    SUM(fv.valor_total_liquido) AS receita
FROM fact.FACT_VENDAS fv
JOIN dim.DIM_VENDEDOR v ON fv.vendedor_id = v.vendedor_id
JOIN dim.DIM_EQUIPE e ON v.equipe_id = e.equipe_id
GROUP BY e.nome_equipe;
```

❌ **Alternativa descartada:**

```sql
-- FK redundante na FACT_VENDAS
ALTER TABLE fact.FACT_VENDAS
ADD equipe_id INT; -- ❌ DESNECESSÁRIO!

-- Por quê? Adiciona complexidade sem benefício real
-- - Queries funcionam perfeitamente com JOIN transitivo
-- - Aumenta tamanho da fact
-- - Dificulta manutenção (e se vendedor trocar de equipe?)
```

**SQL da FK:**

```sql
ALTER TABLE dim.DIM_VENDEDOR
ADD CONSTRAINT FK_DIM_VENDEDOR_equipe 
    FOREIGN KEY (equipe_id) REFERENCES dim.DIM_EQUIPE(equipe_id);
```

---

### DIM_EQUIPE → DIM_VENDEDOR (Circular)

```
DIM_EQUIPE
│
└─► DIM_VENDEDOR (lider_equipe_id)
    └─ Cardinalidade: 1:1 (ou N:1 se líder liderar múltiplas equipes)
    └─ Descrição: QUEM é o líder desta equipe?
    └─ Obrigatório: NÃO (NULL permitido inicialmente)
    └─ NULL quando: Equipe sem líder atribuído
    └─ Exemplo: Equipe "Alpha SP" → Líder "Carlos Silva"
```

**⚠️ Problema: Dependência Circular**

```
DIM_EQUIPE precisa de DIM_VENDEDOR existir (lider_equipe_id)
    ↓
DIM_VENDEDOR precisa de DIM_EQUIPE existir (equipe_id)
    ↓
💥 DEADLOCK DE CRIAÇÃO!
```

**✅ Solução: Criação em 2 Etapas**

```sql
-- ETAPA 1: Criar DIM_EQUIPE SEM FK para líder
CREATE TABLE dim.DIM_EQUIPE (
    equipe_id INT PRIMARY KEY,
    lider_equipe_id INT NULL,  -- SEM FK ainda!
    ...
);

-- ETAPA 2: Criar DIM_VENDEDOR com FK para equipe
CREATE TABLE dim.DIM_VENDEDOR (
    vendedor_id INT PRIMARY KEY,
    equipe_id INT,
    CONSTRAINT FK_VENDEDOR_equipe 
        FOREIGN KEY (equipe_id) REFERENCES dim.DIM_EQUIPE(equipe_id)
);

-- ETAPA 3: Popular vendedores
INSERT INTO dim.DIM_VENDEDOR (...) VALUES (...);

-- ETAPA 4: Adicionar FK de líder na equipe
ALTER TABLE dim.DIM_EQUIPE
ADD CONSTRAINT FK_EQUIPE_lider 
    FOREIGN KEY (lider_equipe_id) REFERENCES dim.DIM_VENDEDOR(vendedor_id);

-- ETAPA 5: Atualizar líderes
UPDATE dim.DIM_EQUIPE 
SET lider_equipe_id = 1 
WHERE equipe_id = 1;
```

---

## 🔄 Relacionamentos Especiais

### Self-Join: DIM_VENDEDOR → DIM_VENDEDOR

```
DIM_VENDEDOR (Hierarquia Gerencial)
│
└─► DIM_VENDEDOR (gerente_id)
    └─ Cardinalidade: N:1
    └─ Descrição: QUEM é o gerente deste vendedor?
    └─ Obrigatório: NÃO (NULL permitido)
    └─ NULL quando: Vendedor é o topo da hierarquia (CEO/Diretor)
    └─ Exemplo: Vendedor "João" → Gerente "Carlos"
```

**Exemplo de Hierarquia:**

```
CEO (gerente_id = NULL)
 └─ Diretor Comercial (gerente_id = CEO_id)
     └─ Gerente Regional (gerente_id = Diretor_id)
         └─ Coordenador (gerente_id = Gerente_id)
             └─ Vendedor Sênior (gerente_id = Coordenador_id)
                 └─ Vendedor Júnior (gerente_id = Sênior_id)
```

**SQL da FK:**

```sql
ALTER TABLE dim.DIM_VENDEDOR
ADD CONSTRAINT FK_DIM_VENDEDOR_gerente 
    FOREIGN KEY (gerente_id) REFERENCES dim.DIM_VENDEDOR(vendedor_id);
```

**Query de Exemplo:**

```sql
-- Hierarquia completa (3 níveis)
SELECT 
    v1.nome_vendedor AS vendedor,
    v2.nome_vendedor AS gerente,
    v3.nome_vendedor AS gerente_do_gerente
FROM dim.DIM_VENDEDOR v1
LEFT JOIN dim.DIM_VENDEDOR v2 ON v1.gerente_id = v2.vendedor_id
LEFT JOIN dim.DIM_VENDEDOR v3 ON v2.gerente_id = v3.vendedor_id
WHERE v1.eh_ativo = 1;
```

---

### FACT → FACT: FACT_DESCONTOS → FACT_VENDAS

```
FACT_VENDAS ──1:N──► FACT_DESCONTOS
    ▲
    │
    └─ 1 venda pode ter N descontos aplicados
    
Exemplo:
Venda #12345
├─ Desconto 1: BLACKFRIDAY (-10%)
├─ Desconto 2: VOLUME (-5%)
└─ Desconto 3: FRETE_GRATIS (-30 reais)
```

**Por que esse relacionamento?**

- ✅ Permite múltiplos descontos por venda
- ✅ Mantém flexibilidade (cenários futuros)
- ✅ Análise detalhada de cada desconto

**Query de Exemplo:**

```sql
-- Vendas com múltiplos descontos
SELECT 
    fv.numero_pedido,
    fv.valor_total_liquido,
    COUNT(fd.desconto_aplicado_id) AS qtd_descontos,
    STRING_AGG(d.codigo_desconto, ', ') AS cupons_usados
FROM fact.FACT_VENDAS fv
LEFT JOIN fact.FACT_DESCONTOS fd ON fv.venda_id = fd.venda_id
LEFT JOIN dim.DIM_DESCONTO d ON fd.desconto_id = d.desconto_id
GROUP BY fv.numero_pedido, fv.valor_total_liquido
HAVING COUNT(fd.desconto_aplicado_id) > 1;
```

---

## 🛡️ Integridade Referencial

### Regras de Negócio Aplicadas

```sql
-- 1. Não permitir órfãos em FACT_VENDAS
-- ✅ Garantido por FKs com ON DELETE RESTRICT (padrão)

-- 2. Quantidade vendida sempre positiva
ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT CK_FACT_VENDAS_quantidade_positiva 
    CHECK (quantidade_vendida > 0);

-- 3. Valor líquido = bruto - descontos
ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT CK_FACT_VENDAS_valor_liquido_coerente 
    CHECK (valor_total_liquido = valor_total_bruto - valor_total_descontos);

-- 4. Meta sempre positiva
ALTER TABLE fact.FACT_METAS
ADD CONSTRAINT CK_FACT_METAS_valor_meta_positivo 
    CHECK (valor_meta > 0);

-- 5. Meta batida coerente com percentual
ALTER TABLE fact.FACT_METAS
ADD CONSTRAINT CK_FACT_METAS_meta_batida_coerente 
    CHECK (
        (meta_batida = 0 AND percentual_atingido < 100) OR
        (meta_batida = 1 AND percentual_atingido >= 100)
    );

-- 6. Única meta por vendedor por período
ALTER TABLE fact.FACT_METAS
ADD CONSTRAINT UK_FACT_METAS_vendedor_periodo 
    UNIQUE (vendedor_id, data_id, tipo_periodo);
```

### Comportamento em Cascata

| Operação | Comportamento | Justificativa |
|----------|---------------|---------------|
| **DELETE Dimensão** | RESTRICT (erro) | Não permitir deletar dimensão com facts dependentes |
| **DELETE FACT_VENDAS** | CASCADE em FACT_DESCONTOS | Se venda é deletada, seus descontos também devem ser |
| **UPDATE PK Dimensão** | RESTRICT (erro) | PKs surrogate nunca mudam |

```sql
-- Exemplo: FK com DELETE RESTRICT (padrão)
ALTER TABLE fact.FACT_VENDAS
ADD CONSTRAINT FK_FACT_VENDAS_cliente 
    FOREIGN KEY (cliente_id) 
    REFERENCES dim.DIM_CLIENTE(cliente_id)
    ON DELETE RESTRICT;  -- ❌ Não permite deletar cliente com vendas

-- Exemplo: FK com DELETE CASCADE
ALTER TABLE fact.FACT_DESCONTOS
ADD CONSTRAINT FK_FACT_DESCONTOS_venda 
    FOREIGN KEY (venda_id) 
    REFERENCES fact.FACT_VENDAS(venda_id)
    ON DELETE CASCADE;  -- ✅ Deleta descontos ao deletar venda
```

---

## 📊 Diagrama Completo

```
                    DIM_DATA
                    ┌─────────────┐
                    │ data_id (PK)│
                    └──────┬──────┘
                           │
                           │ (FK)
                           │
    DIM_EQUIPE        DIM_VENDEDOR        DIM_DESCONTO
    ┌───────────┐     ┌───────────┐       ┌───────────┐
    │equipe_id  │◄────┤vendedor_id│       │desconto_id│
    │(PK)       │(FK) │(PK)       │       │(PK)       │
    │           │     │equipe_id  │       └─────┬─────┘
    │lider_eq_id│────►│gerente_id │◄┐          │
    └───────────┘     └─────┬─────┘ │(self-FK) │
                            │       │          │
                            │(FK)   └──────────┘
                            │
         ┌──────────────────┴─────────────────────────────┐
         │                                                 │
         ▼                                                 ▼
    ┌────────────────────────────┐        ┌───────────────────────────┐
    │    FACT_VENDAS             │        │   FACT_DESCONTOS          │
    │    ┌────────────────┐      │        │   ┌────────────────┐     │
    │    │venda_id (PK)   │◄─────┼────────┼───┤venda_id (FK)   │     │
    │    │data_id (FK)────┼──┐   │        │   │desconto_id(FK)─┼──┐  │
    │    │cliente_id (FK)─┼──┼───┼────┐   │   │data_apl_id(FK)─┼──┼┐ │
    │    │produto_id (FK)─┼──┼───┼────┼───┼───┤cliente_id (FK)─┼──┼┤ │
    │    │regiao_id (FK)──┼──┼───┼────┼───┼───┤produto_id (FK)─┼──┼┤ │
    │    │vendedor_id(FK)─┼──┼───┼────┘   │   └────────────────┘  ││ │
    │    └────────────────┘  │   │        └───────────────────────┘│ │
    └────────────────────────┼───┘                                 │ │
                             │                                     │ │
         ┌───────────────────┼─────────────────────────────────────┘ │
         │                   │                                       │
         ▼                   ▼                                       ▼
    ┌─────────┐      ┌──────────┐      ┌──────────┐       ┌─────────┐
    │DIM_DATA │      │DIM_CLIENT│      │DIM_PRODUT│       │DIM_REGIA│
    └─────────┘      └──────────┘      └──────────┘       └─────────┘


         ┌──────────────────────────────────────┐
         │       FACT_METAS                     │
         │   ┌────────────────┐                 │
         │   │meta_id (PK)    │                 │
         │   │vendedor_id(FK)─┼─────────►DIM_VENDEDOR
         │   │data_id (FK)────┼─────────►DIM_DATA
         │   └────────────────┘                 │
         └──────────────────────────────────────┘
```

**Legenda:**
- `→` : Relacionamento N:1 (FK)
- `◄` : Relacionamento 1:N (reverso)
- `◄┐` : Self-join (mesma tabela)

---

## 📋 Checklist de Validação

```sql
-- 1. Verificar todas FKs existem
SELECT 
    OBJECT_NAME(parent_object_id) AS tabela_fato,
    name AS fk_name,
    OBJECT_NAME(referenced_object_id) AS tabela_dimensao
FROM sys.foreign_keys
WHERE OBJECT_SCHEMA_NAME(parent_object_id) IN ('fact', 'dim')
ORDER BY tabela_fato, fk_name;

-- 2. Verificar órfãos (deveria retornar 0)
SELECT COUNT(*) AS vendas_orfas
FROM fact.FACT_VENDAS fv
WHERE NOT EXISTS (SELECT 1 FROM dim.DIM_DATA d WHERE d.data_id = fv.data_id)
   OR NOT EXISTS (SELECT 1 FROM dim.DIM_CLIENTE c WHERE c.cliente_id = fv.cliente_id)
   OR NOT EXISTS (SELECT 1 FROM dim.DIM_PRODUTO p WHERE p.produto_id = fv.produto_id);

-- 3. Verificar integridade de valores calculados
SELECT COUNT(*) AS inconsistencias
FROM fact.FACT_VENDAS
WHERE valor_total_liquido <> (valor_total_bruto - valor_total_descontos);

-- 4. Verificar metas duplicadas
SELECT vendedor_id, data_id, tipo_periodo, COUNT(*)
FROM fact.FACT_METAS
GROUP BY vendedor_id, data_id, tipo_periodo
HAVING COUNT(*) > 1;
```

---

## 📚 Próximos Documentos

- **[Queries Analíticas](../queries/README.md)** - Exemplos práticos usando relacionamentos
- **[Decisões de Design](../decisoes/01_decisoes_modelagem.md)** - Por que modelamos assim

---

<div align="center">

**[⬆ Voltar ao topo](#-relacionamentos-e-integridade-referencial)**

</div>