---
name: databricks-audit-rules
description: >-
  Audita e valida códigos Python, SQL e notebooks Databricks quanto a padrões de
  engenharia de software, governança no Unity Catalog, proteção de memória no
  driver, segurança de credenciais e prontidão para CI/CD via Databricks Asset
  Bundles (DABs), gerando relatório com IDs de regra, severidade e veredito.
  Use ao revisar, auditar, fazer code review ou validar notebooks, scripts
  PySpark ou modelos SQL do Databricks antes de commit, Pull Request ou deploy —
  por exemplo "audite meu notebook", "revise este código Databricks", "isso está
  pronto para produção?". Use when reviewing, linting, auditing, or evaluating
  Databricks notebooks, PySpark scripts, or SQL models. Para remediar
  credenciais vazadas (rotação, secret scopes, ACLs), use
  databricks-security-secrets.
---

# Databricks Audit Rules

Esta skill atua como **Auditor Estático e Consultor de Qualidade de Código para Databricks**. Ela inspeciona notebooks, módulos `.py` e arquivos `.sql` e identifica violações estruturais, vulnerabilidades de segurança, riscos de estouro de memória no driver (*OOM*) e impedimentos de automação antes que o código chegue a ambientes compartilhados.

A auditoria é **baseada em evidência**: todo achado cita arquivo, linha e trecho. Não declare conformidade sem evidência e não aponte violação sem evidência. Quando algo não puder ser verificado com o material disponível, marque como **N/V (não verificável)**. Não presuma nem o pior nem o melhor caso.

---

## 1. Quando usar e escopo

**Acionar em:**
- Code reviews e Pull Requests de projetos Databricks.
- Refatoração de notebooks exploratórios em pipelines de produção.
- Validação antes de deploy via Databricks Asset Bundles (DABs).
- Pedidos como *"audite meu código"*, *"verifique as boas práticas deste notebook"*, *"este script está pronto para produção?"*.

**Abrangência:** análise estática, conformidade com Unity Catalog, separação de I/O e lógica de negócio, higiene de repositório, uso seguro de computação e prontidão para CI/CD.

**Limites:** esta skill **não** executa deploys nem comandos na nuvem (escopo de CI/CD) e **não** redefine a modelagem conceitual de negócio (escopo de arquitetura de dados).

---

## 2. Severidades e veredito

| Severidade | Significado |
|---|---|
| 🔴 **Bloqueador** | Risco de segurança ou de perda de dados. Impede merge/deploy. |
| 🟠 **Alto** | Quebra governança, portabilidade ou estabilidade em produção. Corrigir antes do deploy. |
| 🟡 **Médio** | Prejudica manutenção, observabilidade ou testabilidade. Corrigir no ciclo atual. |
| 🔵 **Sugestão** | Melhoria recomendada, sem impacto imediato. |

**Veredito final (objetivo):**
- ❌ **Reprovado**: há 1 ou mais Bloqueadores não excepcionados.
- ⚠️ **Aprovado com ressalvas**: não há Bloqueador, mas há achados Altos ou Médios.
- ✅ **Aprovado**: só há Sugestões, ou nenhum achado.

**Evidência ambígua:** se a evidência não for conclusiva (por exemplo, o volume do DataFrame é desconhecido), reporte com severidade **um nível abaixo** da padrão e adicione a marca `(verificar)`.

**Notebook exploratório:** se o cabeçalho declarar `Finalidade: exploratório`, rebaixe MEM-01 e MOD-02 para Sugestão. As regras SEC continuam valendo integralmente.

---

## 3. Processo

Siga os passos na ordem. Cada regra tem um ID, que deve aparecer no relatório.

### Passo 0: Identificar e normalizar a entrada

