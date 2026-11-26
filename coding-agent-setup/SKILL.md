---
name: "coding-agent-setup"
description: "Escolhe e configura agentes de código (OpenCode, Claude Code, Codex, Copilot, Gemini CLI, Cursor, Pi) no modo pessoal, empresa ou assinatura via gateway, diz o que cada assinatura permite e faz os modos conviverem."
---

# coding-agent-setup

Skill agnóstica e **ponto de entrada** para configurar harnesses de código (o programa que roda o agente) contra provedores de modelo. Ela cuida de quatro coisas:

- decidir **qual modo** usar (pessoal, empresa ou assinatura via gateway);
- dizer **o que é permitido** com cada assinatura e harness;
- configurar o **modo pessoal**;
- fazer os modos **conviverem** na mesma máquina.

Tudo o que é específico do Unity Gateway da Databricks (pré-requisitos, `ug`, configs manuais, MCP, troubleshooting) fica na skill **databricks-unity-gateway**.

Última verificação: 2026-11-26.

> As ferramentas mudam rápido. Se a versão instalada for bem mais nova que a data acima, avise o usuário e confira com `<ferramenta> --help` antes de aplicar.

---

## 1. Regras fixas

1. **Confira flags com `--help`** antes de usar.
2. **Nunca invente nome de modelo.** Liste com `/models` (OpenCode) ou `/model` (Claude Code, Codex).
3. **Use placeholders e nunca valores reais**: `<anthropic-api-key>`, `<openai-api-key>`. Em arquivo, prefira variável de ambiente (`{env:VAR}` no OpenCode).
4. **Segredo nunca vai para o git.** Exemplos: `~/.local/share/opencode/auth.json`, `opencode.json` com chave, `~/.claude/settings.json`, `~/.codex/`. Configs versionados no projeto não podem conter credencial.
5. **Edite com merge.** Faça backup antes (`cp arq arq.bak.$(date +%F)`).
6. **Dados da empresa só no modo empresa.** Código, dados de clientes ou beneficiários e qualquer informação sujeita à LGPD não passam por assinatura nem por API key pessoal.
7. **Só declare sucesso depois da verificação da seção 6**, incluindo conferir *quem foi cobrado*.

---

## 2. Os três modos

| Modo | Quem paga | Governança | Quando usar |
|---|---|---|---|
| **A. Pessoal direto** | O usuário (assinatura ou API key própria) | Nenhuma corporativa | Projetos pessoais, estudo, sem dado da empresa |
| **B. Empresa via Unity Gateway** | A empresa (modelos `system.ai.*` ou BYOK) | Permissões UC, rate limit, budget, logs | Trabalho da empresa (padrão corporativo) |
| **C. Assinatura pessoal via gateway** | O usuário (assinatura) | Governança e logs da empresa | Só se a empresa permitir; ver restrições |

Os modos B e C são configurados pela skill **databricks-unity-gateway** (seções 2 a 5 dela).

### Matriz harness × modo

