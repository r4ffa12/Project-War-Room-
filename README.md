# Project-War-Room-


# 📊 Backlog Analytics — Análise Operacional com SQL

Projeto desenvolvido para análise de uma base operacional de **backlog**, utilizando **SQLite e SQL**, com o objetivo de transformar análises manuais realizadas em Excel em consultas estruturadas e indicadores operacionais.

O projeto permite analisar a evolução do backlog, reincidências, produtividade, qualidade dos tratamentos, testes de parâmetros, qualidade dos dados, redes, cidades, armários e CTOs.

---

# 🎯 Objetivo

O projeto busca responder perguntas como:

- O backlog está diminuindo ou apenas girando?
- Quantos registros entram e saem por dia?
- Quais registros permanecem há mais tempo?
- Quais designadores são reincidentes?
- Quais armários possuem maior concentração de problemas?
- Quais CTOs possuem múltiplas ocorrências?
- Quais clusters estão acumulando backlog?
- Qual é a taxa de associação?
- Existem inconsistências entre teste e status?
- Existem registros sem VLAN, operador ou cluster?
- Qual rede possui maior volume?
- Quais cidades concentram mais problemas?

---

# 🗂️ Estrutura do banco

A análise utiliza principalmente a tabela:

```text
backlog
```

Principais campos utilizados:

```text
data_ref
designador
dias
cluster
armario
cto
status
motivo
operador
teste
vlan
sp_calc
sp_shelf
sp_final
cidade
rede
```

Outras tabelas presentes no projeto:

```text
historico
teste_parametros
cto
base_operador
importacoes
```

---

# 📌 1. Estoque diário do backlog

Mostra a quantidade de registros existentes em cada dia.

```sql
SELECT
    data_ref,
    COUNT(DISTINCT designador) AS estoque
FROM backlog
GROUP BY data_ref
ORDER BY data_ref;
```

### Objetivo

Permite acompanhar se o estoque do backlog está aumentando ou diminuindo ao longo dos dias.

---

# 📌 2. Entradas e saídas por dia

Compara os designadores de um dia com o dia anterior.

```sql
WITH dias AS (
    SELECT DISTINCT data_ref
    FROM backlog
),
base AS (
    SELECT
        data_ref,
        designador
    FROM backlog
    WHERE TRIM(COALESCE(designador, '')) <> ''
    GROUP BY data_ref, designador
),
comparacao AS (
    SELECT
        d.data_ref,
        COUNT(DISTINCT b.designador) AS estoque,
        COUNT(DISTINCT CASE
            WHEN p.designador IS NULL THEN b.designador
        END) AS entradas
    FROM dias d
    LEFT JOIN base b
        ON b.data_ref = d.data_ref
    LEFT JOIN base p
        ON p.data_ref = (
            SELECT MAX(data_ref)
            FROM dias
            WHERE data_ref < d.data_ref
        )
        AND p.designador = b.designador
    GROUP BY d.data_ref
)
SELECT *
FROM comparacao
ORDER BY data_ref;
```

---

# 📌 3. Entradas, saídas e estoque

Consulta consolidada para acompanhar a movimentação diária.

```sql
WITH base AS (
    SELECT DISTINCT
        data_ref,
        TRIM(designador) AS designador
    FROM backlog
    WHERE TRIM(COALESCE(designador, '')) <> ''
),
dias AS (
    SELECT DISTINCT data_ref
    FROM base
),
movimento AS (
    SELECT
        d.data_ref,

        COUNT(DISTINCT atual.designador) AS estoque,

        COUNT(DISTINCT CASE
            WHEN anterior.designador IS NULL
            THEN atual.designador
        END) AS entradas,

        COUNT(DISTINCT CASE
            WHEN atual.designador IS NULL
            THEN anterior.designador
        END) AS saidas

    FROM dias d

    LEFT JOIN base atual
        ON atual.data_ref = d.data_ref

    LEFT JOIN base anterior
        ON anterior.data_ref = (
            SELECT MAX(data_ref)
            FROM dias
            WHERE data_ref < d.data_ref
        )

    GROUP BY d.data_ref
)
SELECT *
FROM movimento
ORDER BY data_ref;
```

---