1. **Determine o escopo:** um arquivo isolado, vários arquivos ou um repositório inteiro. Com acesso ao repositório, procure também por `databricks.yml`, `resources/*.yml`, `.gitignore`, `requirements.txt` e `pyproject.toml`.
2. **Normalize o formato de notebook Databricks** antes de auditar:
   - `.py` que começa com `# Databricks notebook source`: as células são separadas por `# COMMAND ----------`.
   - Linhas `# MAGIC %sql`, `# MAGIC %python` etc.: remova o prefixo `# MAGIC ` e audite o conteúdo **na linguagem da magic**. SQL dentro de `%sql` é auditado como SQL.
   - `.sql` que começa com `-- Databricks notebook source`: o mesmo, com o prefixo `-- MAGIC `.
   - `.ipynb`: audite o `source` de cada célula e use o número da célula como referência de localização.
   - SQL dentro de `spark.sql("...")` / `spark.sql(f"...")`: audite como SQL.
3. Reporte as linhas pela numeração **do arquivo original**, não da versão normalizada.

### Passo 1: Cabeçalho e rastreabilidade

**HDR-01: Cabeçalho de governança** (🟡 se ausente; 🔵 se incompleto)
O topo do arquivo deve identificar: autor/time, domínio de negócio, finalidade técnica, camada e destino, e tipo de carga. Modelo de referência (use `--` em SQL):

```python
# Databricks notebook source
# =============================================================
# Autor/Time    : time-dados-financeiro (dados-fin@empresa.com)
# Domínio       : Financeiro / Contas a Receber
# Finalidade    : Consolida títulos em aberto por cliente (D-1)
# Camada        : Silver -> Gold
# Destino       : ${catalog}.gold_financeiro.fato_titulos_abertos
# Tipo de carga : incremental (MERGE) | full-refresh
# =============================================================
```

O campo **Tipo de carga** é usado pela regra DATA-03, e o campo **Camada** pela regra UC-03.

### Passo 2: Segurança e gestão de segredos (prioridade máxima)

**SEC-01: Credencial em texto claro** (🔴, **não admite exceção**)
- Procure atribuições literais a tokens, senhas, chaves de API, connection strings e chaves de storage. A lista de padrões (regex) está em [`references/padroes-segredos.md`](references/padroes-segredos.md).
- A correção obrigatória é `dbutils.secrets.get(scope, key)`, variável de ambiente gerenciada, ou referência a secret em `spark_conf` do job/cluster (`{{secrets/scope/key}}`).
- No relatório, **mascare o valor** (por exemplo `dapi****…a1f3`). Nunca reproduza a credencial completa.
- Credencial encontrada é credencial comprometida. Para o plano de remediação (revogação, rotação, escolha entre Unity Catalog e Secret Scope, ACLs, limpeza do histórico), indique a skill **`databricks-security-secrets`** na seção Correção do achado.

**SEC-02: Exposição de segredo em runtime** (🟠)
- `print()`, `logging`, `display()` ou gravação em tabela de um valor vindo de `dbutils.secrets.get`.
- Transformações que contornam a redação automática do Databricks, como `" ".join(segredo)`, `segredo[::-1]` ou `base64` do segredo.

