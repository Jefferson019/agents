# agents

Coleção de **skills** (arquivos `SKILL.md`) para agentes de IA, voltadas ao ecossistema **Databricks** e ao uso governado de agentes de código. As instruções são escritas em português.

Cada skill é uma pasta com um `SKILL.md`: um frontmatter YAML com `name` e `description`, seguido das instruções. Algumas pastas têm também `references/` com material de apoio. O agente lê a `description` para decidir quando acionar a skill e só carrega o resto quando precisa.

## Skills

### Engenharia e qualidade no Databricks

| Skill | Para quê |
|---|---|
| [`databricks-audit-rules`](databricks-audit-rules/SKILL.md) | Audita notebooks, PySpark e SQL quanto a padrões de engenharia, governança no Unity Catalog, uso de memória no driver, credenciais e prontidão para DABs. Gera relatório com IDs de regra, severidade e veredito. |
| [`databricks-security-secrets`](databricks-security-secrets/SKILL.md) | Remedia credenciais expostas: revogação e rotação, Secret Scopes com ACLs, connections e service credentials do UC, OAuth/Service Principal e OIDC no CI/CD, limpeza do histórico Git. |
| [`databricks-dabs-deploy`](databricks-dabs-deploy/SKILL.md) | Cria, valida e implanta Databricks Asset Bundles: `databricks.yml`, targets dev/staging/prod com `run_as`, migração de jobs da UI para código e CI/CD com OIDC. |

Fluxo típico: `databricks-audit-rules` encontra os problemas, `databricks-security-secrets` corrige os de credenciais e `databricks-dabs-deploy` leva o código para produção.

### Agentes de código e Unity Gateway

| Skill | Para quem | Para quê |
|---|---|---|
| [`coding-agent-setup`](coding-agent-setup/SKILL.md) | Qualquer usuário | Ponto de entrada. Ajuda a escolher o modo (pessoal, empresa ou assinatura via gateway), mostra o que cada assinatura permite em cada ferramenta (OpenCode, Claude Code, Codex, Copilot, Gemini CLI, Cursor, Pi), configura o uso pessoal e faz os modos conviverem na mesma máquina. |
| [`databricks-unity-gateway`](databricks-unity-gateway/SKILL.md) | Desenvolvedor | Configura os agentes no Unity Gateway da Databricks com a CLI `ug`. Cobre pré-requisitos passo a passo, receitas por agente, modo assinatura, MCP e skills do UC, perfis analítico e web, verificação e troubleshooting. |
| [`databricks-unity-gateway-admin`](databricks-unity-gateway-admin/SKILL.md) | Admin do workspace ou da conta | Libera e governa o gateway: previews, grants, provider services (BYOK e assinaturas), Agent configuration, budgets, políticas, auditoria em `system.ai_gateway.usage`, LGPD e separação entre públicos. |

Como as três se conectam:

```
coding-agent-setup ──modo pessoal──► resolve sozinha
        │
        └─modo empresa──► databricks-unity-gateway ──falta permissão──► databricks-unity-gateway-admin
```

> As skills de Unity Gateway foram verificadas em 2026-11-26, com `ug` 0.2.x. O CLI muda com frequência, então cada skill manda conferir os comandos com `--help` antes de aplicar.

## Instalação

**Claude Code**: copie as pastas desejadas para um destes locais.

```bash
# para todos os projetos do usuário
cp -r databricks-audit-rules ~/.claude/skills/

# só para um projeto (versionado junto com ele)
mkdir -p .claude/skills && cp -r databricks-audit-rules .claude/skills/
```

**claude.ai / app Claude**: compacte a pasta da skill em `.zip` e envie pela área de Skills nas configurações.

**Outros agentes**: os que leem skills no formato `SKILL.md` em `~/.agents/skills/` (por exemplo GitHub Copilot CLI e Pi) aceitam as pastas como estão. Confira a documentação do seu agente.

## Estrutura

```
<nome-da-skill>/
├── SKILL.md          # obrigatório: frontmatter (name, description) + instruções
└── references/       # opcional: exemplos e material carregado sob demanda
```

## Convenções para novas skills

- O nome da pasta é igual ao `name` do frontmatter, em kebab-case.
- A `description` diz **o que a skill faz e quando usá-la**, com frases que o usuário falaria. Uma versão curta em inglês ajuda o acionamento.
- Nada de segredos: tokens, hosts e IDs reais nunca entram nas skills. Use placeholders como `<workspace-host>` e `<databricks-pat>`.
- Conteúdo que envelhece rápido (CLIs em versão 0.x, recursos Preview/Beta) leva a data de "Última verificação" e uma seção de fontes.
- Skills relacionadas apontam umas para as outras em vez de duplicar conteúdo.