# 📌 4. Idade do backlog

Distribui os registros por faixa de idade.

```sql
SELECT
    data_ref,
    CASE
        WHEN CAST(dias AS REAL) <= 1 THEN '0-1 dias'
        WHEN CAST(dias AS REAL) <= 3 THEN '2-3 dias'
        WHEN CAST(dias AS REAL) <= 7 THEN '4-7 dias'
        WHEN CAST(dias AS REAL) <= 15 THEN '8-15 dias'
        WHEN CAST(dias AS REAL) <= 30 THEN '16-30 dias'
        ELSE '>30 dias'
    END AS faixa_idade,
    COUNT(*) AS quantidade
FROM backlog
GROUP BY
    data_ref,
    faixa_idade
ORDER BY
    data_ref,
    faixa_idade;
```

---

# 📌 5. Backlog por cluster

```sql
SELECT
    data_ref,
    COALESCE(NULLIF(TRIM(cluster), ''), 'SEM CLUSTER') AS cluster,
    COUNT(DISTINCT designador) AS estoque
FROM backlog
GROUP BY
    data_ref,
    cluster
ORDER BY
    data_ref,
    estoque DESC;
```

---

# 📌 6. Clusters acumulando ou reduzindo backlog

```sql
WITH estoque AS (
    SELECT
        data_ref,
        COALESCE(NULLIF(TRIM(cluster), ''), 'SEM CLUSTER') AS cluster,
        COUNT(DISTINCT designador) AS quantidade
    FROM backlog
    GROUP BY data_ref, cluster
),
comparacao AS (
    SELECT
        atual.cluster,
        atual.data_ref,
        atual.quantidade AS estoque_atual,
        anterior.quantidade AS estoque_anterior,
        atual.quantidade - COALESCE(anterior.quantidade, 0) AS variacao
    FROM estoque atual
    LEFT JOIN estoque anterior
        ON anterior.cluster = atual.cluster
        AND anterior.data_ref = (
            SELECT MAX(data_ref)
            FROM estoque
            WHERE data_ref < atual.data_ref
              AND cluster = atual.cluster
        )
)
SELECT
    *,
    CASE
        WHEN variacao > 0 THEN 'ACUMULANDO'
        WHEN variacao < 0 THEN 'REDUZINDO'
        ELSE 'ESTÁVEL'
    END AS comportamento
FROM comparacao
ORDER BY data_ref, ABS(variacao) DESC;
```

---

# 📌 7. Designadores reincidentes

Identifica designadores que aparecem em mais de um dia.

```sql
SELECT
    TRIM(designador) AS designador,
    COUNT(DISTINCT data_ref) AS dias_no_backlog,
    MIN(data_ref) AS primeira_ocorrencia,
    MAX(data_ref) AS ultima_ocorrencia
FROM backlog
WHERE TRIM(COALESCE(designador, '')) <> ''
GROUP BY TRIM(designador)
HAVING COUNT(DISTINCT data_ref) > 1
ORDER BY dias_no_backlog DESC;
```

---

# 📌 8. Percentual de reincidência

```sql
WITH base AS (
    SELECT
        TRIM(designador) AS designador,
        COUNT(DISTINCT data_ref) AS dias
    FROM backlog
    WHERE TRIM(COALESCE(designador, '')) <> ''
    GROUP BY TRIM(designador)
),
totais AS (
    SELECT
        COUNT(*) AS total_designadores,
        SUM(CASE WHEN dias > 1 THEN 1 ELSE 0 END) AS reincidentes
    FROM base
)
SELECT
    total_designadores,
    reincidentes,
    ROUND(
        100.0 * reincidentes / NULLIF(total_designadores, 0),
        2
    ) AS percentual_reincidente
FROM totais;
```

---

# 📌 9. Top 20 armários

```sql
SELECT
    COALESCE(NULLIF(TRIM(armario), ''), 'SEM ARMARIO') AS armario,
    COUNT(DISTINCT designador) AS ocorrencias,
    COUNT(DISTINCT cto) AS quantidade_ctos,
    ROUND(AVG(CAST(dias AS REAL)), 2) AS idade_media,
    MAX(CAST(dias AS REAL)) AS maior_idade
FROM backlog
GROUP BY armario
ORDER BY ocorrencias DESC
LIMIT 20;
```

