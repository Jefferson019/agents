---
name: "databricks-unity-gateway"
description: "Configura agentes de código (Claude Code, Codex, OpenCode, Copilot, Gemini CLI, Cursor, Pi) no Unity Gateway da Databricks via CLI ug, com pré-requisitos passo a passo. Use ao citar ug, ucode, unity-gateway, /ai-gateway, system.ai ou erros 403."
---

# databricks-unity-gateway: agentes de código via Unity Gateway da Databricks

Esta skill configura **harnesses** de código (o programa que roda o agente: Claude Code, Codex, OpenCode etc.) para usar modelos, MCPs e skills governados pelo **Unity Gateway** da Databricks.

Para escolher o modo (pessoal, empresa ou assinatura via gateway), ver o que é permitido com cada assinatura e configurar o uso pessoal, use a skill **coding-agent-setup**.

Última verificação: 2026-11-26. Referências: `ug` 0.2.x, Gemini CLI < 0.45.0, Copilot CLI ≥ 1.0.91.

> O `ug` está na versão 0.x e muda rápido. Se a versão instalada (`ug --version`) for bem mais nova que a da data acima, avise que a receita pode estar desatualizada e confira cada comando com `ug <cmd> --help` antes de aplicar.

---

## 0. Modelo mental (leia antes de tudo)

- **Unity Gateway (serviço, GA)**: é o control plane de IA da Databricks sobre o Unity Catalog. Ele governa *model services* (LLMs), *model provider services* (OpenAI, Anthropic, Bedrock, assinaturas), *MCP services* e *skills*. As permissões usam os privilégios do UC, como `EXECUTE`. O gateway aplica rate limits, budgets e guardrails e registra o uso em `system.ai_gateway.usage`.
- **`ug` (CLI, repositório `databricks/unity-gateway`, antigo `ucode`)**: roda **só na máquina do usuário**. Não há servidor para subir, nem Docker ou App. O `ug` faz quatro coisas:
  1. autentica no workspace (OAuth via Databricks CLI);
  2. descobre os modelos;
  3. grava a configuração de cada agente (com backup);
  4. sobe um proxy local que renova o token a cada requisição.
- **Nomes de modelo têm sempre três partes**: `<catalog>.<schema>.<modelo>`, por exemplo `system.ai.claude-sonnet-4-6`.
- **Superfícies de API do gateway** (`https://<workspace-host>/ai-gateway/...`):

| Superfície | Caminho | Protocolo | Quem usa |
|---|---|---|---|
| Anthropic | `/ai-gateway/anthropic` (SDKs: `/anthropic/v1`) | Anthropic Messages | Claude Code, Claude desktop, OpenCode, Pi |
| Codex | `/ai-gateway/codex/v1` | OpenAI Responses | Codex, ChatGPT desktop, Pi |
| Gemini | `/ai-gateway/gemini` (SDKs: `/v1beta`) | Google GenAI | Gemini CLI, OpenCode, Pi |
| OpenAI-compatível | `/ai-gateway/mlflow/v1` | Chat Completions / Responses / Embeddings | Copilot (manual), SDKs, clientes genéricos |
| Cursor | `/ai-gateway/cursor/v1` | override OpenAI | Cursor IDE |
| MCP | `/ai-gateway/mcp-services/<c>.<s>.<svc>?o=<workspace-id>` | MCP streamable HTTP | todos os clientes MCP |
| Skills | `/ai-gateway/skills/?schema=<c>.<s>` (mantenha a barra final) | MCP | todos os clientes MCP |

---

## 1. Regras fixas

1. **Prefira sempre `ug <agente>`.** A configuração manual é só fallback.
2. **Confira flags com `--help`.** O `ug` já mudou flags entre versões: `--workspaces` virou `--workspace`, `ug configure mcp` virou `ug mcp add`, `ug configure skills` virou `ug skills add`.
3. **Nunca invente nome de modelo.** Liste com `/model` (ou `/models`) dentro do agente, com `ug status` ou pela UI do Unity Gateway.
4. **Use placeholders e nunca valores reais**: `<workspace-host>`, `<workspace-id>`, `<databricks-pat>`, `<perfil>`.
5. **Segredo não vai para o git.** Arquivos como `~/.claude/settings.json`, `~/.codex/config.toml`, `opencode.json`, `~/.cursor/mcp.json`, `~/.databrickscfg` e `~/.ucode/` são privados. Em máquina compartilhada, rode `chmod -R go-rwx ~/.ucode`: a issue #1544 relata o JWT OAuth gravado como legível por todos.
6. **Edite com merge.** Antes de editar à mão, faça backup (`cp arq arq.bak.$(date +%F)`).
7. **Dados da empresa** (código, dados de clientes ou beneficiários, LGPD) só passam pelos modelos governados pela empresa no gateway.
8. **Não declare sucesso antes da verificação da seção 7.**

---

## 2. Pré-requisitos, com passo a passo para obter cada um

