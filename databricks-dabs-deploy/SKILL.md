---
name: databricks-dabs-deploy
description: >-
  Cria, valida, parametriza e implanta projetos Databricks via Databricks Asset
  Bundles (DABs): estrutura do databricks.yml, targets dev/staging/prod com
  service principal (run_as), migração de jobs existentes da UI para código
  (bundle generate/bind), pipelines de CI/CD com OIDC e deploy pelo workspace
  (UI ou notebook). Use ao criar ou revisar um databricks.yml, configurar
  ambientes, migrar jobs para bundle, montar esteira de CI/CD (GitHub Actions,
  Azure DevOps, GitLab) ou fazer deploy de bundle — por exemplo "configure o
  bundle deste projeto", "crie o databricks.yml", "migre este job para DABs".
  Use when creating a databricks.yml, configuring multi-environment targets,
  migrating workspace jobs to code, or setting up CI/CD for Databricks bundles.
---

# Databricks Asset Bundles (DABs) & Deploy

Esta skill atua como **Engenheiro de Infraestrutura como Código e CI/CD no Databricks**. Seu propósito é transformar código, notebooks e jobs em pacotes versionados via **Databricks Asset Bundles**, com:

- segregação estrita entre ambientes (*dev*, *staging*, *prod*);
- execução impessoal via **service principals**;
- fim de jobs manuais apontando para pastas pessoais.

> **Regra de execução para o Claude:**
> - `databricks bundle validate` e `databricks bundle summary` podem ser executados livremente.
> - `bundle deploy`, `bundle run` e `bundle destroy` em **dev** só podem ser executados após **confirmação explícita** do usuário na conversa.
> - Em **staging/prod**, **nunca** execute `bundle deploy`, `bundle run` ou `bundle destroy`. Entregue os comandos e a configuração da esteira de CI/CD.
>
> Para reforçar isso no harness, veja a seção 6.

---

## 1. Quando usar e escopo

**Acionar em:**
- Estruturar um novo projeto com `databricks.yml`.
- Migrar jobs criados pela UI ou apontando para pastas pessoais para código versionado.
- Configurar targets com isolamento de catálogos e schemas.
- Montar ou revisar esteiras de CI/CD (GitHub Actions, Azure DevOps, GitLab CI).
- Padronizar governança de jobs: `tags`, `schedule`, `permissions`, `run_as`.
- Fazer deploy pelo workspace (UI ou notebook) em dev/staging.

**Divisão com outras skills:**
- **`databricks-audit-rules`**: qualidade, memória e governança *do código* executado pelos jobs.
- **`databricks-security-secrets`**: criação de secret scopes, ACLs, rotação e remediação de credenciais.
- **Esta skill**: a declaração dos recursos, os ambientes, a identidade de execução e o fluxo de deploy.

---

## 2. Processo

```text
[1 Estrutura] → [2 Targets] → [3 Recursos] → [4 Migração de jobs existentes]
                                                        │
[7 Relatório] ← [6 Deploy controlado] ← [5 Validação] ←─┘
```

### Passo 1: Estrutura do repositório e dependências

1. **Estrutura padrão:**
   ```text
   meu_projeto/
   ├── databricks.yml          # bundle, variáveis, targets e include
   ├── resources/              # um .yml por recurso (jobs, pipelines, schemas)
   │   └── medallion_etl.job.yml
   ├── src/                    # código produtivo (.py, .sql, notebooks finos)
   ├── tests/                  # testes unitários (pytest)
   ├── pyproject.toml          # dependências e build do wheel (opcional)
   └── requirements.txt        # dependências pinadas (==)
   ```
   - Caminhos dentro de `resources/*.yml` são **relativos ao arquivo que os declara**. Por isso, de `resources/` para o código usa-se `../src/...`.
2. **Dependências declaradas no bundle**, e não espalhadas em notebooks:
   - **Serverless:** em `environments[].spec.dependencies`, com versões pinadas, ou apontando para o wheel do projeto.
   - **Cluster clássico:** em `libraries` da task (`pypi` com versão, ou `whl`).
   - `%pip install` solto em notebooks de produção deve ser migrado para o bundle. Um `%pip install -r requirements.txt` pinado é tolerado, mas o bundle é preferível. Mesmo critério da regra CICD-02 da `databricks-audit-rules`.
   - Para código empacotado, use `artifacts` com build do wheel (exemplo na seção 3).