---

# 📌 10. CTOs com múltiplos pedidos

```sql
SELECT
    COALESCE(NULLIF(TRIM(armario), ''), 'SEM ARMARIO') AS armario,
    COALESCE(NULLIF(TRIM(cto), ''), 'SEM CTO') AS cto,
    COUNT(DISTINCT designador) AS quantidade_designadores
FROM backlog
GROUP BY
    armario,
    cto
HAVING COUNT(DISTINCT designador) > 1
ORDER BY quantidade_designadores DESC;
```

---

# 📌 11. Registros tratados que retornaram

```sql
WITH ocorrencias AS (
    SELECT
        TRIM(designador) AS designador,
        data_ref,
        status,
        ROW_NUMBER() OVER (
            PARTITION BY TRIM(designador)
            ORDER BY data_ref
        ) AS ordem
    FROM backlog
    WHERE TRIM(COALESCE(designador, '')) <> ''
)
SELECT
    designador,
    MIN(data_ref) AS primeira_data,
    MAX(data_ref) AS ultima_data,
    COUNT(DISTINCT data_ref) AS dias,
    GROUP_CONCAT(
        DISTINCT COALESCE(NULLIF(TRIM(status), ''), 'SEM STATUS')
    ) AS status_observados
FROM ocorrencias
GROUP BY designador
HAVING COUNT(DISTINCT data_ref) > 1
ORDER BY dias DESC;
```

---

# 📌 12. Distribuição de status por cluster

```sql
SELECT
    data_ref,
    COALESCE(NULLIF(TRIM(cluster), ''), 'SEM CLUSTER') AS cluster,
    COALESCE(NULLIF(TRIM(status), ''), 'SEM STATUS') AS status,
    COUNT(*) AS quantidade
FROM backlog
GROUP BY
    data_ref,
    cluster,
    status
ORDER BY
    data_ref,
    cluster,
    quantidade DESC;
```

---

# 📌 13. Percentual de SEM EVENTO ARD

```sql
SELECT
    data_ref,
    COUNT(*) AS total,
    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(motivo, ''))) = 'SEM EVENTO ARD'
            THEN 1
            ELSE 0
        END
    ) AS sem_evento_ard,
    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN UPPER(TRIM(COALESCE(motivo, ''))) = 'SEM EVENTO ARD'
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS percentual
FROM backlog
GROUP BY data_ref
ORDER BY data_ref;
```

---

# 📌 14. SEM EVENTO ARD por operador

```sql
SELECT
    COALESCE(NULLIF(TRIM(operador), ''), 'SEM OPERADOR') AS operador,
    COUNT(*) AS total,
    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(motivo, ''))) = 'SEM EVENTO ARD'
            THEN 1
            ELSE 0
        END
    ) AS sem_evento_ard,
    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN UPPER(TRIM(COALESCE(motivo, ''))) = 'SEM EVENTO ARD'
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS percentual
FROM backlog
GROUP BY operador
ORDER BY sem_evento_ard DESC;
```

---

# 📌 15. Taxa de associação por cluster

```sql
SELECT
    data_ref,
    COALESCE(NULLIF(TRIM(cluster), ''), 'SEM CLUSTER') AS cluster,
    COUNT(*) AS trabalhados,
    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(status, ''))) = 'ASSOCIADO'
            THEN 1
            ELSE 0
        END
    ) AS associados,
    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN UPPER(TRIM(COALESCE(status, ''))) = 'ASSOCIADO'
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS taxa_associacao
FROM backlog
WHERE TRIM(COALESCE(status, '')) <> ''
GROUP BY data_ref, cluster
ORDER BY data_ref, taxa_associacao DESC;
```

---

# 📌 16. Taxa de associação por operador

```sql
SELECT
    data_ref,
    COALESCE(NULLIF(TRIM(operador), ''), 'SEM OPERADOR') AS operador,
    COUNT(*) AS trabalhados,
    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(status, ''))) = 'ASSOCIADO'
            THEN 1
            ELSE 0
        END
    ) AS associados,
    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN UPPER(TRIM(COALESCE(status, ''))) = 'ASSOCIADO'
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS taxa_associacao
FROM backlog
WHERE TRIM(COALESCE(status, '')) <> ''
GROUP BY data_ref, operador
ORDER BY data_ref, taxa_associacao DESC;
```