Cada item segue o formato **o que é → como verificar → como obter**. Itens marcados **[ADMIN]** dependem de um admin do workspace ou da conta. Se o usuário não for admin, gere o pedido da seção 2.3.

### 2.1 Do lado Databricks

**P1. Workspace com Unity Catalog e Unity Gateway disponível** [ADMIN]
- **Verificar:** no workspace, o menu lateral tem a área "AI Gateway / Unity Gateway", e o Catalog Explorer mostra o catálogo `system` com o schema `system.ai`.
- **Obter:** o admin de conta habilita o Unity Catalog no workspace. Alguns recursos dependem de preview no **account console > Previews**:
  - os MCPs `system.ai.dbsql`, `system.ai.sandbox` e `system.ai.web_search` exigem "Enhanced Unity Gateway";
  - as skills do UC são Beta;
  - o modo assinatura Claude (Beta) depende de preview.

**P2. Entitlement do usuário** [ADMIN]
- **O que é:** o usuário precisa de *Workspace access* ou *Consumer access*. Com só *Consumer access*, alguns MCPs (Vector Search, UC Functions, Genie, Apps) não funcionam.
- **Verificar:** `databricks current-user me --profile <perfil>` (veja os `entitlements` no JSON) ou peça ao admin.
- **Obter:** o admin vai em **Settings > Identity and access > Users (ou Groups) > <usuário/grupo> > Entitlements** e marca *Workspace access*. Prefira conceder ao grupo.

**P3. Acesso aos model services**
- **O que é:**
  - Para consultar um model service: `EXECUTE` no serviço, mais `USE CATALOG` e `USE SCHEMA` no catálogo e no schema dele.
  - **Por padrão, todos os usuários da conta já têm `EXECUTE` nos model services fornecidos pela Databricks (`system.ai`)**, então em geral não há nada a pedir para eles.
  - Grant de Model Serving (`CAN_QUERY`) e permissão sobre o modelo **não** dão acesso ao model service.
- **Verificar:** `ug status` (depois da seção 2.2) lista os modelos visíveis. Também dá para abrir o serviço no Catalog Explorer, ou fazer o teste de chamada do passo M8.
- **Obter, para serviços fora de `system.ai`** [ADMIN, exige `MANAGE` no serviço/schema/catálogo]:
  1. No Catalog Explorer, abra o model service, vá à aba **Permissions**, clique em **Grant**, escolha o grupo e marque `EXECUTE`.
  2. Conceda o acesso ao catálogo e ao schema (SQL padrão do UC):
     ```sql
     GRANT USE CATALOG ON CATALOG <catalog> TO `<grupo>`;
     GRANT USE SCHEMA  ON SCHEMA  <catalog>.<schema> TO `<grupo>`;
     ```
  3. Mudanças de privilégio levam **alguns minutos** para valer.

**P4. (Opcional) Model provider service para BYOK ou para assinatura** [ADMIN]
- **Quando:** só se for usar `--provider`, isto é, chave própria da empresa num provedor externo, ou o modo assinatura da seção 5.
- **Obter:**
  1. Quem cria precisa de `CREATE SERVICE`, `USE CATALOG` e `USE SCHEMA` no schema de destino.
  2. Crie o model provider service pela UI do Unity Gateway:
     - **BYOK:** guarde a chave num secret do UC.
     - **Assinatura Claude:** escolha o modo de autenticação "Claude Max/Enterprise". O serviço não guarda credencial nenhuma.
     - **Assinatura ChatGPT:** escolha o modo equivalente.
  3. Conceda `EXECUTE` no provider service, mais `USE CATALOG`/`USE SCHEMA`, ao grupo que vai usá-lo.

**P5. (Opcional) Acesso a MCP services** [ADMIN]
- **O que é:** `EXECUTE` no MCP service, por exemplo `system.ai.dbsql`, mais o preview do P1 quando for o caso. MCPs de terceiros (GitHub, Slack) pedem também um login próprio no provedor.

**P6. (Opcional) Configuração gerenciada pelo admin** [ADMIN]
- **O que é:** o admin publica em **Unity Gateway > Govern > Agent configuration** quais agentes ficam habilitados, qual é o agente padrão, os modelos por agente, os MCPs (`mcp_servers.sources`) e as skills.
- **Efeito para o usuário:**
  - `ug` sem argumentos abre o agente padrão.
  - Agentes não habilitados exigem `ug agents add <agente>`.
  - `ug export` mostra a configuração recebida.

### 2.2 Na máquina do usuário

**M1. Descobrir o host e o workspace ID**
- **Host:** é o endereço do workspace no navegador, por exemplo `https://<algo>.cloud.databricks.com` (AWS), `https://adb-<id>.<n>.azuredatabricks.net` (Azure) ou `https://<id>.<n>.gcp.databricks.com` (GCP).
- **Workspace ID:** é o número depois de `?o=` na URL do workspace. No Azure também aparece no host (`adb-<id>`). Depois de configurar, `ug status` também mostra.

