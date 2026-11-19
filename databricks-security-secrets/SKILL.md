---
name: databricks-security-secrets
description: >-
  Remedia e configura o gerenciamento de credenciais, segredos e conexões no
  Databricks: contém vazamentos (revogação e rotação), elimina segredos do
  código usando Unity Catalog (connections, external locations, service
  credentials), configura Secret Scopes com ACLs, autenticação OAuth/Service
  Principal e OIDC no CI/CD, e limpa o histórico Git. Use ao corrigir ou
  remover credenciais do código, tratar token vazado ou exposto, guardar senha
  com segurança, criar ou configurar secret scope, conectar a APIs, bancos
  (JDBC) ou storages externos, ou limpar .env e logs do repositório. Use when
  remediating hardcoded credentials, rotating leaked tokens, configuring
  Databricks Secret Scopes, or setting up secure authentication. Para auditoria
  geral de qualidade de código, use databricks-audit-rules.
---

# Databricks Security & Secrets

Esta skill atua como **Especialista em Remediação de Segredos no Databricks**. Seu propósito é:

- eliminar credenciais em texto claro do código e do histórico Git;
- conter vazamentos;
- levar cada credencial para o mecanismo mais seguro disponível, em ordem de preferência:
  1. **nenhum segredo**: identidade gerenciada ou objeto do Unity Catalog;
  2. **Secret Scope** com ACL mínima;
  3. **variável de ambiente** gerenciada pela plataforma.

> **Regra de ouro:** nunca reproduza um segredo completo em respostas, relatórios, comandos ou diffs. Mascare sempre o valor, mantendo no máximo o prefixo identificador e os 4 últimos caracteres (`dapi****…a1f3`).

---

## 1. Quando usar e escopo

**Acionar em:**
- Escrever ou refatorar código que conecta a APIs externas, bancos (JDBC), cloud storages ou modelos externos.
- Token, senha ou chave exposta em commit, notebook, log ou saída de célula.
- Configurar a autenticação de jobs, Databricks Apps, Databricks SDK ou pipelines de CI/CD.
- Limpar o repositório de `.env`, logs, chaves privadas e `.databrickscfg`.
- Pedidos como *"como guardar esta senha com segurança"*, *"remova este token do código"*, *"configure um secret scope"*, *"vazei um token"*.

**Divisão com outras skills:**
- **`databricks-audit-rules`** *detecta* e classifica violações de segurança (regras SEC-01 a SEC-03) dentro de uma auditoria geral de código. Quando a correção envolver credenciais, ela aponta para esta skill.
- **Esta skill** *remedia*: contenção, rotação, escolha do mecanismo, configuração de scopes e ACLs, refatoração, limpeza do histórico e prevenção.
- **Fora de escopo:** linting e performance (`databricks-audit-rules`) e deploy de infraestrutura em geral (escopo de CI/CD, por exemplo `databricks-dabs-deploy`).

---

## 2. Classificação dos achados

| Classe | Critério | Exemplos |
|---|---|---|
| 🔴 **Incidente ativo** | Credencial real em texto claro no código, em config, na saída de notebook ou no histórico Git | `token = "dapi…"`, `.databrickscfg` versionado, senha na URL JDBC |
| 🟠 **Exposição** | Segredo acessível ou vazável sem estar literal no código | segredo impresso ou logado, ACL de scope ampla demais, segredo passado como parâmetro de job/widget, job de produção rodando com PAT de uma pessoa |
| 🟡 **Higiene** | Ausência de proteção preventiva | `.gitignore` incompleto, sem pre-commit, nome de scope genérico, scope compartilhado entre dev e prod |

**Falsos positivos (não são incidente):**
- placeholders evidentes: `"xxx"`, `"<token>"`, `"changeme"`, `"your-api-key"`, string vazia;
- valores em testes que sejam claramente fictícios.

Em código de produção, placeholders valem uma observação 🟡.

---

## 3. Processo

Execute na ordem. A **contenção vem antes de tudo o que não for detecção**.

### Passo 1: Detectar

1. **Onde procurar:**
   - `.py` e `.sql`, incluindo linhas `# MAGIC` / `-- MAGIC` de notebooks Databricks;
   - `.ipynb`, **inclusive as saídas das células** (`outputs`), onde segredos impressos costumam ficar;
   - `databricks.yml`, `resources/*.yml`, `*.json`, `*.cfg`, `*.ini`, `*.toml` e scripts de CI (`.github/workflows/*.yml`, `azure-pipelines.yml`).