**SEC-03: Higiene de repositório** (🟠; 🔴 se o arquivo contiver credencial)
- **Com acesso ao repositório:** procure arquivos versionados proibidos, como `.env`, `.databrickscfg`, `*.log`, dados brutos soltos (`*.csv`, `*.parquet`, `*.xlsx`, `*.json` de dados), pastas de backup manual (`backup/`, `*_old.py`, `*_copia.*`). Verifique se o `.gitignore` cobre esses padrões.
- **Só com o arquivo:** sinalize referências a arquivos locais sensíveis (`load_dotenv()`, `open(".env")`, caminhos `C:\` ou `/dbfs/FileStore/...` com dados). Marque a parte de repositório como **N/V**.

### Passo 3: Governança no Unity Catalog e camadas

**UC-01: Resolução determinística de nomes de tabela** (🟠)
- Inspecione `spark.table()`, `spark.read.table()`, `saveAsTable()`, `insertInto()`, `DeltaTable.forName()` e as cláusulas SQL `FROM`, `JOIN`, `INTO`, `TABLE`, `MERGE INTO`, `USING`, `UPDATE`, `DELETE FROM`.
- O objetivo **não** é contar níveis. O objetivo é garantir que todo nome de tabela resolva de forma **explícita e previsível** em qualquer ambiente. A violação é uma referência de 1 nível (`tabela`) ou 2 níveis (`schema.tabela`) que dependa do catálogo/schema padrão do workspace ou do cluster, que pode inclusive ser `hive_metastore`.
- **Referências curtas aceitas**, desde que o contexto seja definido de forma explícita:
  - **2 níveis** quando o arquivo define `USE CATALOG` antes da referência;
  - **1 nível** quando o arquivo define `USE CATALOG` e `USE SCHEMA` antes da referência;
  - o `USE` deve ser **parametrizado** (`USE CATALOG IDENTIFIER(:catalog)`, `spark.sql(f"USE CATALOG {catalog}")`, `${catalog}`). Um `USE CATALOG prod` literal cai em UC-02;
  - **pipelines Lakeflow/DLT:** datasets do próprio pipeline e tabelas do catálogo/schema de destino do pipeline podem ser referenciados sem qualificação, porque esse é o padrão da ferramenta. Fontes **externas** ao pipeline seguem a regra geral;
  - **catálogo/schema padrão definido na configuração** da tarefa do job, do SQL warehouse, do `profiles.yml` do dbt ou de `spark.databricks.sql.initial.catalog.name` no bundle. Com acesso ao repositório, confirme a configuração; sem acesso, veja a regra de severidade abaixo.
- **Severidade:**
  - 🟠 quando não há contexto no arquivo **e** a configuração do job/pipeline foi verificada e também não define;
  - 🟡 `(verificar)` quando não há contexto no arquivo e a configuração não está disponível. Ela pode existir fora do código.
- **Contexto ambíguo** (🟡): vários `USE CATALOG`/`USE SCHEMA` no mesmo arquivo, `USE` dentro de notebook chamado via `%run` alterando o contexto de quem chama, ou mistura de referências curtas e completas para o mesmo catálogo.
- **Referências entre catálogos** (ler de um catálogo diferente do contexto atual) devem usar sempre 3 níveis.
- `hive_metastore.schema.tabela`: 🟠, porque a tabela está fora do Unity Catalog. Recomende a migração.
- **Não são violação:**
  - nomes de CTE (`WITH x AS (...) ... FROM x`);
  - temp views (`createOrReplaceTempView`, `CREATE TEMP VIEW`);
  - `FROM VALUES`, `FROM range(...)`, table-valued functions (`read_files(...)`);
  - leituras por caminho (`delta.\`/Volumes/...\``, `spark.read.load(path)`);
  - aliases de subquery;
  - tabelas de sistema (`system.*`).
- **Formas aceitas:** `f"{catalog}.{schema}.tabela"`, `IDENTIFIER(:catalog || '.schema.tabela')`, `${catalog}.schema.tabela` (substituição de variável do bundle/SQL), parâmetros `:param` em tarefas SQL.

**UC-02: Catálogo/schema fixo por ambiente** (🟠)
- UC-01 exige resolução explícita; UC-02 exige que o catálogo/schema usado nessa resolução seja **parametrizado**, seja no nome de 3 níveis, seja no `USE CATALOG`/`USE SCHEMA`.
- Sinalize literais que identificam ambiente no nome do catálogo ou schema (`prod.`, `dev.`, `hml.`, `stg.`, `*_prd`, `*_dev`), inclusive em `USE CATALOG prod`, e catálogos fixos em pipelines destinados a múltiplos ambientes.
- **Fontes aceitas:** `dbutils.widgets.get()`, parâmetros de job, `argparse`, variáveis de bundle (`${var.catalog}`), `spark.conf.get()`.

**UC-03: Segregação Medallion** (🟡, condicional)
- **Premissa:** a camada é identificada pelo campo `Camada` do cabeçalho ou por `bronze|silver|gold` no nome do catálogo/schema. Se nenhum dos dois existir, marque **N/V** e informe a premissa usada.
- Violações:
  - pipeline Gold lendo diretamente da Bronze;
  - leitura da Bronze fora de pipelines de ingestão da Silver;
  - escrita em camada "anterior" à de origem (por exemplo, Gold → Silver).

### Passo 4: Memória do driver e modularização