### Passo 2: Variáveis e targets

1. **Variáveis:** centralize em `variables`. Nomes de catálogo e schema **nunca** ficam fixos nos recursos, só nas variáveis por target.
2. **Targets:**
   - **`dev`:**
     - `mode: development`, que coloca o prefixo `[dev <usuário>]` nos nomes, pausa os agendamentos e faz o deploy na pasta do usuário;
     - schema isolado por usuário (`dev_${workspace.current_user.short_name}`);
     - roda com a identidade de quem faz o deploy.
   - **`staging`:**
     - `mode: production`;
     - `run_as` com service principal;
     - gatilhos pausados via `presets.trigger_pause_status: PAUSED`, a menos que o time queira execuções agendadas em homologação.
   - **`prod`:**
     - `mode: production`;
     - `run_as` com service principal;
     - `permissions` corporativas;
     - `git.branch: main`, para que o CLI recuse deploy a partir de outra branch;
     - deploy **somente** pela esteira de CI/CD.
3. **Isolamento de workspace:** cada target aponta para o workspace correto (`workspace.host`). Se dev e prod compartilharem o mesmo workspace, o isolamento passa a depender inteiramente de catálogos diferentes e de `permissions`. Isso deve ficar explícito no relatório.

### Passo 3: Declaração de recursos

Para cada job em `resources.jobs`:
- **Nome:** um nome de negócio. O prefixo de ambiente em dev é aplicado automaticamente pelo `mode: development`. Em staging/prod, inclua o target no nome (`"[${bundle.target}] Pipeline Medallion"`) só se os ambientes compartilharem o mesmo workspace.
- **Tags obrigatórias:** `environment`, `project`, `team`, `criticality`.
- **Tipo de tarefa:** prefira `spark_python_task` / `python_wheel_task` (código modular) ou `notebook_task` (notebooks finos). Passe parâmetros via `${var.catalog}` e `${var.schema}`.
- **Computação:** serverless (`environment_key`) para tarefas padrão. Use `job_clusters` quando houver requisito de cluster clássico, como `spark_env_vars` com secrets, bibliotecas nativas ou instâncias específicas. Nunca use `existing_cluster_id` de cluster interativo em staging/prod.
- **Notificações:** `email_notifications` ou `webhook_notifications` em `on_failure` para staging/prod.
- **Schemas:** se o schema de destino não existir, declare-o em `resources.schemas`, para que o bundle o crie e governe.

### Passo 4: Migração de jobs existentes (UI ou pastas pessoais)

1. **Gerar a definição** a partir do job existente:
   ```bash
   databricks bundle generate job --existing-job-id <job_id> -t dev
   ```
   O comando cria `resources/<nome>.job.yml` e baixa os notebooks e arquivos referenciados para `src/`.
2. **Revisar o YAML gerado:**
   - troque caminhos absolutos (`/Workspace/Users/...`, `/Repos/...`) por caminhos relativos do bundle;
   - troque catálogos e schemas fixos por `${var.*}`;
   - troque `existing_cluster_id` por serverless ou `job_clusters`;
   - adicione tags, `permissions` e notificações.
3. **Vincular ao job original**, para evitar duplicação e preservar o histórico de execuções:
   ```bash
   databricks bundle deployment bind <resource_key> <job_id> -t prod
   ```
   Depois do `bind`, o próximo `deploy` **sobrescreve** o job original com a definição do bundle. Revise o diff antes. No caso de prod, o deploy fica a cargo da esteira.
4. **Regra de caminhos:**
   - as tasks nunca referenciam caminhos absolutos de usuários. O código vem sempre da árvore relativa do bundle (`../src/...`);
   - os caminhos `/Workspace/Users/<usuário-ou-SP>/.bundle/...` criados **pelo próprio deploy** são esperados e não são violação.
   - Isso é coerente com a regra CICD-01 da `databricks-audit-rules`.