2. **Padrões:**

   | Origem | Padrão |
   |---|---|
   | Databricks PAT | `dapi[0-9a-f]{32}(-\d)?` |
   | Databricks OAuth secret | `dose[0-9a-f]{32}` |
   | AWS | `(AKIA\|ASIA)[0-9A-Z]{16}`; `aws_secret_access_key` com valor de 40 chars |
   | Azure Storage | `AccountKey=[A-Za-z0-9+/=]{80,}`; SAS `[?&]sig=` |
   | `spark.conf.set` com literal | chaves `fs.azure.account.key.*`, `fs.azure.account.oauth2.client.secret.*`, `fs.s3a.secret.key` |
   | GitHub / OpenAI / Slack | `gh[pousr]_[A-Za-z0-9]{36,}` · `sk-(proj-)?[A-Za-z0-9_-]{20,}` · `xox[baprs]-[A-Za-z0-9-]{10,}` |
   | Bearer literal | `Bearer\s+[A-Za-z0-9._-]{20,}` dentro de string (não a palavra "Bearer" sozinha) |
   | JDBC / connection string | `password=`/`pwd=` com valor na URL; `.option("password", "<literal>")` |
   | Atribuição genérica | `(?i)(password\|senha\|secret\|token\|api[_-]?key\|client[_-]?secret)\s*[:=]\s*["'][^"']{6,}["']` |
   | Chave privada | `-----BEGIN (RSA \|EC \|OPENSSH \|)PRIVATE KEY-----` |

3. **Histórico Git:** um segredo removido continua no histórico. Com acesso ao repositório, varra o histórico completo:
   ```bash
   gitleaks detect --source . --log-opts="--all"     # preferencial
   trufflehog git file://. --only-verified           # alternativa
   git log --all -p -S "dapi" -- .                   # mínimo, sem ferramentas
   ```
   Sem acesso ao repositório, marque o histórico como **não verificado** no relatório.

### Passo 2: Conter (obrigatório para todo 🔴)

**Segredo commitado, impresso ou visto em texto claro é segredo comprometido.** Trocá-lo no código por `dbutils.secrets` **não** anula o risco enquanto a credencial antiga estiver ativa.

Para cada credencial 🔴, entregue este checklist:

- [ ] **Revogar** a credencial no provedor de origem:
  - PAT próprio: `databricks tokens delete <token-id>`;
  - PAT de terceiros, como admin: `databricks token-management delete <token-id>`;
  - AWS / Azure / API parceira: no console do provedor.
- [ ] **Investigar uso indevido** durante a janela de exposição (consulta abaixo).
- [ ] **Gerar** a nova credencial, preferindo o mecanismo sem segredo do Passo 3.
- [ ] **Armazenar** o novo valor direto no destino (Passo 4), sem colá-lo em chat, ticket ou terminal com histórico.
- [ ] **Notificar** o time de segurança e registrar o incidente: o que vazou, onde, desde quando, ações tomadas.

Consulta de apoio no `system.access.audit` (ajuste filtros e período ao caso):

```sql
-- Leituras de segredos de um scope no período de exposição
SELECT event_time, user_identity.email, source_ip_address,
       request_params.scope, request_params.key
FROM system.access.audit
WHERE service_name = 'secrets'
  AND action_name = 'getSecret'
  AND request_params.scope = '<scope>'
  AND event_time >= '<inicio_exposicao>'
ORDER BY event_time DESC;

-- Atividade da identidade dona do token vazado, a partir de IPs incomuns
SELECT event_time, service_name, action_name, source_ip_address, user_agent
FROM system.access.audit
WHERE user_identity.email = '<dono_do_token>'
  AND event_time >= '<inicio_exposicao>'
ORDER BY event_time DESC;
```

### Passo 3: Decidir o destino de cada credencial

Antes de criar um secret, pergunte: **dá para não ter segredo nenhum?**