| Harness | A. Pessoal | B. Empresa | C. Assinatura via gateway |
|---|---|---|---|
| **OpenCode** | ✅ ChatGPT Plus/Pro via `/connect`; ✅ API keys; ❌ **assinatura Claude** | ✅ `ug opencode` | ❌ (issue #608) |
| **Claude Code** | ✅ assinatura Claude ou API key | ✅ `ug claude` | ✅ Beta; prefira plano Team/Enterprise |
| **Codex** | ✅ login ChatGPT ou API key | ✅ `ug codex` | ✅ |
| **GitHub Copilot** | ✅ assinatura Copilot | ✅ `ug copilot` | — |
| **Gemini CLI** | ✅ conta Google ou API key | ✅ `ug gemini` | — |
| **Cursor** | ✅ assinatura Cursor | IDE manual; CLI só MCP | — |
| **Pi** | ✅ API keys | ✅ `ug pi` (sem MCP) | — |
| **Outros** (Cline, Continue, Aider, Goose) | ✅ API keys | ⚠️ genérico, não oficial | — |

### Restrição crítica: assinatura Claude em harness de terceiros

Os termos da Anthropic (revisados em fev/2026 e aplicados desde abr/2026) **proíbem usar o login OAuth das assinaturas Claude Free, Pro ou Max em harnesses de terceiros**. Houve contas banidas, e o OpenCode removeu o suporte.

- **Nunca** configure assinatura Claude no OpenCode, Cline, Aider etc., nem por plugins da comunidade (`opencode-anthropic-auth`, proxies do tipo "meridian"). Se o usuário pedir, explique o risco e ofereça as alternativas: API key da Anthropic, assinatura no Claude Code ou o modo empresa.
- A assinatura Claude só pode ser usada **no Claude Code** e nos apps oficiais da Anthropic.
- O **modo C com Claude** é Beta na Databricks. O cliente é o próprio Claude Code, mas não há aval público da Anthropic, então é uma zona cinzenta para Pro/Max pessoal. Com plano **Team/Enterprise** contratado pela empresa, vale o contrato comercial. Sempre avise, junto com a regra 6.
- A **assinatura ChatGPT** em harnesses de terceiros é aceita na prática (o OpenCode suporta via `/connect` desde a v1.1.11). Use para fins pessoais.

---

## 3. Árvore de decisão

```
1. O trabalho envolve código ou dados da empresa?
   ├─ Sim → Modo B. A empresa tem Unity Gateway?
   │        ├─ Sim → skill databricks-unity-gateway.
   │        └─ Não → só ferramentas/contas aprovadas pela empresa. Não improvise com conta pessoal.
   └─ Não → Modo A (seção 4).
2. No modo A, qual assinatura ou chave o usuário tem?
   ├─ Claude Pro/Max   → Claude Code (único harness permitido).
   ├─ ChatGPT Plus/Pro → Codex ou OpenCode (/connect → OpenAI).
   ├─ API keys         → OpenCode (agnóstico, troca de modelo com /models).
   └─ Várias           → OpenCode (API keys + ChatGPT) e Claude Code (assinatura Claude).
3. Pessoal e empresa na mesma máquina? → seção 5.
4. No modo empresa, qual é o perfil?
   ├─ Analítico (dados, notebooks, ML) → databricks-unity-gateway §9.1
   │     (preferir agente no compute Databricks via SSH tunnel; MCPs de dados só sobre views mascaradas)
   └─ Desenvolvimento web → databricks-unity-gateway §9.2
         (local com ug; sem MCPs de dados; CI com PAT ai-gateway)
```

**Setup agnóstico recomendado:** o OpenCode é o harness principal.
- No uso pessoal, ele é aberto com `opencode` e lê `~/.config/opencode/`.
- No uso da empresa, é aberto com `ug opencode`, que usa um config isolado.
- O Claude Code fica como segundo harness, para quem tem assinatura Claude.

---

## 4. Receitas do modo pessoal (A)

Em todas as receitas abaixo, o modo empresa é `ug <agente>`, com os detalhes na skill **databricks-unity-gateway**.

### 4.1 OpenCode
- **Instalar:** `curl -fsSL https://opencode.ai/install | bash` e conferir com `opencode --version`. As credenciais do `/connect` ficam em `~/.local/share/opencode/auth.json` (privado).
- **ChatGPT Plus/Pro:** abra `opencode`, rode `/connect` → OpenAI → login ChatGPT (ou `opencode auth login`). Escolha o modelo com `/models`.
- **API key Anthropic/OpenAI/Google:** `/connect` → provedor → API key, ou por variável de ambiente no config.
- **Assinatura Claude: NÃO** (seção 2).

Exemplo de `~/.config/opencode/opencode.json` (confira o schema em https://opencode.ai/config.json):
```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "openai/<modelo-gpt>",
  "provider": {
    "anthropic": { "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" } }
  }
}
```
Não misture providers do gateway neste arquivo: para isso existe o `ug opencode` (seção 5).

### 4.2 Claude Code
- **Assinatura (Pro/Max/Team/Enterprise):** rode `claude` → login → "Claude account with subscription".
- **API key:** `export ANTHROPIC_API_KEY=<anthropic-api-key>`. Se as duas formas estiverem presentes, a API key tem precedência e a cobrança vai para ela.

### 4.3 Codex
- **Assinatura ChatGPT:** `codex login`, com login no navegador.
- **API key:** `codex login --with-api-key` (lê a chave do stdin) ou `OPENAI_API_KEY`. Confira com `codex login --help`.

### 4.4 GitHub Copilot
Assinatura Copilot normal, com login GitHub no `copilot` ou no VS Code.

### 4.5 Gemini CLI
Login com conta Google ou `GEMINI_API_KEY`.

### 4.6 Cursor
Assinatura Cursor normal. Se o override de base URL do gateway estiver ativo na IDE, ele vale para tudo (seção 5).

### 4.7 Pi e outros (Cline, Continue, Aider, Goose)
Use as API keys do próprio provedor, conforme a doc de cada ferramenta. **Nunca** use a assinatura Claude.

---

## 5. Convivência pessoal + empresa na mesma máquina

**Princípio:** cada modo tem o seu ponto de entrada.

| Harness | Pessoal | Empresa |
|---|---|---|
| OpenCode | `opencode` (lê `~/.config/opencode/`) | `ug opencode` (config isolado em `~/.ucode/opencode-xdg/`) |
| Claude Code | `claude` | `ug claude` |
| Codex | `codex` | `ug codex` |
| Gemini CLI | `gemini` | `ug gemini` (HOME isolado) |
| Copilot CLI | `copilot` | `ug copilot` |

Regras:
1. Configure o `ug` com `ug configure --user` (ou `UG_SKIP_OS_SETTINGS=1`). Sem isso, ele grava settings de máquina via sudo, os comandos `claude` e `codex` puros passam a usar o gateway e o modo pessoal deixa de funcionar.
2. **Não use a configuração manual do gateway** em `~/.claude/settings.json` ou `~/.codex/config.toml`, porque ela redireciona o comando puro. Prefira o `ug`.
3. O override de base URL do Cursor IDE é global. Para o modo empresa, prefira outro harness.
4. **Teste de isolamento:** rode o comando puro e confira no `/status` que aparece a conta pessoal. Depois rode `ug <agente>` e confira que aparece o gateway.
5. Mantenha as pastas de trabalho da empresa separadas e use só `ug <agente>` dentro delas (regra 6).

---

## 6. Verificação (sempre, antes de declarar sucesso)

1. O harness lista os modelos esperados (`/models` ou `/model`).
2. Uma pergunta curta recebe resposta.
3. **Cobrança correta:** `/status` (ou equivalente) mostra o método esperado: assinatura, API key ou gateway.
4. No modo B ou C, rode também o checklist de verificação da skill databricks-unity-gateway.
5. Se os modos convivem na máquina, o teste de isolamento da seção 5 passou.

---

## 7. Quando sair desta skill

- **Modo B ou C** (pré-requisitos do Databricks, instalação e uso do `ug`, configs manuais do gateway, MCP e skills do UC, erros 403): skill **databricks-unity-gateway**.
- Se ela não existir, consulte https://docs.databricks.com/aws/en/ai-gateway/ e https://github.com/databricks/unity-gateway.

## Fontes

- Termos da Anthropic sobre OAuth de assinatura em terceiros: https://code.claude.com/docs/en/legal-and-compliance (fev/2026).
- Docs e changelog do OpenCode (`/connect`, ChatGPT Plus/Pro desde a v1.1.11; remoção do login Claude Pro/Max em mar/2026).
- Repositório `databricks/unity-gateway` (issues #608 e #1282) e docs Databricks "Claude Max and Anthropic Enterprise support".