### Passo 5: Validação

1. **Validar a sintaxe** e a resolução de variáveis em cada target:
   ```bash
   databricks bundle validate -t dev
   databricks bundle validate -t dev -o json     # inspecionar a configuração resolvida
   ```
2. **Inspecionar a saída JSON:**
   - `${var.catalog}`, `${var.schema}` e `${bundle.target}` substituídos corretamente;
   - caminhos de arquivos resolvidos;
   - `run_as` presente em staging/prod.
3. **Validar staging/prod** com credenciais do respectivo workspace. Normalmente isso acontece **na esteira**, porque o desenvolvedor pode não ter acesso ao workspace de produção.
4. **Revisar o que já está implantado:**
   ```bash
   databricks bundle summary -t dev
   ```
   O `validate` **não** é um dry-run do deploy: ele valida a configuração, mas não mostra o que vai mudar nos recursos.

### Passo 6: Deploy controlado

1. **Ciclo de desenvolvimento** (após confirmação do usuário):
   ```bash
   databricks bundle deploy -t dev
   databricks bundle run -t dev medallion_etl_job
   ```
2. **Staging e produção:** apenas via esteira de CI/CD, após a aprovação do Pull Request (exemplo na seção 4).

### Passo 6.1: Deploy a partir do workspace (UI e notebook)

O deploy pelo workspace serve para **desenvolvimento e homologação** quando o time ainda não tem esteira de CI/CD, ou quando quer iterar sem instalar o CLI localmente. **Produção continua exclusiva da esteira.**

| Via | Identidade do deploy | Uso recomendado |
|---|---|---|
| **UI: bundles no workspace** | Usuário logado | `dev` pessoal, iteração rápida |
| **Notebook com o CLI** | Service principal (OAuth M2M) | `dev`/`staging` compartilhados, sem esteira |
| **CI/CD** (seção 4) | Service principal (de preferência via OIDC) | `staging` e **`prod`** |

#### Opção A: Pela interface (bundles no workspace)

Pré-requisitos:
- o projeto está em uma **Git folder** do workspace, com o `databricks.yml` na raiz;
- o target escolhido aponta para o **mesmo workspace** em que você está (`workspace.host` igual ao atual, ou omitido).

Passo a passo (os nomes dos menus podem variar conforme a versão do workspace):
1. **Workspace → Create → Git folder:** clone o repositório e faça checkout da branch de trabalho.
2. Abra o `databricks.yml`. O workspace reconhece a pasta como bundle e exibe o painel de **Deployments** (ícone de foguete 🚀) na lateral.
3. Selecione o **target** `dev`.
4. Clique em **Validate** e confira as variáveis resolvidas e os caminhos. É o equivalente a `bundle validate`.
5. Clique em **Deploy**. Os recursos são criados em `/Workspace/Users/<você>/.bundle/<bundle>/dev/`, com o prefixo `[dev <usuário>]` nos nomes.
6. Na lista de recursos do painel, clique em **Run** no job, ou abra-o em **Jobs & Pipelines**.
7. Depois de testar, faça commit e push pela Git folder e abra o Pull Request. Staging/prod seguem pela esteira.

> O deploy pela UI usa **a sua identidade**. Isso é adequado para `dev`, mas não para targets com `run_as` de service principal, porque o `run_as` exige a permissão *Service Principal User* sobre o SP. Também não é adequado para produção.

#### Opção B: A partir de um notebook (CLI como operador)

Use quando um ambiente compartilhado (`dev`/`staging`) precisa ser implantado por uma identidade impessoal e ainda não há esteira. O notebook se autentica como **service principal via OAuth M2M**, com as credenciais lidas de um Secret Scope (padrão da `databricks-security-secrets`). Nenhum token é extraído do contexto do notebook.

Pré-requisitos:
- **cluster clássico dedicado** (single user). Serverless não é indicado para instalar e executar binários.
- Secret Scope `cicd-<target>` com `sp-client-id` e `sp-client-secret`, e `READ` concedido apenas ao grupo de operadores.
- O SP de deploy tem permissão no workspace e nos catálogos de destino. Se o target tiver `run_as` com **outro** SP, o SP de deploy precisa da permissão *Service Principal User* sobre ele.