| Caso | Destino preferencial | Fallback |
|---|---|---|
| Acesso a storage (ADLS, S3, GCS) via chave em `spark.conf` | **Storage credential + External location** (ou Volumes) no UC, com identidade gerenciada | Secret Scope referenciado em `spark_conf` |
| Banco externo via JDBC | **UC Connection** (`CREATE CONNECTION`, Lakehouse Federation), com credenciais via `secret()` | `dbutils.secrets` no código |
| SDK de nuvem (boto3, azure-identity) em código | **UC Service Credential** (`dbutils.credentials.getServiceCredentialsProvider`) | Secret Scope |
| Automação contra a API do Databricks (CI/CD, scripts) | **OIDC / workload identity federation** de service principal | OAuth M2M (`client_id` + `client_secret` no cofre do CI) |
| Jobs de produção | **`run_as` service principal** | — (nunca PAT pessoal) |
| API de terceiros sem federação | **Secret Scope** | Variável de ambiente do cluster via `{{secrets/...}}` |

Os exemplos de cada destino estão no Passo 5.

### Passo 4: Armazenar no Secret Scope (quando necessário)

1. **Tipo de scope:** o Databricks tem apenas **dois tipos**:
   - **Databricks-backed:** padrão, disponível em todas as nuvens. Os segredos são gravados com `put-secret`.
   - **Azure Key Vault-backed:** somente na Azure. É **somente leitura** pelo Databricks: os segredos são gravados e rotacionados no Key Vault, e `put-secret` falha.
   - **Não existe scope backed por AWS Secrets Manager.** Na AWS, use um scope Databricks-backed, ou leia o Secrets Manager via boto3 com uma UC Service Credential.
2. **Nomenclatura:** use `<dominio-ou-sistema>-<ambiente>`, como `crm-prd`, `crm-dev`, `pagamentos-prd`. Não use nomes genéricos (`meu-escopo`, `teste`) nem o sufixo redundante `-scope`.
   - Nome da chave: indica o tipo da credencial, como `api-token`, `db-user`, `db-password`, `client-secret`.
   - **Um scope por ambiente**, para que identidades de dev não leiam segredos de prod.
3. **ACL mínima:** o `[REDACTED]` protege apenas a saída do notebook. **Quem tem `READ` no scope consegue extrair o valor.** Por isso:
   - conceda `READ` a **grupos ou service principals**, e não a usuários individuais;
   - restrinja `MANAGE` aos administradores do domínio;
   - revise as ACLs existentes (`list-acls`) e remova acessos amplos, como `users` com `READ` em scope de prod.
4. **Gravação segura:** use `put-secret` **sem** `--string-value`, porque o valor passado na linha de comando fica no histórico do shell. Deixe o CLI pedir o valor de forma interativa, ou leia o valor de um arquivo temporário apagado logo depois.

### Passo 5: Refatorar o código

**Padrão A: `dbutils.secrets` (notebooks e scripts PySpark)**
```python
# ANTES (🔴):
# api_token = "dapi****…a1f3"
# db_password = "Minha****…123"

# DEPOIS:
scope = dbutils.widgets.get("secret_scope")          # ex.: "crm-prd", vindo do job/bundle
api_token = dbutils.secrets.get(scope=scope, key="api-token")
db_password = dbutils.secrets.get(scope=scope, key="db-password")
```
Em SQL, use `secret('crm-prd', 'api-token')`. O usuário precisa de `READ` no scope.

**Padrão B: Databricks SDK e Databricks Apps (autenticação unificada)**
```python
# ANTES (🔴):
# w = WorkspaceClient(host="https://...", token="dapi****…a1f3")

# DEPOIS:
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
```
- **Dentro do Databricks** (notebook, job, App): o SDK usa a identidade do contexto automaticamente.
- **Fora do Databricks** (CI/CD, máquina local): o SDK lê de variáveis de ambiente ou do perfil do CLI.
  - Preferencial, sem segredo: OIDC. Exemplo com GitHub Actions, que exige uma *federation policy* no service principal:
    ```yaml
    permissions:
      id-token: write
    env:
      DATABRICKS_AUTH_TYPE: github-oidc
      DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
      DATABRICKS_CLIENT_ID: ${{ vars.DATABRICKS_SP_CLIENT_ID }}
    ```
  - Fallback: OAuth M2M, com `DATABRICKS_HOST`, `DATABRICKS_CLIENT_ID` e `DATABRICKS_CLIENT_SECRET`. O secret fica no cofre do CI (GitHub Secrets, Azure DevOps Library), nunca no repositório.
- Localmente, use `databricks auth login` (OAuth U2M), e não PAT em arquivo.

**Padrão C: Segredos em jobs e DABs**

A sintaxe `{{secrets/<scope>/<key>}}` é aceita **apenas** em `spark_conf` e `spark_env_vars` de clusters clássicos:

