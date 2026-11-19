# Padrões de detecção de segredos (SEC-01)

Use estes padrões como **ponto de partida**. Todo match exige confirmação contextual: placeholders como `"<seu-token>"`, `"xxx"`, `"changeme"` e valores vazios não são violação, mas valem uma 🔵 se estiverem em código de produção.

## Atribuições suspeitas (qualquer linguagem)

Nome de variável/chave sugestivo **com valor literal** (string não vazia, que não seja uma chamada de função):

```regex
(?i)(password|passwd|pwd|senha|secret|token|api[_-]?key|access[_-]?key|client[_-]?secret|private[_-]?key|credential)\s*[:=]\s*["'][^"']{6,}["']
```

## Tokens e chaves com formato conhecido

| Origem | Regex |
|---|---|
| Databricks PAT | `dapi[0-9a-f]{32}(-\d)?` |
| Databricks OAuth secret | `dose[0-9a-f]{32}` |
| AWS Access Key ID | `(AKIA\|ASIA)[0-9A-Z]{16}` |
| AWS Secret Access Key | `(?i)aws_secret_access_key\s*[:=]\s*["']?[A-Za-z0-9/+=]{40}` |
| Azure Storage account key | `(?i)AccountKey=[A-Za-z0-9+/=]{80,}` |
| Azure SAS token | `(?i)[?&]sig=[A-Za-z0-9%+/=]{20,}` |
| GitHub token | `gh[pousr]_[A-Za-z0-9]{36,}` |
| Slack token | `xox[baprs]-[A-Za-z0-9-]{10,}` |
| Chave privada | `-----BEGIN (RSA \|EC \|OPENSSH \|)PRIVATE KEY-----` |
| JWT | `eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}` |

## Padrões específicos de Spark/Databricks

| Padrão | Exemplo de violação |
|---|---|
| Chave de storage em `spark.conf.set` | `spark.conf.set("fs.azure.account.key.conta.dfs.core.windows.net", "abc...")` |
| Client secret OAuth do ADLS | `spark.conf.set("fs.azure.account.oauth2.client.secret...", "abc...")` |
| Credenciais S3 | `spark.conf.set("fs.s3a.secret.key", "abc...")` |
| JDBC com senha na URL | `jdbc:sqlserver://host;user=x;password=abc123` |
| JDBC com `option("password", "...")` | `.option("password", "abc123")` |
| Connection string genérica | `(?i)(Server\|Host)=.+;(Password\|Pwd)=[^;]+` |
| `.databrickscfg` versionado | arquivo com `token = dapi...` |

## Formas corretas (não são violação)

```python
dbutils.secrets.get(scope="kv-dados", key="sql-password")
os.environ["DATABRICKS_TOKEN"]            # variável gerenciada pelo job/CI
spark.conf.set("fs.azure.account.key...", dbutils.secrets.get("kv", "storage-key"))
```

```yaml
# databricks.yml / resources: referência a secret, não ao valor
spark_conf:
  fs.azure.account.key.conta.dfs.core.windows.net: "{{secrets/kv-dados/storage-key}}"
```

## Ao reportar

Mascare sempre o valor: mantenha no máximo o prefixo identificador e os 4 últimos caracteres (`dapi****…a1f3`, `AKIA****…7QXZ`).