---

# 📌 17. Mesmo designador com status diferentes

```sql
SELECT
    TRIM(designador) AS designador,
    COUNT(DISTINCT UPPER(TRIM(status))) AS quantidade_status,
    GROUP_CONCAT(
        DISTINCT UPPER(TRIM(status))
    ) AS status_observados,
    COUNT(DISTINCT data_ref) AS dias
FROM backlog
WHERE TRIM(COALESCE(designador, '')) <> ''
  AND TRIM(COALESCE(status, '')) <> ''
GROUP BY TRIM(designador)
HAVING COUNT(DISTINCT UPPER(TRIM(status))) > 1
ORDER BY dias DESC;
```

---

# 📌 18. Produtividade por operador

```sql
SELECT
    data_ref,
    COALESCE(NULLIF(TRIM(operador), ''), 'SEM OPERADOR') AS operador,
    COUNT(*) AS total_registros,

    SUM(
        CASE
            WHEN TRIM(COALESCE(status, '')) <> ''
            THEN 1
            ELSE 0
        END
    ) AS trabalhados,

    SUM(
        CASE
            WHEN TRIM(COALESCE(status, '')) = ''
            THEN 1
            ELSE 0
        END
    ) AS a_trabalhar,

    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN TRIM(COALESCE(status, '')) <> ''
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS percentual_trabalhado

FROM backlog
GROUP BY data_ref, operador
ORDER BY data_ref, total_registros DESC;
```

---

# 📌 19. Teste × Status

```sql
SELECT
    UPPER(TRIM(COALESCE(teste, 'SEM TESTE'))) AS teste,
    UPPER(TRIM(COALESCE(status, 'SEM STATUS'))) AS status,
    COUNT(*) AS quantidade
FROM backlog
GROUP BY
    teste,
    status
ORDER BY quantidade DESC;
```

---

# 📌 20. Inconsistências entre teste e status

```sql
SELECT
    data_ref,
    designador,
    cluster,
    armario,
    cto,
    teste,
    status,
    motivo
FROM backlog
WHERE
    (
        UPPER(TRIM(COALESCE(teste, ''))) = 'OK'
        AND
        UPPER(TRIM(COALESCE(status, ''))) = 'NÃO ASSOCIADO'
    )
    OR
    (
        UPPER(TRIM(COALESCE(teste, ''))) = 'NOK'
        AND
        UPPER(TRIM(COALESCE(status, ''))) = 'FORA DO EVENTO'
    )
ORDER BY data_ref, armario, cto;
```

---

# 📌 21. Cobertura dos testes

```sql
SELECT
    data_ref,
    COUNT(*) AS total,

    SUM(
        CASE
            WHEN TRIM(COALESCE(teste, '')) <> ''
            THEN 1
            ELSE 0
        END
    ) AS registros_com_teste,

    SUM(
        CASE
            WHEN TRIM(COALESCE(teste, '')) = ''
            THEN 1
            ELSE 0
        END
    ) AS registros_sem_teste,

    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN TRIM(COALESCE(teste, '')) <> ''
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS cobertura_percentual

FROM backlog
GROUP BY data_ref
ORDER BY data_ref;
```

---

# 📌 22. VLAN S/D por dia

```sql
SELECT
    data_ref,
    COUNT(*) AS total,

    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(vlan, ''))) = 'S/D'
            THEN 1
            ELSE 0
        END
    ) AS vlan_sd,

    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN UPPER(TRIM(COALESCE(vlan, ''))) = 'S/D'
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS percentual_vlan_sd

FROM backlog
GROUP BY data_ref
ORDER BY data_ref;
```

---

# 📌 23. VLAN S/D por cluster