```yaml
# databricks.yml (ou resources/pipeline_crm.yml)
resources:
  jobs:
    pipeline_crm:
      job_clusters:
        - job_cluster_key: main
          new_cluster:
            spark_version: 15.4.x-scala2.12
            node_type_id: Standard_D4ds_v5
            num_workers: 2
            spark_env_vars:
              EXTERNAL_API_KEY: "{{secrets/crm-${bundle.target}/api-token}}"
      tasks:
        - task_key: run_pipeline
          job_cluster_key: main
          spark_python_task:
            python_file: ../src/main.py

targets:
  prd:
    run_as:
      service_principal_name: "<application-id-do-sp>"
```
- O nome `crm-${bundle.target}` só funciona se os targets do bundle tiverem os mesmos nomes usados nos scopes (`dev`, `prd`). Se não tiverem, use uma variável do bundle (`${var.secret_scope}`).
- **Serverless:** não existe `spark_env_vars`. Leia o segredo no código com `dbutils.secrets.get`, recebendo o **nome do scope** como parâmetro.
- **Nunca** passe o *valor* de um segredo como parâmetro de job, widget ou `base_parameters`, porque ele fica visível no histórico de execuções. Passe apenas o nome do scope e da chave.

**Padrão D: Eliminar o segredo com o Unity Catalog**
```sql
-- Storage: troca account key em spark.conf por external location
CREATE EXTERNAL LOCATION IF NOT EXISTS landing_crm
  URL 'abfss://landing@stcrmprd.dfs.core.windows.net/crm'
  WITH (STORAGE CREDENTIAL sc_crm_prd);   -- credential baseada em managed identity / IAM role
GRANT READ FILES ON EXTERNAL LOCATION landing_crm TO `grp-eng-crm`;

-- Banco externo: troca JDBC com senha por UC Connection
CREATE CONNECTION IF NOT EXISTS pg_crm_prd TYPE postgresql
OPTIONS (
  host 'pg-crm.empresa.internal',
  port '5432',
  user secret('crm-prd', 'db-user'),
  password secret('crm-prd', 'db-password')
);
CREATE FOREIGN CATALOG IF NOT EXISTS crm_pg USING CONNECTION pg_crm_prd OPTIONS (database 'crm');
```
```python
# SDK de nuvem: troca access key por UC Service Credential
import boto3
provider = dbutils.credentials.getServiceCredentialsProvider("svc-cred-s3-crm")
s3 = boto3.client("s3", botocore_session=provider, region_name="us-east-1")
```

### Passo 6: Limpar o histórico Git (quando houver 🔴 no histórico)

1. **Só depois da rotação (Passo 2).** A limpeza do histórico reduz a exposição futura, mas **não corrige** um segredo que já vazou.
2. Use `git filter-repo` (preferencial) ou BFG:
   ```bash
   git filter-repo --invert-paths --path .env --path config/.databrickscfg   # remover arquivos
   git filter-repo --replace-text segredos-a-remover.txt                     # substituir valores
   ```
   Apague o `segredos-a-remover.txt` em seguida: ele contém os valores.
3. **Avisos obrigatórios ao desenvolvedor:**
   - a reescrita exige `git push --force` e que todo o time faça clone novo;
   - no GitHub, refs de PRs, forks e caches **continuam com o conteúdo antigo**. Para purgar, é preciso acionar o suporte da plataforma.
   - Coordene com o responsável pelo repositório antes de executar.

### Passo 7: Prevenir

1. **`.gitignore` mínimo:**
   ```gitignore
   # Segredos e variáveis locais
   .env
   .env.*
   !.env.example
   .databrickscfg
   credentials.json
   *.log

   # Chaves e certificados
   *.pem
   *.key
   *.pfx
   *.p12
   id_rsa*

   # Caches e ambientes
   .databricks/
   .venv/
   __pycache__/
   *.pyc
   .pytest_cache/
   ```
2. **Pre-commit com gitleaks:**
   ```yaml
   # .pre-commit-config.yaml
   repos:
     - repo: https://github.com/gitleaks/gitleaks
       rev: v8.21.2
       hooks:
         - id: gitleaks
   ```
   No CI, rode `gitleaks detect` em todo PR.
3. **Notebooks:** limpe as saídas antes do commit (`nbstripout` para `.ipynb`), ou versione em formato source (`.py`).
4. **Produção:** jobs com `run_as` service principal e tempo de vida máximo para PATs definido pelo admin do workspace.