**M2. Instalar o `uv` e o Python 3.12+**
```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

uv --version
uv python install 3.12      # se não houver Python 3.12+
```

**M3. Instalar a Databricks CLI (versão nova, ≥ 0.2xx; não a antiga `databricks-cli` do pip)**
```bash
# macOS (Homebrew)
brew tap databricks/tap && brew install databricks
# Linux / macOS (script oficial)
curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
# Windows
winget install Databricks.DatabricksCLI

databricks --version
```

**M4. Login OAuth (o caminho recomendado)**
```bash
databricks auth login --host https://<workspace-host> --profile <perfil>
databricks auth profiles                       # o perfil deve aparecer como válido
databricks current-user me --profile <perfil>  # confirma identidade e entitlements
```
Os tokens OAuth expiram em cerca de 1 hora. O `ug` renova sozinho, então **não** copie esse token para arquivos de configuração.

**M5. (Só para CI/headless ou config manual) PAT com escopo `ai-gateway`**
1. No workspace, clique no seu usuário e vá a **Settings > Developer > Access tokens > Manage > Generate new token**.
2. Escolha o escopo **`ai-gateway`** (mínimo privilégio), uma validade curta e um comentário identificando o uso. Se a tela de escopos estiver diferente, consulte a doc "Personal access tokens" da Databricks.
3. Salve num perfil:
   ```bash
   databricks configure --profile <perfil-pat>   # pede o host e o token
   ```
4. Cuidado: o host regional legado `*.ai-gateway.*` exige `all-apis`. Com o escopo `ai-gateway` nele, a resposta é `403: required scopes: all-apis`. Use sempre `https://<workspace-host>/ai-gateway/...`.

**M6. Instalar o Node.js LTS e o npm** (vários agentes são distribuídos via npm)
- Instale pelo site oficial, pelo `nvm` ou pelo gerenciador do sistema, e confira com `node -v && npm -v`.

**M7. Instalar o `ug` e diagnosticar**
```bash
uv tool install git+https://github.com/databricks/unity-gateway
ug --version          # anote o hash g<commit> para bug reports
ug doctor             # verifica uv, npm, Databricks CLI, workspace, credenciais e CLIs dos agentes; oferece correções
```
- Para atualizar: `ug upgrade` (ou `uv tool upgrade unity-gateway`).
- **Não** use o pacote `unity-gateway` do PyPI: é um placeholder antigo.
- O comando antigo `ucode` ainda funciona como alias.

**M8. Teste de acesso ao gateway, antes de configurar qualquer agente**
```bash
TOKEN=$(databricks auth token --profile <perfil> | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
curl -s https://<workspace-host>/ai-gateway/mlflow/v1/chat/completions \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"system.ai.<modelo>","messages":[{"role":"user","content":"diga ok"}],"max_tokens":10}'
```
Como interpretar o resultado:
- **Resposta com texto:** os pré-requisitos estão OK.
- **403 ou `PERMISSION_DENIED`:** problema em P2/P3.
- **Modelo inexistente:** confira o nome com `ug status` ou na UI.

**M9. Instalar o agente desejado**
O `ug` oferece instalar o agente quando ele falta. Para instalar à mão, confira o comando na doc oficial de cada um:

| Agente | Instalação típica | Versão exigida |
|---|---|---|
| Claude Code | `curl -fsSL https://claude.ai/install.sh \| bash` | atual |
| Codex | `npm install -g @openai/codex` | Windows desktop: ≥ 0.156.0 |
| OpenCode | `curl -fsSL https://opencode.ai/install \| bash` | atual |
| Copilot CLI | `npm install -g @github/copilot` | ≥ 1.0.91 |
| Copilot no VS Code | VS Code + comando `code` no PATH | VS Code ≥ 1.128 (MCP: ≥ 1.142) |
| Gemini CLI | `npm install -g "@google/gemini-cli@<0.45.0"` | < 0.45.0 |
| Pi, Kilo, Antigravity | deixe o `ug` instalar ou veja a doc do agente | — |

### 2.3 Modelo de pedido ao admin

Quando faltar um item [ADMIN], gere este pedido preenchido:

```
Assunto: Acesso ao Unity Gateway para agentes de código

Workspace: https://<workspace-host>
Usuário/grupo: <email ou grupo>
Agente(s): <Claude Code / Codex / OpenCode ...>

Preciso de:
[ ] Entitlement "Workspace access" (P2)
[ ] EXECUTE no(s) model service(s): <c>.<s>.<serviço>  + USE CATALOG/USE SCHEMA (P3)
[ ] EXECUTE no(s) MCP service(s): <c>.<s>.<mcp> (P5)  — e preview "Enhanced Unity Gateway" se for dbsql/sandbox/web_search
[ ] (opcional) model provider service <c>.<s>.<nome> para BYOK/assinatura + EXECUTE (P4)
[ ] (opcional) habilitar o agente <x> no Agent configuration (P6)

Erro observado (se houver): <mensagem exata>
```