```sql
SELECT
    data_ref,
    COALESCE(NULLIF(TRIM(cluster), ''), 'SEM CLUSTER') AS cluster,
    COUNT(*) AS total,

    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(vlan, ''))) = 'S/D'
            THEN 1
            ELSE 0
        END
    ) AS vlan_sd,

    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN UPPER(TRIM(COALESCE(vlan, ''))) = 'S/D'
                THEN 1
                ELSE 0
            END
        ) / NULLIF(COUNT(*), 0),
        2
    ) AS percentual_vlan_sd

FROM backlog
GROUP BY data_ref, cluster
ORDER BY data_ref, vlan_sd DESC;
```

---

# 📌 24. Divergência entre SP calculado e SP shelf

```sql
SELECT
    data_ref,

    COUNT(*) AS total_com_sp,

    SUM(
        CASE
            WHEN
                TRIM(COALESCE(sp_calc, '')) <> ''
                AND TRIM(COALESCE(sp_shelf, '')) <> ''
                AND UPPER(TRIM(sp_calc)) <> UPPER(TRIM(sp_shelf))
            THEN 1
            ELSE 0
        END
    ) AS divergencias,

    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN
                    TRIM(COALESCE(sp_calc, '')) <> ''
                    AND TRIM(COALESCE(sp_shelf, '')) <> ''
                    AND UPPER(TRIM(sp_calc)) <> UPPER(TRIM(sp_shelf))
                THEN 1
                ELSE 0
            END
        )
        /
        NULLIF(
            SUM(
                CASE
                    WHEN
                        TRIM(COALESCE(sp_calc, '')) <> ''
                        AND TRIM(COALESCE(sp_shelf, '')) <> ''
                    THEN 1
                    ELSE 0
                END
            ),
            0
        ),
        2
    ) AS percentual_divergencia

FROM backlog
GROUP BY data_ref
ORDER BY data_ref;
```

---

# 📌 25. Lista de divergências de SP

```sql
SELECT
    data_ref,
    designador,
    cluster,
    armario,
    cto,
    sp_calc,
    sp_shelf,
    sp_final
FROM backlog
WHERE
    TRIM(COALESCE(sp_calc, '')) <> ''
    AND TRIM(COALESCE(sp_shelf, '')) <> ''
    AND UPPER(TRIM(sp_calc)) <> UPPER(TRIM(sp_shelf))
ORDER BY data_ref, armario, cto;
```

---

# 📌 26. Registros sem operador ou cluster

```sql
SELECT
    data_ref,
    designador,
    cluster,
    operador,
    armario,
    cto,
    status
FROM backlog
WHERE
    TRIM(COALESCE(operador, '')) = ''
    OR
    TRIM(COALESCE(cluster, '')) = ''
ORDER BY data_ref, designador;
```

---

# 📌 27. Comparação VIVO2 × FIBRASIL

```sql
SELECT
    data_ref,
    UPPER(TRIM(COALESCE(rede, 'SEM REDE'))) AS rede,
    COUNT(DISTINCT designador) AS estoque,
    ROUND(AVG(CAST(dias AS REAL)), 2) AS idade_media,

    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(status, ''))) = 'ASSOCIADO'
            THEN 1
            ELSE 0
        END
    ) AS associados,

    ROUND(
        100.0 *
        SUM(
            CASE
                WHEN UPPER(TRIM(COALESCE(status, ''))) = 'ASSOCIADO'
                THEN 1
                ELSE 0
            END
        )
        /
        NULLIF(COUNT(*), 0),
        2
    ) AS taxa_associacao

FROM backlog
WHERE UPPER(TRIM(COALESCE(rede, ''))) IN ('VIVO2', 'FIBRASIL')
GROUP BY data_ref, rede
ORDER BY data_ref, rede;
```

---

# 📌 28. Cidades com maior quantidade de problemas

```sql
SELECT
    COALESCE(NULLIF(TRIM(cidade), ''), 'SEM CIDADE') AS cidade,
    COUNT(DISTINCT designador) AS ocorrencias,
    ROUND(AVG(CAST(dias AS REAL)), 2) AS idade_media,
    MAX(CAST(dias AS REAL)) AS maior_idade
FROM backlog
GROUP BY cidade
ORDER BY ocorrencias DESC;
```

---

# 📌 29. Ranking completo de armários

