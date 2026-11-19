# Exemplos: código ruim e corrigido

Pares de referência para as principais regras. Use-os para calibrar os achados e redigir a seção **Correção** do relatório.

---

## SEC-01: Credencial em texto claro

❌
```python
jdbc_password = "S3nh@Forte2024"
df = (spark.read.format("jdbc")
      .option("url", "jdbc:sqlserver://srv:1433;database=erp")
      .option("user", "etl_user")
      .option("password", jdbc_password)
      .load())
```

✅
```python
jdbc_password = dbutils.secrets.get(scope="kv-erp", key="etl-password")
df = (spark.read.format("jdbc")
      .option("url", "jdbc:sqlserver://srv:1433;database=erp")
      .option("user", dbutils.secrets.get(scope="kv-erp", key="etl-user"))
      .option("password", jdbc_password)
      .load())
```

---

## SEC-02: Exposição de segredo

❌
```python
token = dbutils.secrets.get("kv", "api-token")
print(f"Usando token: {' '.join(token)}")   # contorna a redação [REDACTED]
```

✅
```python
token = dbutils.secrets.get("kv", "api-token")
logger.info("Token de API carregado do scope 'kv'.")
```

---

## UC-01 + UC-02: Endereçamento e parametrização

❌
```python
df = spark.table("vendas")                          # 1 nível
df2 = spark.table("silver.clientes")                # 2 níveis
df.write.saveAsTable("prod.gold.fato_vendas")       # 3 níveis, mas catálogo fixo
```

✅
```python
dbutils.widgets.text("catalog", "dev")
catalog = dbutils.widgets.get("catalog")

df = spark.table(f"{catalog}.bronze.vendas")
df2 = spark.table(f"{catalog}.silver.clientes")
df.write.saveAsTable(f"{catalog}.gold.fato_vendas")
```

✅ SQL (tarefa SQL com parâmetro)
```sql
SELECT * FROM IDENTIFIER(:catalog || '.silver.clientes')
```

✅ Também aceito: contexto parametrizado com `USE`
```python
catalog = dbutils.widgets.get("catalog")
spark.sql(f"USE CATALOG {catalog}")

df = spark.table("silver.clientes")          # 2 níveis, resolve via USE CATALOG
```

```sql
USE CATALOG IDENTIFIER(:catalog);
USE SCHEMA silver;
SELECT * FROM clientes;                      -- 1 nível, contexto explícito
```

❌ Contexto implícito: depende do catálogo padrão do workspace/cluster
```python
df = spark.table("silver.clientes")          # sem USE CATALOG e sem default configurado no job
```

✅ Não é violação: pipeline Lakeflow/DLT (destino definido na configuração do pipeline)
```python
import dlt

@dlt.table
def pedidos_limpos():
    return spark.read.table("pedidos_bronze").filter("valor > 0")   # dataset do próprio pipeline
```

✅ Não é violação (CTE e temp view)
```sql
WITH base AS (SELECT * FROM ${catalog}.silver.pedidos)
SELECT cliente_id, count(*) FROM base GROUP BY cliente_id
```

---

## MEM-01: Coleta no driver

❌
```python
pdf = spark.table(f"{catalog}.silver.transacoes").toPandas()
pdf["valor_ajustado"] = pdf["valor"] * 1.1
```

✅
```python
from pyspark.sql import functions as F
df = (spark.table(f"{catalog}.silver.transacoes")
      .withColumn("valor_ajustado", F.col("valor") * 1.1))
```

✅ Não é violação (escalar após agregação)
```python
max_dt = spark.sql(f"SELECT max(dt_ref) FROM {catalog}.silver.transacoes").collect()[0][0]
```

---

## MOD-01: I/O misturado com transformação

❌
```python
def processar():
    df = spark.table(f"{catalog}.silver.pedidos")
    df = df.filter("status = 'PAGO'").groupBy("cliente_id").sum("valor")
    df.write.mode("overwrite").saveAsTable(f"{catalog}.gold.receita_cliente")
```

✅
```python
def calcular_receita_por_cliente(pedidos: DataFrame) -> DataFrame:
    """Regra de negócio pura: testável sem Spark Session de produção."""
    return (pedidos.filter(F.col("status") == "PAGO")
                   .groupBy("cliente_id")
                   .agg(F.sum("valor").alias("receita")))

# Orquestração (bordas)
pedidos = spark.table(f"{catalog}.silver.pedidos")
receita = calcular_receita_por_cliente(pedidos)
receita.write.mode("overwrite").saveAsTable(f"{catalog}.gold.receita_cliente")
```

---

## DATA-01 / DATA-02: Operações destrutivas

❌
```sql
DELETE FROM ${catalog}.silver.pedidos;
DROP SCHEMA ${catalog}.staging CASCADE;
```

✅
```sql
DELETE FROM ${catalog}.silver.pedidos WHERE dt_ref = :dt_ref;
DROP TABLE IF EXISTS ${catalog}.staging.tmp_pedidos_carga;
```

---

## DATA-03: Sobrescrita total em carga incremental

❌ (cabeçalho declara `Tipo de carga: incremental`)
```python
df_dia.write.mode("overwrite").saveAsTable(f"{catalog}.silver.pedidos")
```

✅ Por partição/período
```python
(df_dia.write.mode("overwrite")
       .option("replaceWhere", f"dt_ref = '{dt_ref}'")
       .saveAsTable(f"{catalog}.silver.pedidos"))
```

✅ Por chave de negócio
```sql
MERGE INTO ${catalog}.silver.pedidos AS t
USING novos_pedidos AS s
  ON t.pedido_id = s.pedido_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

---

## CICD-01: Caminho pessoal

❌
```python
dbutils.notebook.run("/Workspace/Users/joao@empresa.com/etl/limpeza", 600)
```

✅
```python
dbutils.notebook.run("./etl/limpeza", 600)   # relativo, dentro do bundle
```

```yaml
# resources/job.yml
tasks:
  - task_key: limpeza
    notebook_task:
      notebook_path: ../src/etl/limpeza.py
```

---

## CICD-02: Dependências

❌
```python
%pip install pandas
# ... várias células depois ...
%pip install requests
```

✅
```python
# Primeira célula
%pip install -r ../requirements.txt   # versões pinadas (pandas==2.2.3, requests==2.32.3)
dbutils.library.restartPython()
```