```python
# Databricks notebook source
# =============================================================
# Finalidade : Operador de deploy de DABs (dev/staging). NUNCA prod.
# Identidade : Service principal via OAuth M2M (scope cicd-<target>)
# =============================================================

# COMMAND ----------

# MAGIC %sh
# MAGIC # Instala o Databricks CLI (fixe a versão conforme a política do time)
# MAGIC curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
# MAGIC databricks --version

# COMMAND ----------

import os
import subprocess

dbutils.widgets.dropdown("target", "dev", ["dev", "staging"])
dbutils.widgets.dropdown("acao", "validate", ["validate", "deploy", "run"])
target = dbutils.widgets.get("target")
acao = dbutils.widgets.get("acao")

# Trava de segurança: produção só via CI/CD
if target not in {"dev", "staging"}:
    raise ValueError(f"Target '{target}' não permitido neste notebook. Prod é exclusivo da esteira de CI/CD.")

# Raiz do bundle: este notebook fica em <raiz>/notebooks/ dentro da Git folder
ctx = dbutils.notebook.entry_point.getDbutils().notebook().getContext()
notebook_dir = os.path.dirname(ctx.notebookPath().get())
bundle_root = f"/Workspace{os.path.dirname(notebook_dir)}"   # ajuste se o notebook estiver em outro nível
assert os.path.exists(f"{bundle_root}/databricks.yml"), f"databricks.yml não encontrado em {bundle_root}"

# Autenticação OAuth M2M do service principal (sem PAT, sem token do contexto)
scope = f"cicd-{target}"
env = {
    **os.environ,
    "DATABRICKS_HOST": f"https://{spark.conf.get('spark.databricks.workspaceUrl')}",
    "DATABRICKS_AUTH_TYPE": "oauth-m2m",
    "DATABRICKS_CLIENT_ID": dbutils.secrets.get(scope, "sp-client-id"),
    "DATABRICKS_CLIENT_SECRET": dbutils.secrets.get(scope, "sp-client-secret"),
}
env.pop("DATABRICKS_TOKEN", None)   # evita conflito de métodos de autenticação

def bundle(*args: str) -> str:
    """Executa `databricks bundle ...` na raiz do bundle. Não imprime o env, que contém segredos."""
    cmd = ["databricks", "bundle", *args, "-t", target]
    r = subprocess.run(cmd, cwd=bundle_root, env=env, capture_output=True, text=True)
    print(f"$ {' '.join(cmd)}  (exit {r.returncode})\n{r.stdout}\n{r.stderr}")
    if r.returncode != 0:
        raise RuntimeError(f"Falha em: {' '.join(cmd)}")
    return r.stdout

# COMMAND ----------

# Validate sempre roda primeiro; deploy e run dependem dele
bundle("validate")

if acao in {"deploy", "run"}:
    bundle("deploy")

if acao == "run":
    bundle("run", "medallion_etl_job")
```

Observações:
- Com `mode: development`, a identidade do deploy é o SP. Portanto `${workspace.current_user.short_name}` resolve para o SP, e o schema "pessoal" fica compartilhado. Para sandbox **pessoal**, use a Opção A.
- O código implantado é o da **branch em checkout na Git folder**. Faça *pull* antes de executar.
- Não transforme este notebook em job agendado de deploy automático. Isso é papel da esteira.

### Passo 7: Gerar o relatório (formato da seção 7)

---

## 3. Template de referência

### `databricks.yml`