```sql
SELECT
    COALESCE(NULLIF(TRIM(armario), ''), 'SEM ARMARIO') AS armario,

    COUNT(DISTINCT designador) AS ocorrencias,

    COUNT(DISTINCT cto) AS ctos,

    ROUND(
        AVG(CAST(dias AS REAL)),
        2
    ) AS idade_media,

    MAX(CAST(dias AS REAL)) AS maior_idade,

    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(status, ''))) = 'ASSOCIADO'
            THEN 1
            ELSE 0
        END
    ) AS associados,

    SUM(
        CASE
            WHEN UPPER(TRIM(COALESCE(teste, ''))) = 'NOK'
            THEN 1
            ELSE 0
        END
    ) AS testes_nok

FROM backlog
GROUP BY armario
ORDER BY ocorrencias DESC;
```

---

# 📌 30. Dashboard com cinco indicadores

Consulta consolidada para obter:

- Estoque;
- Entradas;
- Saídas;
- Percentual trabalhado;
- Percentual reincidente.

```sql
WITH datas AS (
    SELECT MAX(data_ref) AS data_atual
    FROM backlog
),
anterior AS (
    SELECT MAX(data_ref) AS data_anterior
    FROM backlog
    WHERE data_ref < (
        SELECT data_atual
        FROM datas
    )
),
atual AS (
    SELECT DISTINCT
        TRIM(designador) AS designador
    FROM backlog
    WHERE data_ref = (SELECT data_atual FROM datas)
      AND TRIM(COALESCE(designador, '')) <> ''
),
base_anterior AS (
    SELECT DISTINCT
        TRIM(designador) AS designador
    FROM backlog
    WHERE data_ref = (SELECT data_anterior FROM anterior)
      AND TRIM(COALESCE(designador, '')) <> ''
),
movimento AS (
    SELECT
        (SELECT COUNT(*) FROM atual) AS estoque,

        (
            SELECT COUNT(*)
            FROM atual a
            WHERE NOT EXISTS (
                SELECT 1
                FROM base_anterior b
                WHERE b.designador = a.designador
            )
        ) AS entradas,

        (
            SELECT COUNT(*)
            FROM base_anterior b
            WHERE NOT EXISTS (
                SELECT 1
                FROM atual a
                WHERE a.designador = b.designador
            )
        ) AS saidas
),
trabalho AS (
    SELECT
        COUNT(*) AS total,
        SUM(
            CASE
                WHEN TRIM(COALESCE(status, '')) <> ''
                THEN 1
                ELSE 0
            END
        ) AS trabalhados
    FROM backlog
    WHERE data_ref = (SELECT data_atual FROM datas)
),
reincidencia AS (
    SELECT
        COUNT(*) AS total,

        SUM(
            CASE
                WHEN dias_ocorrencia > 1
                THEN 1
                ELSE 0
            END
        ) AS reincidentes

    FROM (
        SELECT
            TRIM(designador) AS designador,
            COUNT(DISTINCT data_ref) AS dias_ocorrencia
        FROM backlog
        WHERE TRIM(COALESCE(designador, '')) <> ''
        GROUP BY TRIM(designador)
    )
)
SELECT
    (SELECT data_atual FROM datas) AS data_ref,
    movimento.estoque,
    movimento.entradas,
    movimento.saidas,

    ROUND(
        100.0 * trabalho.trabalhados /
        NULLIF(trabalho.total, 0),
        2
    ) AS percentual_trabalhado,

    ROUND(
        100.0 * reincidencia.reincidentes /
        NULLIF(reincidencia.total, 0),
        2
    ) AS percentual_reincidente

FROM movimento, trabalho, reincidencia;
```

---

# 📌 31. Reincidência por armário e CTO

```sql
SELECT
    COALESCE(NULLIF(TRIM(armario), ''), 'SEM ARMARIO') AS armario,
    COALESCE(NULLIF(TRIM(cto), ''), 'SEM CTO') AS cto,

    COUNT(DISTINCT designador) AS designadores,

    COUNT(DISTINCT data_ref) AS dias_com_ocorrencia,

    ROUND(
        AVG(CAST(dias AS REAL)),
        2
    ) AS idade_media,

    MAX(CAST(dias AS REAL)) AS maior_idade

FROM backlog
WHERE TRIM(COALESCE(designador, '')) <> ''

GROUP BY
    armario,
    cto

HAVING COUNT(DISTINCT data_ref) > 1

ORDER BY
    dias_com_ocorrencia DESC,
    designadores DESC;
```

