---
name: "databricks-unity-gateway-admin"
description: "Guia o admin Databricks na liberação do Unity Gateway para agentes de código: previews, grants, provider services (BYOK e assinaturas), Agent configuration, budgets, políticas, provisionamento, auditoria e LGPD."
---

# databricks-unity-gateway-admin

Skill do **lado do workspace e da conta**. Ela habilita, governa e audita o uso de agentes de código (Claude Code, Codex, OpenCode, Copilot, Gemini CLI, Cursor, Pi) pelo Unity Gateway da Databricks.

Outras skills relacionadas:
- **coding-agent-setup**: escolha do modo (pessoal, empresa ou assinatura via gateway) e o que cada assinatura permite.
- **databricks-unity-gateway**: configuração na máquina do desenvolvedor (`ug`, receitas por agente, troubleshooting).

Última verificação: 2026-11-26.

> Várias telas e recursos são Preview ou Beta e mudam rápido. Use os caminhos de menu como referência e confirme na doc oficial (https://docs.databricks.com/aws/en/ai-gateway/) antes de aplicar. Se algo não bater, avise o usuário em vez de improvisar.

---

## 1. Regras fixas

1. **Conceda acesso a grupos, nunca a usuários individuais.** Exemplo: `grp-ia-coding-agents`.
2. **Menor privilégio.** Libere só os modelos, MCPs e skills necessários. Use PAT apenas com escopo `ai-gateway`; o padrão para pessoas é OAuth.
3. **Coloque budget e rate limit antes de liberar acesso.** Liberar sem controle de custo não é aceitável.
4. **Toda mudança precisa ser rastreável.** Versione o JSON do Agent configuration num repositório interno e anote quem mudou e por quê.
5. **Chaves de provedor ficam em secret do Unity Catalog** e nunca em texto puro, notebook ou repositório.
6. **Dados sensíveis (LGPD).** Antes de liberar MCPs de dados (`dbsql`, Genie, Vector Search), passe pelo checklist da seção 9.
7. **Antes de executar, diga ao usuário quais privilégios cada passo exige.** Se ele não tiver o papel necessário, gere o pedido para quem tem.

### Papéis necessários

| Ação | Papel |
|---|---|
| Previews de conta (Enhanced Unity Gateway, Consumer access) | Account admin |
| Previews do workspace, entitlements, Agent configuration, Enforce Unity Gateway | Workspace admin |
| Grants em `system.ai` | Metastore admin ou `MANAGE` em `system.ai` |
| Grants em serviços próprios | `MANAGE` no serviço, schema ou catálogo |
| Criar model/provider service | `CREATE SERVICE` + `USE CATALOG` + `USE SCHEMA` no destino |
| Consultar `system.ai_gateway.usage` | Account admin + metastore admin, ou grant explícito em system tables |

---

## 2. Fluxo de habilitação (siga nesta ordem)

```
0. Decisões    → quais agentes, modelos, MCPs; quem paga (empresa/BYOK/assinatura); grupos
1. Previews    → conta e workspace (seção 3)
2. Identidade  → grupo + entitlements (seção 4)
3. Modelos     → grants em model services; provider services se BYOK/assinatura (seções 4 e 5)
4. Controles   → budgets, rate limits, políticas/guardrails (seção 6)
5. Observar    → usage tracking, inference tables, acesso às system tables (seção 8)
6. Padronizar  → Agent configuration (seção 7)
7. Piloto      → 2–3 devs seguem a skill databricks-unity-gateway; validar ponta a ponta
8. Enforce     → ligar "Enforce Unity Gateway" após migrar integrações legadas (seção 6.4)
9. Provisionar → instalação em massa, se houver MDM (seção 10)
10. Comunicar  → anúncio ao time (seção 11)
```

No passo 0, levante com o usuário:
- quais agentes serão habilitados e qual será o padrão;
- quais modelos (`system.ai.*` e/ou BYOK);
- se o modo assinatura será permitido e com quais planos;
- quais MCPs;
- o budget por usuário e o total;
- quais dados os agentes poderão acessar.

---

## 3. Previews

Os nomes dos previews mudam. Confira a lista atual na página de Previews.

| Preview | Onde | Para quê |
|---|---|---|
| **Enhanced Unity Gateway** | Account console > Previews | MCPs `system.ai.dbsql`, `system.ai.sandbox`, `system.ai.web_search`; budgets com bloqueio |
| **Consumer access to Unity Gateway** | Account console > Previews | Usuários só com Consumer access |
| **Unity Gateway Managed Configuration** | Workspace > Previews | Publicar o Agent configuration |
| Model provider services / assinatura Claude (Beta) | Previews (conta ou workspace, conforme a doc) | BYOK e modo assinatura |

---

## 4. Identidade e grants

### 4.1 Grupo e entitlements
1. Crie o grupo (prefira o account console ou sincronize via SCIM/IdP, como o Entra ID).
2. Em **Settings > Identity and access > Groups > <grupo> > Entitlements**, marque **Workspace access**.
   - Com só *Consumer access*, o grupo precisa do preview "Consumer access to Unity Gateway" e perde alguns MCPs (Vector Search, UC Functions, Genie, Apps).

### 4.2 Model services
- **Os model services do `system.ai` já vêm com `EXECUTE` para todos os usuários da conta.** Para restringir, revogue esse acesso e conceda só ao grupo. Isso exige metastore admin ou `MANAGE` em `system.ai`.
- Em serviços próprios: Catalog Explorer > serviço > **Permissions** > **Grant** > grupo > `EXECUTE`. Além disso:
  ```sql
  GRANT USE CATALOG ON CATALOG <catalog> TO `<grupo>`;
  GRANT USE SCHEMA  ON SCHEMA  <catalog>.<schema> TO `<grupo>`;
  ```
- Para criar um model service próprio: `EXECUTE` no modelo de base, mais `CREATE SERVICE`, `USE CATALOG` e `USE SCHEMA` no destino.
- `CAN_QUERY` de Model Serving e permissão sobre o modelo **não** dão acesso ao model service.
- Mudanças de privilégio levam alguns minutos para valer.

### 4.3 MCP services e skills
- Conceda `EXECUTE` nos MCP services liberados (ex.: `system.ai.dbsql`, `main.developer_tools.github`) e nas skills do UC.
- MCPs de terceiros (GitHub, Slack) exigem um segundo login do usuário no provedor. Prefira políticas somente leitura (seção 6.3).

---

## 5. Model provider services (BYOK e assinaturas)

Privilégios para criar: `CREATE SERVICE`, `USE CATALOG` e `USE SCHEMA` no schema de destino. Para os usuários: `EXECUTE` no serviço, mais `USE CATALOG`/`USE SCHEMA`.

| Tipo | Credencial | Quem paga | Agentes (`--provider`) |
|---|---|---|---|
| **BYOK** (OpenAI, Anthropic, Bedrock etc.) | API key da empresa num **secret do UC** | Empresa (conta do provedor) | `ug claude`, `ug codex` |
| **Assinatura Claude** ("Claude Max/Enterprise", Beta) | Nenhuma: repassa o token OAuth de cada usuário | Assinatura do usuário (Pro/Max/Team/Enterprise) | Só Claude Code |
| **Assinatura ChatGPT** | Nenhuma: repassa a credencial do usuário | Assinatura do usuário (Go/Plus/Pro/Enterprise) | Codex e ChatGPT desktop |

Passos:
1. Vá em Unity Gateway > criar model provider service, escolha o tipo e o modo de autenticação e defina `<catalog>.<schema>.<nome>`.
2. No BYOK, aponte para o secret do UC que guarda a chave.
3. Defina quais modelos o serviço permite.
4. Conceda `EXECUTE` ao grupo.
5. Aplique rate limits e políticas no nível do serviço.
6. Informe aos devs o nome de três partes para usar em `--provider`.

**Decisão sobre o modo assinatura Claude.** Recomende ao usuário:
- Prefira liberar só com planos **Team/Enterprise** contratados pela empresa, porque aí vale o contrato comercial.
- Com Pro/Max pessoal há zona cinzenta: os termos da Anthropic (fev/2026) proíbem terceiros de rotear requisições com credenciais Free/Pro/Max, e não há aval público da Anthropic a esse arranjo.
- Com assinatura pessoal, os dados seguem as regras de retenção e uso de plano de consumidor, o que conflita com dado sensível (seção 9).
- O OpenCode não suporta `--provider` (issue #608).

---

## 6. Controles: budgets, rate limits, políticas e enforce

### 6.1 Budgets
- Tipos: mensais e diários, pessoais (por usuário) e compartilhados. Webhooks de alerta estão em Beta.
- Para ligar o budget ao Agent configuration:
  1. O account admin habilita o preview **Enhanced Unity Gateway**.
  2. Crie um budget do Unity Gateway com **limite por usuário** e **Block usage** ativado.
  3. Coloque o ID em `smart_defaults.budget_id` no Agent configuration (seção 7).
  4. Os devs acompanham o próprio gasto com `ug usage`.
- Funções de IA como `ai_parse_document` e `ai_extract` contam no budget, mas não aparecem na tabela de uso.

### 6.2 Rate limits
- Por serviço, ou por usuário, grupo ou service principal, em requisições e/ou tokens por minuto.
- São configurados no model service, no provider service ou no MCP service.
- Sugestão inicial: um limite por usuário conservador no piloto, ajustado depois pelo uso real (seção 8).

### 6.3 Políticas e guardrails
- **Guardrails de modelo:** jailbreak e PII (essencial com dado de saúde).
- **Políticas de MCP:** por exemplo, GitHub com escrita desabilitada e aprovação humana para ferramentas destrutivas.
- Aplique primeiro nos MCPs de dados.

### 6.4 Enforce Unity Gateway (workspace admin)
- Bloqueia o acesso legado (`/serving-endpoints`, endpoints `databricks-*`). As chamadas antigas passam a falhar com `PERMISSION_DENIED ... Please use Unity Gateway`.
- **Antes de ligar:** faça o inventário de jobs, apps, notebooks e integrações que usam `/serving-endpoints` e migre para `/ai-gateway/mlflow/v1` com nomes `system.ai.*`.
- Em Databricks Apps, o app precisa do scope `ai-gateway` em `user_api_scopes`.

---

## 7. Agent configuration (configuração gerenciada)

**Pré-requisito:** preview "Unity Gateway Managed Configuration" e papel de workspace admin. O workspace tem **uma** configuração.

**Pela UI:** Unity Gateway > Govern > Agent configuration > **Edit configuration** > editar o JSON > **Format JSON** > **Save**.

**Pela API (recomendado, permite versionar):**
```bash
databricks api post /api/ai-gateway/v2/coding-agent-configs --json @coding-agent-config.json
# alterações: PATCH /api/ai-gateway/v2/coding-agent-configs/{id}?update_mask=<campos>
```

**Campos:**
- `default_agent`: precisa estar também em `enabled_agents`.
- `enabled_agents[]`: lista de `{ agent, config }`.
- `config.models`: **exatamente uma** fonte por agente:
  - `model_services`: nomes UC completos, compatíveis com a API do agente;
  - `model_provider_service`: nome UC de um provider service (todos os modelos dele);
  - `unity_catalog_location`: catálogo e schema (os serviços ali ficam disponíveis a quem tem `EXECUTE`).
- `config.default_models.default_model`.
- `mcp_servers.names` e `skills.names`.
- `smart_defaults.budget_id`.
- Smart Routing e tracing aparecem como áreas configuráveis, sem chave documentada. Confira na doc.

Exemplo mínimo:
```json
{
  "default_agent": "CODING_AGENT_CLAUDE_CODE",
  "enabled_agents": [
    {
      "agent": "CODING_AGENT_CLAUDE_CODE",
      "config": {
        "models": { "model_services": ["system.ai.<modelo-claude>"] },
        "default_models": { "default_model": "system.ai.<modelo-claude>" }
      }
    }
  ],
  "mcp_servers": { "names": ["system.ai.dbsql"] },
  "skills": { "names": ["<catalog>.<schema>.<skill>"] },
  "smart_defaults": { "budget_id": "<budget-id>" }
}
```
- Os identificadores dos outros agentes seguem o padrão `CODING_AGENT_*`. Confirme os valores exatos na doc ou com `ug export`. **Não invente.**
- **Sincronização:** os agentes checam atualizações ao iniciar, e a mudança pode levar alguns minutos. Para forçar, o dev roda `ug configure`. Para conferir o que chegou: `ug export`.
- Agentes fora de `enabled_agents` ainda podem ser usados em modo "self-managed" com `ug agents add <agente>`. Se isso não for aceitável, trate por política interna.

---

## 8. Observabilidade e auditoria

### 8.1 Usage tracking: `system.ai_gateway.usage`
- Não há passo de habilitação separado. Os registros costumam aparecer em até cerca de 1 hora; em workspaces novos pode demorar mais.
- **É um recurso cobrado.**
- Por padrão, só account admin + metastore admin consultam. Para liberar a um grupo de FinOps ou de auditoria:
  ```sql
  GRANT USE SCHEMA ON SCHEMA system.ai_gateway TO `<grupo-auditoria>`;
  GRANT SELECT ON TABLE system.ai_gateway.usage TO `<grupo-auditoria>`;
  ```
- Colunas úteis: `service_type` (`MODEL_SERVICE`, `MCP_SERVICE`, `MODEL_PROVIDER_SERVICE`), `service_name`, `requester`, `user_agent`, `auth_mode`, `status_code`, `input_tokens`, `output_tokens`, `total_tokens`, `latency_ms`, `request_tags`, `event_time` e `session_metadata` (`coding_agent`, `client_session_id`, `surface`, `reasoning_effort`, `smart_router_name`).
- Limitação: respostas não streaming acima de 1 MiB não têm tokens registrados.

Consultas prontas:
```sql
-- Tokens por usuário e agente nos últimos 30 dias
SELECT requester,
       session_metadata.coding_agent AS agente,
       SUM(total_tokens) AS tokens,
       COUNT(*) AS chamadas
FROM system.ai_gateway.usage
WHERE event_time >= current_date() - INTERVAL 30 DAYS
GROUP BY ALL
ORDER BY tokens DESC;

-- Erros por serviço e usuário (diagnóstico de permissão/rate limit)
SELECT service_name, requester, status_code, COUNT(*) AS chamadas
FROM system.ai_gateway.usage
WHERE status_code >= 400 AND event_time >= current_date() - INTERVAL 7 DAYS
GROUP BY ALL
ORDER BY chamadas DESC;

-- Uso de MCPs (quem chama quais ferramentas)
SELECT service_name, requester, COUNT(*) AS chamadas
FROM system.ai_gateway.usage
WHERE service_type = 'MCP_SERVICE'
GROUP BY ALL
ORDER BY chamadas DESC;

-- Uso por modo de autenticação (separa assinatura de modelo da empresa)
SELECT service_type, auth_mode, COUNT(*) AS chamadas, SUM(total_tokens) AS tokens
FROM system.ai_gateway.usage
GROUP BY ALL;

-- Por projeto (exige o header Databricks-Ai-Gateway-Request-Tags)
SELECT request_tags['project'] AS projeto, COUNT(*) AS chamadas, SUM(total_tokens) AS tokens
FROM system.ai_gateway.usage
WHERE request_tags['project'] IS NOT NULL
GROUP BY ALL
ORDER BY tokens DESC;
```

- Os campos aninhados de `session_metadata` e os valores exatos de `auth_mode` podem variar. Se a consulta falhar, rode `DESCRIBE system.ai_gateway.usage`.
- **Request tags:** os devs ou apps enviam o header `Databricks-Ai-Gateway-Request-Tags` com um JSON (ex.: `{"project": "x", "team": "y"}`). Com o `ug`, use `--header` ou `UG_HEADERS`.

### 8.2 Outras fontes
- **Inference tables:** payloads completos (prompts e respostas) em Delta. Ligue no serviço e restrinja o acesso, porque contêm dado sensível.
- **Usage Dashboard:** AI Gateway > Govern > Usage Dashboard, aba Coding Agents.
- `system.access.audit` (auditoria) e `system.billing.usage` (cobrança).

---

## 9. Checklist LGPD e dados sensíveis (saúde suplementar)

Antes de liberar acesso a dados reais:
- [ ] Base legal e finalidade definidas com o DPO/Encarregado para o uso de IA com dados pessoais e sensíveis (dados de saúde são sensíveis pela LGPD).
- [ ] MCPs de dados (`dbsql`, Genie, Vector Search) liberados só sobre tabelas e views **mascaradas ou anonimizadas**, com row filters e column masks do UC.
- [ ] Guardrails de PII ativos nos model services usados com dados.
- [ ] Inference tables com acesso restrito e retenção definida, porque guardam os prompts.
- [ ] Modo assinatura pessoal **desabilitado** para workloads com dados de beneficiários.
- [ ] Provedores externos (BYOK) avaliados quanto a retenção de dados e transferência internacional.
- [ ] Se o workspace usa compliance security profile (HIPAA/BAA): o MCP `web_search` não funciona nesse perfil (issue #805).
- [ ] Revisão periódica das consultas da seção 8 por auditoria interna.

Esta seção é orientação técnica e não substitui o parecer jurídico ou do DPO.

---

## 10. Provisionamento em massa (MDM)

- O repositório traz `scripts/mdm/macos-provision.sh`. Ele instala o `ug` em local compartilhado usando `UV_TOOL_DIR`, `UV_TOOL_BIN_DIR` e `UV_PYTHON_INSTALL_DIR`. Leia o script antes de usar.
- **Settings gerenciados de máquina:** sem `--user`, o `ug configure` grava `managed-settings.json` (Claude Code) e `managed_config.toml` (Codex) via sudo. Isso força o gateway na máquina inteira.
  - Útil em máquinas **corporativas** que só devem usar o gateway.
  - Em máquinas que também têm uso pessoal, instrua os devs a usar `--user`.
  - Gera atrito com sudo (issues #1078 e #1282).
- Proxy corporativo: o `ug mcp-proxy` não tem suporte documentado a proxy HTTP (issue #705). Teste na rede corporativa durante o piloto.

---

## 11. Comunicar o time

Modelo de anúncio (preencha):

```
Assunto: Agentes de código via Unity Gateway liberados

Quem: membros do grupo <grupo>
Agentes: <lista>  | Padrão: <agente>
Modelos: <lista de nomes de 3 partes>
MCPs: <lista>     | Budget: <valor>/usuário/<período>

Como começar:
1. uv tool install git+https://github.com/databricks/unity-gateway
2. ug configure --workspace https://<workspace-host> --user
3. ug <agente>  (ex.: ug claude)

Regras: dados de beneficiários só pelos modelos da empresa; nada de assinatura pessoal em trabalho com dados;
nunca commitar tokens. Acompanhe seu gasto com `ug usage`.
Dúvidas/erros: <canal>  (anexe a saída de `ug doctor`)
```

---

## 11b. Públicos distintos: analítico vs. desenvolvimento web

Há dois públicos governados pelo mesmo gateway, com necessidades opostas: um precisa de dados, o outro não deve ter acesso a eles.

**Restrição-chave: o workspace aceita UMA única Agent configuration.** Por isso a diferenciação entre públicos no mesmo workspace não vem do JSON, e sim de **grupos + grants + budgets + rate limits**. Se o isolamento precisar ser mais forte, use workspaces separados (opção B).

### Matriz por público

| Aspecto | Analítico (`grp-ia-analytics`) | Web (`grp-ia-web`) |
|---|---|---|
| Entitlement | *Workspace access* (precisa de MCPs de dados, notebooks, compute) | *Consumer access* basta se só usar modelos (exige o preview "Consumer access to Unity Gateway"); senão, *Workspace access* |
| Modelos (`EXECUTE`) | Modelos de raciocínio/código + embeddings | Modelos de código; considere modelos mais baratos como padrão |
| MCPs (`EXECUTE`) | `system.ai.dbsql`, Genie, Vector Search, UC Functions, **só sobre views mascaradas** (seção 9) | **Nenhum MCP de dados.** GitHub somente leitura, docs, `web_search` (se permitido) |
| Onde o agente roda | Preferir **compute Databricks via SSH tunnel** (dados não saem do workspace) ou local com `ug` | Local com `ug` (IDE: Copilot/VS Code, Cursor, Claude Code) |
| Uso programático | `ai_query` (só modelos `system.ai`), SDKs em notebooks/jobs, Databricks Apps (`user_api_scopes: ai-gateway`) | Apps web chamando `/ai-gateway/mlflow/v1` com identidade de app/SP; CI com PAT `ai-gateway` |
| Budget | Budget compartilhado do time + limite por usuário | Idem, separado (centro de custo próprio) |
| Rate limit | Por grupo | Por grupo |
| Políticas | Guardrail de PII obrigatório; inference tables com acesso restrito | Jailbreak; GitHub sem escrita / aprovação humana para ações destrutivas |
| Request tags | `{"team":"analytics","project":"<x>"}` | `{"team":"web","project":"<x>"}` |

### Opção A: um workspace, dois grupos (mais simples)
1. Crie os dois grupos (seção 4.1) e dê os entitlements da matriz.
2. Revogue o `EXECUTE` padrão de "todos os usuários" nos serviços que um dos públicos não deve usar e conceda por grupo (seção 4.2).
3. Conceda `EXECUTE` nos MCPs de dados **só** ao grupo analítico (seção 4.3).
4. Crie budgets e rate limits por grupo (seção 6).
5. Agent configuration única (seção 7):
   - `enabled_agents` = união dos agentes dos dois públicos;
   - `mcp_servers` = só os MCPs comuns, como o GitHub somente leitura;
   - MCPs de dados ficam **fora** da lista central. O grupo analítico os adiciona com `ug mcp add --names ...` (o grant controla quem consegue usar).
   - Assim o público web não recebe ferramentas que daria 403.
6. Peça request tags por time (`UG_HEADERS` ou `--header`) para separar o uso em `system.ai_gateway.usage`.

### Opção B: workspaces separados (isolamento forte)
- Um workspace "dev-web" sem acesso aos catálogos de dados de beneficiários (workspace binding no UC), com a sua própria Agent configuration, budgets e políticas.
- O workspace analítico mantém os MCPs de dados.
- Recomendado quando o time web é externo/terceirizado ou quando a LGPD exige segregação comprovável.
- Custo: duas configurações para manter e comunicados separados (seção 11).

### Consultas por público
```sql
-- Uso por time (via request tags)
SELECT request_tags['team'] AS time, service_type, COUNT(*) AS chamadas, SUM(total_tokens) AS tokens
FROM system.ai_gateway.usage
WHERE event_time >= current_date() - INTERVAL 30 DAYS
GROUP BY ALL ORDER BY tokens DESC;

-- Alerta: alguém fora do grupo analítico tentando MCP de dados (403)
SELECT requester, service_name, COUNT(*) AS tentativas
FROM system.ai_gateway.usage
WHERE service_type = 'MCP_SERVICE' AND status_code = 403
GROUP BY ALL ORDER BY tentativas DESC;
```

### Verificação extra
- Um usuário do grupo web recebe **403** ao chamar `system.ai.dbsql`.
- Um usuário do grupo analítico consegue um `SELECT 1` via MCP, mas **não** lê tabelas sem máscara.

---

## 12. Verificação do admin

1. Um dev do piloto completa o checklist da skill databricks-unity-gateway (inclusive a tool call MCP).
2. `ug export`, na máquina do dev, mostra a configuração publicada.
3. As chamadas do piloto aparecem em `system.ai_gateway.usage` com o `requester` correto.
4. Um usuário **fora** do grupo recebe 403 (o controle de acesso funciona).
5. O budget bloqueia ao estourar o limite (teste com limite baixo num usuário de teste).
6. Com o enforce ligado, uma chamada legada a `/serving-endpoints` é bloqueada.

## Fontes

- Docs Databricks (out–nov/2026): https://docs.databricks.com/aws/en/ai-gateway/
  - "Configure and govern coding agents", "Usage tracking", "Migrate to Unity Gateway", "Discover and govern access to model services";
  - "Integrate coding agents with model provider services", "Claude Max and Anthropic Enterprise support".
- Repositório https://github.com/databricks/unity-gateway (`scripts/mdm/`, issues #608, #705, #805, #1078 e #1282).
- Termos da Anthropic sobre OAuth de assinatura: https://code.claude.com/docs/en/legal-and-compliance