```yaml
bundle:
  name: medallion_vendas

include:
  - resources/*.yml

# Opcional: empacota src/ como wheel (requer pyproject.toml)
artifacts:
  default:
    type: whl
    path: .
    build: uv build --wheel

variables:
  catalog:
    description: Catálogo do Unity Catalog do ambiente
  schema:
    description: Schema de destino das tabelas
    default: vendas
  notification_email:
    description: Destino dos alertas de falha
    default: dados-alertas@empresa.com

targets:
  dev:
    default: true
    mode: development
    workspace:
      host: https://<workspace-dev>.cloud.databricks.com
    variables:
      catalog: corp_dev
      schema: dev_${workspace.current_user.short_name}

  staging:
    mode: production
    workspace:
      host: https://<workspace-staging>.cloud.databricks.com
    presets:
      trigger_pause_status: PAUSED
    variables:
      catalog: corp_staging
    run_as:
      service_principal_name: "<application-id-sp-staging>"

  prod:
    mode: production
    workspace:
      host: https://<workspace-prod>.cloud.databricks.com
    git:
      branch: main
    variables:
      catalog: corp_prod
    run_as:
      service_principal_name: "<application-id-sp-prod>"
    permissions:
      - level: CAN_MANAGE
        group_name: data_engineering_admins
      - level: CAN_VIEW
        group_name: data_analysts
```

### `resources/medallion_etl.job.yml`

```yaml
resources:
  jobs:
    medallion_etl_job:
      name: Pipeline Medallion Vendas

      tags:
        environment: ${bundle.target}
        project: medallion_vendas
        team: data_engineering
        criticality: tier_1

      schedule:
        quartz_cron_expression: "0 0 4 * * ?"
        timezone_id: America/Sao_Paulo
        pause_status: UNPAUSED        # dev: pausado pelo mode; staging: pausado pelo preset

      email_notifications:
        on_failure:
          - ${var.notification_email}

      tasks:
        - task_key: run_etl
          environment_key: serverless_env
          spark_python_task:
            python_file: ../src/main.py
            parameters:
              - --catalog
              - ${var.catalog}
              - --schema
              - ${var.schema}

      environments:
        - environment_key: serverless_env
          spec:
            environment_version: "2"   # em versões antigas do CLI/API o campo era `client`
            dependencies:
              - ../dist/*.whl          # wheel gerado em `artifacts`; ou pacotes pinados, ex.: requests==2.32.3
```

---

## 4. CI/CD de referência (GitHub Actions + OIDC)

Pré-requisitos:
- service principals de staging e prod com **federation policy** para o repositório e o environment do GitHub. Sem client secret.
- **Environments** `staging` e `prod` no GitHub, com *required reviewers* em `prod`.
- Variáveis `DATABRICKS_HOST` e `DATABRICKS_CLIENT_ID` definidas em cada environment.

```yaml
# .github/workflows/bundle.yml
name: databricks-bundle

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write      # necessário para o OIDC
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    environment: staging
    env:
      DATABRICKS_AUTH_TYPE: github-oidc
      DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
      DATABRICKS_CLIENT_ID: ${{ vars.DATABRICKS_CLIENT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main        # fixe uma tag ou SHA conforme a política do time
      - run: pip install pytest && pytest tests/
      - run: databricks bundle validate -t staging

  deploy-staging:
    if: github.event_name == 'push'
    needs: validate
    runs-on: ubuntu-latest
    environment: staging
    env:
      DATABRICKS_AUTH_TYPE: github-oidc
      DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
      DATABRICKS_CLIENT_ID: ${{ vars.DATABRICKS_CLIENT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
      - run: databricks bundle deploy -t staging

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: prod                         # aprovação manual via required reviewers
    env:
      DATABRICKS_AUTH_TYPE: github-oidc
      DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
      DATABRICKS_CLIENT_ID: ${{ vars.DATABRICKS_CLIENT_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
      - run: databricks bundle validate -t prod
      - run: databricks bundle deploy -t prod
```

Sem OIDC, use OAuth M2M: `DATABRICKS_CLIENT_SECRET` como *secret* do environment. Nunca use PAT pessoal. No Azure DevOps e no GitLab, a estrutura é a mesma: validate no PR, deploy de staging no merge e deploy de prod com aprovação manual.

---

## 5. Anti-racionalizações (o que NÃO tolerar)