---

# 📊 Principais perguntas respondidas

Com os scripts acima, é possível responder:

### Backlog

> O backlog está diminuindo ou aumentando?

### Idade

> Quantos registros estão há mais de 7, 15 ou 30 dias?

### Reincidência

> Quais registros estão aparecendo repetidamente?

### Armários

> Quais armários concentram mais ocorrências?

### CTO

> Quais CTOs possuem múltiplos problemas?

### Tratamento

> Qual é a taxa de associação?

### Operadores

> Qual é a quantidade de registros trabalhados por operador?

### Testes

> Existem testes incompatíveis com os status?

### Qualidade

> Quantos registros possuem VLAN `S/D`?

### SP

> Existem divergências entre SP calculado e SP shelf?

### Rede

> Como VIVO2 e FIBRASIL se comportam?

### Geografia

> Quais cidades concentram o maior backlog?

---

# ⚠️ Cuidados na interpretação

Os resultados precisam ser analisados considerando algumas limitações.

### 1. Poucos dias de dados

Com apenas dois dias de histórico, ainda não é possível afirmar uma tendência mensal.

O ideal é acumular pelo menos **2 a 3 semanas**.

### 2. Arquivos incompletos

Se uma exportação diária estiver incompleta, a consulta poderá interpretar os registros ausentes como **saídas do backlog**.

### 3. Histórico

A tabela `historico` precisa estar preenchida para análises temporais mais precisas.

### 4. Causa raiz

O SQL identifica:

```text
O QUE aconteceu
ONDE aconteceu
QUANDO aconteceu
```

Mas não necessariamente explica:

```text
POR QUE aconteceu
```

A causa raiz deve ser validada com a operação.

---

# 🚀 Próximos passos

O projeto pode evoluir para uma plataforma completa de análise.

## Etapa 1 — SQL

Criar `VIEWs` para os principais indicadores.

## Etapa 2 — Automação

Automatizar a importação dos arquivos diários.

## Etapa 3 — Dashboard

Criar dashboard com:

- Estoque;
- Entradas;
- Saídas;
- Reincidência;
- Idade;
- Associação;
- Testes;
- VLAN;
- SP;
- Armários;
- CTOs;
- Cidades.

## Etapa 4 — Aplicação Web

Criar uma interface utilizando:

```text
HTML
CSS
JavaScript
```

ou posteriormente:

```text
React
```

## Etapa 5 — Automação completa

Fluxo planejado:

```text
Arquivo diário
      ↓
Importação
      ↓
SQLite
      ↓
SQL
      ↓
Indicadores
      ↓
Dashboard
      ↓
Relatório operacional
```

---

# 🔐 Segurança

Não publique no GitHub dados reais que contenham informações internas, operacionais ou sensíveis da empresa.

Recomenda-se utilizar:

- Dados anonimizados;
- Dados fictícios;
- Banco de exemplo;
- Apenas os scripts SQL.

Evite publicar diretamente:

```text
backlog.db
```

caso contenha dados reais.

---

# 👨‍💻 Tecnologias

- SQLite
- SQL
- Git
- GitHub
- HTML
- CSS
- Python

---

# 📁 Estrutura recomendada

```text
backlog-analytics/
│
├── README.md
│
├── sql/
│   └── analises_backlog.sql
│
├── docs/
│   └── scripts_analise_backlog_sqlite.pdf
│
└── data/
    └── exemplo_backlog.db
```

---

# 📌 Sobre o projeto

Este projeto nasceu da necessidade de transformar uma análise operacional realizada em planilhas em uma solução baseada em **SQL, banco de dados e indicadores**.

A proposta é reduzir o trabalho manual, aumentar a confiabilidade das análises e permitir que os dados sejam utilizados posteriormente em dashboards e aplicações web.

---

## 📈 Evolução do projeto

```text
Excel
  ↓
SQLite
  ↓
SQL
  ↓
Views
  ↓
Dashboard
  ↓
Automação
  ↓
Plataforma Web
```

---

**Projeto de análise de dados e automação operacional utilizando SQL e SQLite.**