---

## 3. Configurar o `ug`

```bash
ug configure                                                    # picker interativo (agentes + workspace)
ug configure --workspace https://<workspace-host> --agents claude,codex,opencode --user
ug configure --profile <perfil> --agents claude,codex --user    # usa um perfil existente da Databricks CLI
ug configure --profile <perfil-pat> --agents claude --use-pat --skip-validate   # CI/headless
ug configure --dry-run                                          # mostra o que seria escrito
ug configure --agent claude --model-location <catalog>.<schema> # modelos fora de system.ai
```

- **`--user` (ou `UG_SKIP_OS_SETTINGS=1`):** sem ele, o `ug` grava settings de máquina via **sudo** para Claude Code e Codex (`/etc/claude-code/managed-settings.json`, `/Library/Application Support/ClaudeCode/managed-settings.json`, `/etc/codex/managed_config.toml`). Com isso, o `claude` e o `codex` puros também passam a usar o gateway, e o sudo é pedido a cada execução (issues #1078 e #1282). Use `--user`, a menos que a empresa queira forçar o gateway na máquina inteira.
- Para usar um agente que o admin não habilitou: `ug agents add <agente>`.
- Comandos de gestão: `ug status`, `ug doctor`, `ug usage` (`--all`), `ug web`, `ug export`, `ug revert` (restaura os backups) e `ug upgrade`.
- Flags de lançamento:
  - `--refresh` re-checa o workspace, os modelos e a config.
  - `--model <c>.<s>.<m>` escolhe o modelo.
  - `--enable-smart-routing` vale para Claude Code e Codex, só naquela execução.
  - `--header "Nome: valor"` adiciona um header (repetível).
  - Argumentos depois de `--` vão direto para o agente.

---

## 4. Receitas por harness

Esqueleto de cada receita: **via `ug` (recomendado) → manual (fallback, PAT `ai-gateway`) → verificação → problemas conhecidos.** O rollback é sempre `ug revert` ou o `.bak`.

Matriz de suporte (docs oficiais, nov/2026):

| Agente | Comando | Modelos Databricks | `--provider` | MCP | Skills |
|---|---|---|---|---|---|
| Claude Code | `ug claude` | ✅ | ✅ | ✅ | ✅ |
| Claude desktop | `ug configure --agents claude` | ✅ | ✅ | ✅ | ✅ |
| Codex | `ug codex` | ✅ | ✅ | ✅ | ✅ |
| ChatGPT desktop | `ug configure --agents codex` | ✅ | ✅ | ✅ | ✅ |
| Copilot CLI / VS Code | `ug copilot` / `ug configure --agents copilot` | ✅ | ✅ | ✅ | ✅ |
| Gemini CLI | `ug gemini` | ✅ | Gemini Enterprise | ✅ | ✅ |
| OpenCode | `ug opencode` | ✅ | ❌ (#608) | ✅ | ✅ |
| Pi | `ug pi` | ✅ | — | ❌ | ✅ |
| Cursor IDE | manual + `ug mcp add --agents cursor` | ✅ manual | — | ✅ | ✅ |
| Cursor Agent CLI | `ug cursor` (só MCP) | ❌ | ❌ | ✅ | ✅ |
| Kilo, Antigravity | `ug kilo`, `ug antigravity` | só no repositório, sem doc oficial | | | |

### 4.1 Claude Code (e Claude desktop)

**Via `ug`:**
```bash
ug claude                                  # no diretório do projeto; troque o modelo com /model
ug claude -- --permission-mode auto
```
- Arquivos usados: `~/.claude/ucode-settings.json` e `~/.claude.json`, além dos managed settings se o `--user` não tiver sido usado.
- `ug configure --agents claude` também cria e seleciona um perfil no **Claude desktop**.

**Manual:** faça merge em `~/.claude/settings.json`. Isso afeta o `claude` puro.
```json
{
  "env": {
    "ANTHROPIC_MODEL": "<catalog>.<schema>.<modelo-claude>",
    "ANTHROPIC_BASE_URL": "https://<workspace-host>/ai-gateway/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "<databricks-pat>",
    "ANTHROPIC_CUSTOM_HEADERS": "x-databricks-use-coding-agent-mode: true",
    "CLAUDE_CODE_USE_GATEWAY": "1",
    "ENABLE_PROMPT_CACHING_1H": "1",
    "ENABLE_TOOL_SEARCH": "true"
  }
}
```
- O modelo precisa suportar a Anthropic Messages API: Claude ou um open-source compatível. GPT e gpt-oss não funcionam aqui.
- `ENABLE_PROMPT_CACHING_1H` e `ENABLE_TOOL_SEARCH` reduzem custo.
- Claude desktop manual:
  1. Help > Troubleshooting > Enable Developer Mode.
  2. Developer > Configure Third-Party Inference > Gateway, com a base URL `https://<workspace-host>/ai-gateway/anthropic` e o scope `ai-gateway`.
  3. Clique em Test connection.

**Problemas conhecidos:**
- Seletor de *effort* sumido no desktop (issue #1926): adicione `behavesAs` ao perfil em `~/Library/Application Support/Claude-3P/configLibrary/<id>.json`.
- No auto mode com modelos não Anthropic, o classificador do gateway gera custo extra.

### 4.2 Codex (e ChatGPT desktop)

**Via `ug`:** `ug codex` (ou `ug codex --full-auto`).
- Arquivos: `~/.codex/ucode.config.toml` e `~/.ucode/codex-model-catalog.json`.
- `ug configure --agents codex` também configura o ChatGPT desktop. No Windows, exige terminal elevado e Codex ≥ 0.156.0.

**Manual:** em `~/.codex/config.toml`, coloque `model` e `model_provider` **no topo**, antes de qualquer tabela.
```toml
model = "<catalog>.<schema>.<modelo>"
model_provider = "databricks"

[model_providers.databricks]
name = "Databricks"
base_url = "https://<workspace-host>/ai-gateway/codex/v1"
wire_api = "responses"
requires_openai_auth = false
supports_websockets = false
http_headers = { Authorization = "Bearer <databricks-pat>" }
```

**Problemas conhecidos:**
- Erro de WebSocket: falta `supports_websockets = false`.
- `--provider` "grudado" depois de usado (issue #602): rode `ug revert` e reconfigure.
- Smart routing pode pedir uma revisão única dos hooks em `/hooks`.

### 4.3 OpenCode

**Via `ug`:** `ug opencode`.
- O `ug` regenera `~/.ucode/opencode-xdg/opencode/opencode.json` e o plugin `ucode-auth.js` **a cada execução**, então não edite esse arquivo.
- Para ajustes persistentes (por exemplo, o contexto de ~1M que aparece como 200K, issue #905), use um `opencode.jsonc` ao lado dele.

**Manual:** faça merge em `~/.config/opencode/opencode.json`.
```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "databricks-anthropic": {
      "npm": "@ai-sdk/anthropic",
      "name": "Databricks (Anthropic)",
      "options": {
        "baseURL": "https://<workspace-host>/ai-gateway/anthropic/v1",
        "apiKey": "{env:DATABRICKS_TOKEN}",
        "headers": { "Authorization": "Bearer {env:DATABRICKS_TOKEN}" }
      },
      "models": {
        "<catalog>.<schema>.<modelo-claude>": { "name": "<modelo-claude>", "toolStreaming": false }
      }
    },
    "databricks-gemini": {
      "npm": "@ai-sdk/google",
      "options": {
        "baseURL": "https://<workspace-host>/ai-gateway/gemini/v1beta",
        "apiKey": "{env:DATABRICKS_TOKEN}"
      },
      "models": { "<catalog>.<schema>.<modelo-gemini>": {} }
    }
  },
  "mcp": {
    "databricks-sql": {
      "type": "remote",
      "url": "https://<workspace-host>/ai-gateway/mcp-services/system.ai.dbsql?o=<workspace-id>",
      "oauth": false,
      "headers": { "Authorization": "Bearer {env:DATABRICKS_TOKEN}" }
    }
  }
}
```
- `toolStreaming: false` é **obrigatório** em cada modelo.
- Modelos GPT via gateway no OpenCode **não têm receita oficial**. Uma hipótese não verificada é `@ai-sdk/openai-compatible` com `/ai-gateway/mlflow/v1`; teste antes de adotar.
- O `--provider` (BYOK ou assinatura) não é suportado (issue #608).

### 4.4 GitHub Copilot (CLI e VS Code)

**Via `ug`:**
- `ug copilot` exige o Copilot CLI ≥ 1.0.91. Rodar `copilot` puro não carrega os modelos do gateway.
- `ug configure --agents copilot` instala a extensão **Databricks Unity Gateway** no VS Code. Os modelos aparecem no grupo "Databricks" do Copilot Chat. Use `--no-vscode` para configurar só o CLI.
- Gemini e gpt-oss ficam excluídos.

**Manual (só CLI):**
```bash
export COPILOT_PROVIDER_TYPE="openai"
export COPILOT_PROVIDER_BASE_URL="https://<workspace-host>/ai-gateway/mlflow/v1"
export COPILOT_PROVIDER_BEARER_TOKEN="<databricks-pat>"
export COPILOT_MODEL="<catalog>.<schema>.<modelo>"
export COPILOT_OFFLINE="true"
copilot
```
- MCP: `~/.copilot/mcp-config.json`. Skills: `~/.agents/skills/`, recarregadas com `/skills reload`.
- **Problema conhecido:** modelos OpenAI retornam "Unknown parameter: 'snippy'" (issue #855).

### 4.5 Gemini CLI

**Via `ug`:** `ug gemini`. O `ug` usa um HOME isolado (`~/.ucode/.gemini-home/.gemini/settings.json`), então editar `~/.gemini` não afeta o `ug gemini`.

**Manual** (só funciona com versões < 0.45.0):
```bash
export GEMINI_MODEL="<catalog>.<schema>.<modelo-gemini>"
export GOOGLE_GEMINI_BASE_URL="https://<workspace-host>/ai-gateway/gemini"
export GEMINI_API_KEY_AUTH_MECHANISM="bearer"
export GEMINI_API_KEY="<databricks-pat>"
```
- Em `~/.gemini/settings.json`: `{"security":{"auth":{"selectedType":"gemini-api-key"}}}`.
- MCP: use `"httpUrl"` com o header Bearer.

### 4.6 Cursor

**IDE (manual):**
1. Vá em Settings > Models > API Keys > **Override OpenAI Base URL** e coloque `https://<workspace-host>/ai-gateway/cursor/v1`.
2. Coloque o `<databricks-pat>` em "OpenAI API Key".
3. Em **+ Add Custom Model**, adicione os nomes de três partes.
4. Teste no modo Ask (Cmd/Ctrl+L).

O override vale para a IDE inteira.

**MCP:** `ug mcp add --agents cursor` grava `~/.cursor/mcp.json`, usado pela IDE e pelo CLI. O `ug cursor` (Agent CLI) é **só MCP**: os modelos continuam cobrados na conta Cursor.

### 4.7 Pi

**Via `ug`:** `ug pi`. O Pi não tem cliente MCP.

**Manual:** em `~/.pi/agent/models.json`, crie um provider com:
- `"api": "anthropic-messages"`;
- `baseUrl` `https://<workspace-host>/ai-gateway/anthropic`;
- `"authHeader": true`;
- `"compat": {"supportsEagerToolInputStreaming": false}`.

Para GPT, use `openai-responses` com `/ai-gateway/codex/v1`. Para Gemini, use `google-generative-ai` com `/ai-gateway/gemini/v1beta`.

### 4.8 Clientes sem suporte oficial (Cline, Continue, Aider, Goose, Windsurf)

Não há `ug` nem doc oficial para eles. A receita genérica é **não oficial**:
- **Cliente compatível com OpenAI:** base URL `https://<workspace-host>/ai-gateway/mlflow/v1`, API key `<databricks-pat>` (escopo `ai-gateway`), modelo de três partes.
- **Cliente compatível com Anthropic:** base URL `https://<workspace-host>/ai-gateway/anthropic`.
- Para MCP, se o cliente suportar MCP remoto com header: URL `/ai-gateway/mcp-services/...` e `Authorization: Bearer`.

---

## 5. Modo assinatura via gateway (`--provider`)

Nesse modo, a cobrança vai para a assinatura do usuário e o gateway governa o acesso e registra o uso. Pré-requisito: P4 com o modo de autenticação de assinatura.

```bash
ug claude --provider <catalog>.<schema>.<servico-claude-sub> --workspace https://<workspace-host>   # Claude Pro/Max/Team/Enterprise
ug codex  --provider <catalog>.<schema>.<servico-chatgpt-sub> --workspace https://<workspace-host>  # ChatGPT Go/Plus/Pro/Enterprise
```

- **Manual no Claude Code:**
  - Defina `ANTHROPIC_BASE_URL=https://<workspace-host>/ai-gateway/anthropic`.
  - Defina `ANTHROPIC_CUSTOM_HEADERS` com duas linhas, `X-Databricks-AI-Gateway-Token: Bearer <databricks-token>` e `Databricks-Model-Provider-Service: <c>.<s>.<nome>`.
  - Faça login com "Claude account with subscription". Se der problema, use `/logout` e entre de novo.
- **Manual no Codex:** `requires_openai_auth = true` e `http_headers = { "X-Databricks-AI-Gateway-Token" = "Bearer <token>", "Databricks-Model-Provider-Service" = "<c>.<s>.<nome>" }`. Smart Routing não funciona nesse modo.
- **Só Claude Code e Codex** suportam esse modo. OpenCode não suporta (#608).
- **Avisos obrigatórios ao usuário:**
  - **Assinatura Claude:** o recurso é Beta. Os termos da Anthropic (fev/2026) proíbem terceiros de rotear requisições com credenciais Free/Pro/Max. Aqui o cliente é o próprio Claude Code, mas não há aval público da Anthropic. Com plano **Team/Enterprise** da empresa, vale o contrato comercial. Com Pro/Max pessoal, há zona cinzenta.
  - **Nunca** use assinatura Claude em OpenCode ou outros harnesses de terceiros, nem por plugins da comunidade. Houve banimento de contas.
  - **LGPD:** num plano pessoal, os dados da empresa seguem as regras de retenção e uso de plano de consumidor. Para dados sensíveis, use os modelos da empresa.

---

## 6. MCP e skills do Unity Catalog

```bash
ug mcp add                                            # picker
ug mcp add --names system.ai.dbsql,system.ai.github   # nomes do mesmo schema
ug mcp add --location "<catalog>.<schema>"
ug mcp add --agents claude,codex --names <...>
ug mcp list ; ug mcp remove

ug skills add                                         # picker; baixa para ~/.agents/skills/ e .claude/skills/
ug skills add --names <c>.<s>.<skill>
ug skills add --location "<c>.<s>" --via mcp
ug skills list ; ug skills remove
```
- Os MCPs são registrados como servidores stdio que rodam `ug mcp-proxy`, que renova o OAuth a cada requisição.
- Depois de alterar MCPs, **reinicie o agente** com `ug <agente>`.
- MCP manual com OAuth: escopos `ai-gateway` e `offline_access`.
  - Claude Code: `claude mcp add --transport http --scope user --client-id https://claude.ai/oauth/claude-code-client-metadata --callback-port 3118 <nome> "<url-mcp>"`, depois `/mcp`.
  - Codex: `codex mcp add <nome> --url "<url-mcp>" --oauth-client-id https://chatgpt.com/oauth/codex/client.json`, depois `codex mcp login <nome> --scopes ai-gateway,offline_access`.

---

## 7. Verificação (sempre, antes de declarar sucesso)

1. `ug status` mostra o workspace, as base URLs e os modelos. `ug doctor` não aponta erros.
2. Dentro do agente, `/model` (ou `/models`) lista modelos de três partes.
3. Uma pergunta curta recebe resposta.
4. Com MCP: `/mcp` mostra os servidores. Peça "rode `SELECT 1 AS result` via MCP SQL" e confirme que **houve tool call**. Resposta só em texto não valida.
5. O uso aparece na telemetria (pode atrasar alguns minutos):
   ```sql
   SELECT service_name, requester, status_code, COUNT(*) AS calls
   FROM system.ai_gateway.usage
   WHERE service_type = 'MODEL_SERVICE'
   GROUP BY service_name, requester, status_code
   ORDER BY calls DESC;
   ```
   Por padrão, só admins consultam essa tabela. Usuários comuns podem usar `ug usage` ou o Usage Dashboard, se tiverem acesso.
6. Se o usuário também usa os agentes por conta própria, rode o comando puro (`claude`, `codex`) e confirme no `/status` que **não** está indo pelo gateway. Se estiver, faltou o `--user` (seção 3). Os detalhes da convivência estão na skill coding-agent-setup.

---

## 8. Troubleshooting

| Sintoma | Causa provável → correção |
|---|---|
| Qualquer falha de setup | `ug doctor` (oferece correções) e `ug status` |
| `403 ... required scopes: all-apis` | PAT `ai-gateway` usado no host regional legado → use `https://<workspace-host>/ai-gateway/...` |
| 403 / `PERMISSION_DENIED` | Falta P2 ou P3 (`EXECUTE`, `USE CATALOG`/`USE SCHEMA`) → pedido ao admin (2.3); privilégios levam alguns minutos |
| `PERMISSION_DENIED ... Please use Unity Gateway` | "Enforce Unity Gateway" ligado → troque `/serving-endpoints` + `databricks-*` por `/ai-gateway/mlflow/v1` + `system.ai.*` |
| Modelo não aparece | Permissão; nome de três partes; `--refresh`; modelos fora de `system.ai` pedem `--model-location` |
| Claude Code falha com modelo não Anthropic | O modelo precisa suportar Anthropic Messages |
| Tool streaming quebrado | `toolStreaming: false` (OpenCode) / `supportsEagerToolInputStreaming: false` (Pi) |
| Codex: erro de WebSocket | `supports_websockets = false` |
| Gemini CLI falha | Versão ≥ 0.45.0 → instale `<0.45.0` |
| OAuth redirect mismatch / timeout | Redirect do OAuth app ≠ callback; alterações em OAuth apps levam até 30 min; verifique a rede |
| Sudo a cada `ug claude` / `claude` puro indo pelo gateway | `ug revert` + `ug configure --user` ou `UG_SKIP_OS_SETTINGS=1` |
| Proxy corporativo bloqueando MCP | Sem solução documentada (issue #705) → fale com a equipe de rede |
| `web_search` em workspace HIPAA/BAA | Não funciona (issue #805) |
| Windows: `FileNotFoundError` ao lançar `claude` | `ug upgrade` (corrigido no PR #798) |

Não há flag `--debug` documentada. Para depurar, use `ug doctor`, `ug status`, `/mcp` e `system.ai_gateway.usage`.

---

## 9. Perfis de uso: analítico vs. desenvolvimento web

Pergunte ao usuário em qual perfil ele está, porque muda onde o agente roda, quais ferramentas ele recebe e o que é permitido. O admin define os grupos (skill databricks-unity-gateway-admin, seção 11b).

### 9.1 Perfil analítico (projetos de dados, notebooks, ML)

**Opção recomendada: agente rodando no compute Databricks via SSH tunnel.** Os dados não saem do workspace, não há setup local de `ug`, e o Unity Gateway já vem configurado para quem usa o tunnel.
```bash
databricks ssh connect                 # serverless, a partir do terminal da IDE
databricks ssh connect --ide=vscode    # abre VS Code (ou --ide=cursor) apontando para o workspace
```
- Compute clássico exige cluster dedicado (single-user), DBR ≥ 17.0 e UC habilitado; serverless GPU exige AI Runtime. Confira com `databricks ssh --help`.
- Cursor e Copilot funcionam direto no tunnel. Claude Code e Codex podem ser instalados dentro da sessão.
- Os custos do tunnel podem ser atribuídos com `--usage-policy-id` (política de uso serverless).

**Alternativa: local com `ug`** (seções 3 e 4), adicionando os MCPs de dados liberados ao grupo:
```bash
ug mcp add --names system.ai.dbsql      # e Genie / Vector Search / UC Functions, se concedidos
ug claude --header 'Databricks-Ai-Gateway-Request-Tags: {"team":"analytics","project":"<projeto>"}'
```

**Uso programático (notebooks, jobs, apps):**
```python
from openai import OpenAI
import os
client = OpenAI(api_key=os.environ["DATABRICKS_TOKEN"],
                base_url="https://<workspace-host>/ai-gateway/mlflow/v1")
resp = client.chat.completions.create(
    model="system.ai.<modelo>",
    messages=[{"role": "user", "content": "Resuma ..."}],
    extra_headers={"Databricks-Ai-Gateway-Request-Tags": '{"team":"analytics"}'})
```
- A Responses API (`client.responses.create`) e os embeddings usam o mesmo base URL.
- SDK Anthropic: `https://<workspace-host>/ai-gateway/anthropic`. Google GenAI: `.../ai-gateway/gemini`.
- Em SQL, `ai_query(...)` funciona só com modelos `system.ai`.
- Em Databricks Apps, declare `ai-gateway` em `user_api_scopes`; sem isso, a chamada retorna 403.
- **Migração:** código antigo com `/serving-endpoints` + `databricks-<modelo>` deve passar para `/ai-gateway/mlflow/v1` + `system.ai.<modelo>`. Isso é obrigatório se o admin ligar o "Enforce Unity Gateway".
- LangChain: não há guia oficial. Na prática, `ChatOpenAI(base_url=".../ai-gateway/mlflow/v1", model="system.ai.<modelo>")` (não verificado; teste).

**Regras do perfil:**
- Só consulte tabelas e views mascaradas liberadas ao grupo.
- Nunca cole amostras de dados de beneficiários em prompts de agentes locais fora do gateway.
- Prefira o tunnel quando o trabalho tocar dados reais.

### 9.2 Perfil desenvolvimento web

- **Onde roda:** local com `ug`. Agentes típicos:
  - `ug configure --agents copilot` (VS Code);
  - `ug claude`, `ug codex`, `ug opencode`;
  - Cursor IDE com override manual (seção 4.6).
- **Ferramentas:** sem MCPs de dados (o grant não existe e a chamada daria 403). MCP do GitHub conforme liberado, normalmente somente leitura.
- **Request tags:** `UG_HEADERS='Databricks-Ai-Gateway-Request-Tags: {"team":"web","project":"<projeto>"}'` ou `--header`.
- **Aplicações web que chamam LLM:** use `/ai-gateway/mlflow/v1` (ou `/anthropic`) com a identidade da aplicação, e nunca o token pessoal do dev em produção. Em Databricks Apps, use `user_api_scopes: ai-gateway`. Fora do Databricks, combine com o admin uma identidade de serviço.
- **CI (review automatizado, testes com agente):** use perfil com PAT de escopo `ai-gateway` (`ug configure --profile <perfil-pat> --use-pat --skip-validate`), guardado no secret store do CI. O fluxo com service principal (M2M) para o `ug` **não está documentado**; confirme com o admin.
- **Regra do perfil:** código do repositório da empresa só vai pelo gateway (`ug <agente>`), nunca pela conta pessoal.

---

## Fontes

- Repositório: https://github.com/databricks/unity-gateway (README e issues #602, #608, #705, #805, #855, #905, #1078, #1282, #1544 e #1926).
- Docs Databricks (out–nov/2026): https://docs.databricks.com/aws/en/ai-gateway/
  - "Supported coding and work agents", "Unity Gateway CLI", "Migrate to Unity Gateway", "Discover and govern access to model services";
  - páginas "Connect" de Claude Code, Codex, OpenCode, Copilot, Gemini, Pi e Cursor;
  - "Claude Max and Anthropic Enterprise support".
- Termos da Anthropic sobre OAuth de assinatura: https://code.claude.com/docs/en/legal-and-compliance