**MEM-01: Coleta de dados no driver** (🟠)
- Sinalize `.collect()`, `.toPandas()` e `.toLocalIterator()` cuja cadeia **não** contenha redução prévia: `.limit(n)`, `.agg(...)`, `.groupBy(...).agg(...)`, `.count()`, ou um filtro com volume documentado como trivial.
- **Não são violação:** `.take(n)`, `.head(n)`, `.first()`, `.limit(n).collect()` com `n` literal ≤ 1000, e escalares de agregação (`spark.sql("SELECT max(dt) ...").collect()[0][0]`).
- `.toPandas()` de tabela inteira em produção: 🟠 sem rebaixamento.
- **Correção:** manter o processamento distribuído (Spark/Photon), usar `pandas_udf`/`applyInPandas`, ou pandas API on Spark (`pyspark.pandas`).

**MOD-01: I/O misturado com transformação** (🟡)
- Funções que leem (`spark.read`, `spark.table`), transformam **e** gravam (`.write`, `saveAsTable`) no mesmo corpo.
- **Padrão esperado:** funções puras `DataFrame -> DataFrame` para as regras de negócio, com leitura e gravação isoladas nas bordas (orquestração).
- Com acesso ao repositório, se não houver testes (`tests/`, `test_*.py`) das funções de transformação: 🔵.

**MOD-02: Comandos interativos em produção** (🟡)
- `display()`, `df.show()`, `df.printSchema()`, `print(df.collect())`, `df.describe().show()` e prints em loop que poluem os logs de orquestração.
- **Correção:** remover, ou trocar por `logging` em nível adequado e métricas pontuais (contagens).

### Passo 5: Operações destrutivas e integridade de dados

**DATA-01: Remoção massiva sem filtro** (🔴)
- `DELETE FROM tabela` sem `WHERE`, `TRUNCATE TABLE`, `DeltaTable.delete()` sem condição.

**DATA-02: DROP em código de produção** (🟠)
- `DROP TABLE`/`DROP VIEW`/`DROP SCHEMA` em pipelines recorrentes. O risco está na existência do DROP em si, não apenas na falta de `IF EXISTS`.
- `IF EXISTS` torna o comando idempotente, mas também faz o drop passar em silêncio. Quando o DROP for justificado (por exemplo, limpeza de staging), exija `IF EXISTS` (🟡 se faltar) e escopo restrito a objetos temporários ou de staging.
- `DROP SCHEMA ... CASCADE`: 🔴.

**DATA-03: Sobrescrita total em carga incremental** (🟠)
- `mode("overwrite")`, `INSERT OVERWRITE` ou `CREATE OR REPLACE TABLE` **sem** `replaceWhere` / `partitionOverwriteMode=dynamic`, quando o cabeçalho não declarar `Tipo de carga: full-refresh`.
- No Delta, o overwrite **não apaga o histórico** (o time travel continua disponível dentro da retenção). O risco real é substituir a tabela inteira sem querer em uma rotina incremental.
- **Alternativas idempotentes:** `MERGE INTO` (quando há chave de negócio), `replaceWhere` por partição/período, ou dynamic partition overwrite. Recomende conforme o caso: `MERGE` tem custo maior e não é sempre a melhor opção.
- Se o tipo de carga não estiver declarado: 🟡 `(verificar)`.

### Passo 6: Prontidão para CI/CD e jobs

**CICD-01: Execução a partir de pasta pessoal** (🟠)
- Sinalize `notebook_path`, `%run`, `dbutils.notebook.run()` ou `python_file` que apontem para caminho absoluto pessoal (`/Workspace/Users/<email>/...`, `/Users/<email>/...`) **fora** de uma pasta `.bundle/`.
- **Não são violação:**
  - caminhos relativos dentro do bundle (`../src/nb.py`);
  - `${workspace.file_path}`;
  - `git_source` em jobs;
  - os caminhos `/Workspace/Users/<user-ou-SP>/.bundle/...` gerados pelo deploy de DABs.
- `/Repos/...`: 🔵, porque é legado. Recomende Git folders + DABs.