| Argumento ("red flag") | Postura da skill |
|---|---|
| *"Vou apontar a task para o notebook em `/Workspace/Users/meu_usuario/` para testar mais rápido."* | **INACEITÁVEL.** Isso quebra o versionamento e o job falha quando o usuário sai da empresa. Use o target `dev` do bundle, que é igualmente rápido. |
| *"Não preciso de `run_as` com service principal em produção, meu usuário é admin."* | **INACEITÁVEL.** Staging e prod rodam de forma impessoal via SP, o que garante auditoria e evita falhas por desligamento ou expiração de credencial individual. |
| *"Posso rodar `bundle deploy -t prod` do meu computador."* | **INACEITÁVEL.** Prod só pela esteira, a partir da `main` (reforçado por `git.branch`) e após PR aprovado. |
| *"Vou fazer o deploy de prod pelo botão da UI ou pelo notebook de deploy, é mais rápido."* | **INACEITÁVEL.** A UI usa a identidade pessoal, e o notebook implanta o que estiver em checkout, sem PR aprovado. |
| *"Não usei variáveis porque o nome do catálogo nunca vai mudar."* | **INACEITÁVEL.** Nomes fixos impedem a promoção dev → staging → prod e arriscam gravar em tabelas de produção a partir de dev. |
| *"Deixei `mode: development` em prod para facilitar a depuração."* | **INACEITÁVEL.** O modo de desenvolvimento pausa agendamentos, prefixa os nomes e implanta na pasta do usuário. Use `mode: production`. |
| *"Fiz o `bind` e já dei deploy direto, o YAML gerado deve estar certo."* | **INACEITÁVEL.** Depois do `bind`, o deploy sobrescreve o job original. Revise o YAML gerado (Passo 4) e valide primeiro em dev. |

---

## 6. Reforço no harness (opcional, recomendado)

A regra de execução do topo é uma instrução. Para que o próprio Claude Code peça confirmação ou bloqueie comandos, adicione permissões ao `.claude/settings.json` do projeto:

```json
{
  "permissions": {
    "allow": [
      "Bash(databricks bundle validate:*)",
      "Bash(databricks bundle summary:*)"
    ],
    "ask": [
      "Bash(databricks bundle deploy:*)",
      "Bash(databricks bundle run:*)",
      "Bash(databricks bundle destroy:*)",
      "Bash(databricks bundle deployment bind:*)"
    ],
    "deny": [
      "Bash(databricks bundle deploy -t prod:*)",
      "Bash(databricks bundle destroy -t prod:*)"
    ]
  }
}
```

- No Windows com PowerShell, replique as regras com o prefixo `PowerShell(...)`.
- As regras casam por prefixo, então variações como `--target prod` não são pegas pelo `deny`. Trate isso como uma proteção adicional, e não como barreira de segurança: a barreira real é o SP de prod só existir na esteira.

---

## 7. Relatório e checklist de prontidão

Toda sessão termina com:

````markdown
### 📦 Relatório do Bundle

**Bundle:** <nome> · **Targets:** dev, staging, prod
**Validação:** `bundle validate -t dev` ✅ exit 0 | ❌ <erro>

#### Checklist
- [ ] `databricks.yml` na raiz, com `include: resources/*.yml`
- [ ] `dev` com `mode: development` e schema `dev_${workspace.current_user.short_name}`
- [ ] `staging`/`prod` com `mode: production` e `run_as` de service principal
- [ ] `prod` com `git.branch: main` e `permissions` por grupo
- [ ] Nenhum catálogo/schema fixo nos recursos (só em `variables` por target)
- [ ] Tasks com caminhos relativos do bundle (sem `/Workspace/Users/...` ou `/Repos/...`)
- [ ] Sem `existing_cluster_id` em staging/prod
- [ ] Dependências pinadas no bundle (`environments`/`libraries`/`artifacts`)
- [ ] Tags `environment`, `project`, `team`, `criticality` e notificação `on_failure`
- [ ] Esteira de CI/CD com validate no PR e aprovação manual para prod
- [ ] Jobs migrados vinculados com `bundle deployment bind` (se aplicável)

#### Pendências
<o que depende do usuário ou do admin: criar SPs, federation policy, environments do GitHub, permissões>

#### Próximos comandos
```bash
databricks bundle validate -t dev
databricks bundle deploy -t dev          # após confirmação
databricks bundle run -t dev medallion_etl_job
```
````