### Passo 8: Gerar o relatório (formato da seção 5)

---

## 4. Anti-racionalizações (o que NÃO tolerar)

| Justificativa ("red flag") | Postura da skill |
|---|---|
| *"O repositório é privado, então o token aqui não é perigoso."* | **INACEITÁVEL.** Repositórios privados têm muitos acessos e vazam por clones locais, logs de CI/CD, forks e integrações de terceiros. |
| *"É só um token de teste pessoal que expira em 7 dias."* | **INACEITÁVEL.** Sete dias com privilégios pessoais bastam para extrair dados do workspace. Use `dbutils.secrets` ou OAuth desde o primeiro dia. |
| *"Já removi o token no commit seguinte."* | **INACEITÁVEL.** O histórico Git guarda a credencial. Rotacione no provedor imediatamente (Passo 2). |
| *"Deixei a credencial num `.env` para testar localmente."* | **PERMITIDO COM RESSALVA.** O `.env` precisa estar no `.gitignore` e nunca pode ser commitado. Em produção, use o Passo 3. |
| *"Não tenho permissão de criar secret scope, então deixei o token provisório."* | **INACEITÁVEL.** Peça ao admin a criação do scope ou do service principal. Código com credencial estática não é promovido. |
| *"O Databricks mostra [REDACTED], então imprimir o segredo é seguro."* | **INACEITÁVEL.** A redação só cobre a saída literal e é contornada com uma transformação simples (`' '.join(s)`). Não imprima nem logue segredos. |
| *"Passei a senha como parâmetro do job, não está no código."* | **INACEITÁVEL.** Parâmetros ficam visíveis no histórico de execuções. Passe o nome do scope, não o valor. |

---

## 5. Formato do relatório

Toda execução termina com este relatório. Omita as seções que não se aplicam, mas mantenha sempre **Status** e **Pendências**.

````markdown
### 🛡️ Relatório de Segurança e Segredos

**Artefatos analisados:** <arquivos / repositório>
**Histórico Git:** varrido com <ferramenta> | não verificado (sem acesso)
**Status:** 🔴 Incidente ativo | 🟠 Exposições encontradas | 🟡 Apenas higiene | ✅ Sem achados

#### 1. Achados
| # | Classe | Local | Tipo | Valor (mascarado) | No histórico? |
|---|---|---|---|---|---|
| 1 | 🔴 | `src/ingestao.py:14` | Databricks PAT | `dapi****…a1f3` | Sim (commit `a1b2c3d`) |

#### 2. Contenção (para cada 🔴)
- [ ] #1: revogar o PAT `…a1f3` (`databricks token-management delete <token-id>`)
- [ ] #1: consultar `system.access.audit` desde <data do commit>
- [ ] #1: gerar a nova credencial e armazená-la conforme a seção 3
- [ ] Notificar o time de segurança

#### 3. Destino de cada credencial
| # | Destino | Justificativa |
|---|---|---|
| 1 | Service principal + `run_as` | Job de produção não deve usar PAT pessoal |

#### 4. Comandos para aplicar
```bash
# Scope e ACL
databricks secrets create-scope crm-prd
databricks secrets put-secret crm-prd api-token        # valor pedido interativamente
databricks secrets put-acl crm-prd grp-eng-crm READ
databricks secrets put-acl crm-prd sp-pipeline-crm READ
databricks secrets list-acls crm-prd

# Scope Azure Key Vault-backed (somente Azure, exige autenticação Microsoft Entra ID)
databricks secrets create-scope crm-prd --json '{
  "scope": "crm-prd",
  "scope_backend_type": "AZURE_KEYVAULT",
  "backend_azure_keyvault": {
    "resource_id": "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.KeyVault/vaults/<kv>",
    "dns_name": "https://<kv>.vault.azure.net/"
  }
}'
```

#### 5. Refatoração
<diff antes/depois de cada trecho, com valores mascarados>

#### 6. Histórico e prevenção
- [ ] Limpar o histórico com `git filter-repo` (após a rotação; coordenar o force push)
- [ ] Atualizar o `.gitignore`
- [ ] Instalar o pre-commit com gitleaks

#### Pendências do desenvolvedor
<ações que só a pessoa ou o admin podem executar: revogação, criação do SP, força-push, aviso à segurança>
````