**CICD-02: Gestão de dependências** (🟡)
- `%pip install pacote` **sem versão fixada** (`==`) ou com `>=` aberto.
- `%pip install` espalhado por várias células: 🔵. Consolide na primeira célula, seguido de `dbutils.library.restartPython()`.
- `dbutils.library.installPyPI`: 🟡, por estar descontinuado.
- **Aceitos:** `%pip install -r requirements.txt` com versões pinadas, `environments` do job (serverless), bibliotecas declaradas no job/cluster ou em `pyproject.toml`.

**CICD-03: Compatibilidade com DABs** (🟡)
- Parâmetros de ambiente (catálogo, schema, caminhos) devem chegar via widgets, parâmetros de job ou `argparse`, e não por edição manual do código.
- Com acesso ao repositório: verifique se existe `databricks.yml` com `targets` (dev/prod) e variáveis para catálogo/schema. Sem acesso: **N/V**.

### Passo 7: Anti-padrões de Spark (complementar)

| ID | Padrão | Severidade |
|---|---|---|
| PERF-01 | `inferSchema=True` ou leitura sem schema explícito em pipeline de produção | 🔵 |
| PERF-02 | `coalesce(1)` / `repartition(1)` antes de gravar volumes grandes | 🟡 |
| PERF-03 | `.cache()` / `.persist()` sem `.unpersist()` ou sem reuso real | 🔵 |
| PERF-04 | UDF Python (`@udf`) onde existe função nativa equivalente em `pyspark.sql.functions` | 🟡 |
| PERF-05 | `SELECT *` em tabelas/views da camada Gold | 🔵 |

### Passo 8: Consolidar o relatório

Aplique as exceções válidas (seção 4), calcule o veredito (seção 2) e gere o relatório no formato da seção 5.

---

## 4. Exceções declaradas

O time pode justificar um desvio com um comentário **na mesma linha ou na linha imediatamente anterior**:

```python
pdf = df_resumo.toPandas()  # audit:ignore MEM-01 — agregado mensal, < 100 linhas
```
```sql
-- audit:ignore DATA-03 — tabela de referência, full-refresh diário
INSERT OVERWRITE ${catalog}.silver.ref_moedas SELECT ...
```

- Uma exceção só é válida com **ID da regra + justificativa**. Sem justificativa, ignore a exceção e reporte o achado normalmente.
- **SEC-01 nunca pode ser excepcionada.**
- Toda exceção aceita aparece na seção "Exceções aceitas" do relatório, para dar visibilidade ao revisor.

Exemplos de código ruim e corrigido para as principais regras estão em [`references/exemplos.md`](references/exemplos.md).

---

## 5. Formato do relatório

Use **exatamente** esta estrutura e omita seções vazias (exceto o Resumo e o Checklist):

````markdown
# Relatório de Auditoria Databricks

**Escopo:** <arquivos auditados>
**Veredito:** ❌ Reprovado | ⚠️ Aprovado com ressalvas | ✅ Aprovado

## Resumo
| 🔴 Bloqueador | 🟠 Alto | 🟡 Médio | 🔵 Sugestão |
|---|---|---|---|
| 1 | 2 | 3 | 1 |

## Achados
<!-- ordenados por severidade e depois por arquivo/linha -->

### 🔴 [SEC-01] Credencial em texto claro — `src/ingestao.py:14`
**Evidência:**
```python
token = "dapi****…a1f3"
```
**Por que importa:** <impacto em 1–2 frases>
**Correção:**
```python
token = dbutils.secrets.get(scope="kv-dados", key="databricks-token")
```

## Exceções aceitas
| Regra | Local | Justificativa |
|---|---|---|

## Não verificável
| Regra | Motivo |
|---|---|

## Checklist
| Regra | Status |
|---|---|
| HDR-01 | ✅ / ❌ / N/A / N/V |
| SEC-01 | … |
<!-- uma linha por regra: HDR-01, SEC-01..03, UC-01..03, MEM-01, MOD-01..02, DATA-01..03, CICD-01..03, PERF-01..05 -->
````

**Legenda do checklist:** ✅ conforme · ❌ violação encontrada · N/A não se aplica ao artefato · N/V não verificável com o material disponível.
