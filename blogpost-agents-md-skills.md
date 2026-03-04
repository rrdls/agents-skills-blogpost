# AGENTS.md e Agent Skills: Por Debaixo dos Panos dos AI Coding Agents

---

## Introdução: O README que a IA Lê

Se você já usou um AI coding agent (Claude Code, Codex, Cursor, OpenCode...), provavelmente percebeu que ele comete erros que qualquer dev do time não cometeria: usa o formatter errado, ignora convenções de nomenclatura, roda testes com o comando incorreto. O problema é simples: **o agente não conhece o seu projeto**.

O `README.md` existe para humanos. Ele explica o que o projeto faz, como instalar, como contribuir. Mas um agente de IA precisa de informações diferentes: qual o comando de build? Qual o style guide? Como rodar um teste individual? Quais padrões arquiteturais seguir?

A resposta que a comunidade encontrou: criar um **"README para máquinas"**. É disso que se trata o `AGENTS.md` e todo o ecossistema de **Agent Skills** que está surgindo em torno dele.

Neste post, vamos além da documentação superficial. Vamos **abrir o código-fonte** de um AI coding agent real (OpenCode) e entender exatamente:

1. Como o `AGENTS.md` é descoberto e injetado no prompt de sistema
2. Como as **Skills** funcionam via **progressive disclosure**
3. O que a literatura científica diz sobre a efetividade real desses arquivos

---

## 1. O que é o AGENTS.md?

O `AGENTS.md` é um arquivo Markdown que fica na raiz do seu repositório (ou em subdiretórios, para monorepos) e contém instruções direcionadas a agentes de IA. Ele surgiu como uma evolução natural dos arquivos proprietários que cada ferramenta criava:

| Ferramenta | Arquivo proprietário |
|---|---|
| Claude Code | `CLAUDE.md` |
| Cursor | `.cursor/rules/` |
| Windsurf | Memories / Rules |
| Copilot | `.github/copilot-instructions.md` |

O problema? Cada ferramenta lia seu próprio arquivo, criando fragmentação. O `AGENTS.md` nasceu como um **padrão aberto e agnóstico de ecossistema**, e hoje é suportado por mais de 20 ferramentas, incluindo:

**Codex** (OpenAI), **Cursor**, **Antigravity** (Google DeepMind), **Jules** (Google), **GitHub Copilot Coding Agent**, **Windsurf**, **OpenCode**, **Aider**, **Zed**, **Warp**, **RooCode**, **Gemini CLI**, **Devin**, e outros.

Segundo o site oficial [agents.md](https://agents.md), mais de **60.000 repositórios públicos** no GitHub já incluem um context file.

### O que vai dentro de um AGENTS.md?

O conteúdo típico inclui:

```markdown
# AGENTS.md

## Build & Test
- Build: `npm run build`
- Test all: `npm test`
- Test single: `npm test -- --grep "test name"`
- Lint: `npm run lint`

## Code Style
- Use TypeScript strict mode
- Prefer `const` over `let`
- Avoid `try/catch` where possible
- Single-word variable names preferred

## Architecture
- API handlers in `src/api/handlers/`
- Database models in `src/models/`
- Avoid circular dependencies

## Testing
- Avoid mocks when possible
- Test actual implementation
```

O princípio central: **ser conciso e prático**. Nada de textos longos explicando a filosofia do projeto. O agente precisa de regras claras e comandos executáveis.

### Hierarquia: do global ao local

Uma feature poderosa é a **hierarquia de arquivos**. No OpenCode, por exemplo:

```
~/.config/opencode/AGENTS.md          ← Global (todas as sessões)
./AGENTS.md                            ← Projeto (raiz do repo)
./packages/frontend/AGENTS.md          ← Pacote específico (monorepo)
```

O arquivo mais próximo tem prioridade. Isso permite que um monorepo tenha regras globais na raiz e regras específicas em cada pacote.

---

## 2. Como o AGENTS.md Funciona Sob o Capô

Aqui é onde abrimos o código-fonte. Vamos usar o **OpenCode** como case, pois é open source e tem uma implementação limpa e bem documentada.

### 2.1 Descoberta: encontrando o arquivo

O primeiro passo é **descobrir** quais arquivos de instrução existem. No OpenCode, isso acontece no arquivo [`instruction.ts`](https://github.com/anomalyco/opencode):

```typescript
// Arquivos que o sistema procura, em ordem de prioridade
const FILES = [
  "AGENTS.md",
  "CLAUDE.md",
  "CONTEXT.md", // deprecated
]
```

A busca acontece em duas frentes simultâneas:

**Frente 1: Configuração global do usuário.** O sistema primeiro verifica se o desenvolvedor tem um AGENTS.md "pessoal", que vale para todos os projetos. Isso fica em diretórios de configuração do sistema operacional (como `~/.config/opencode/`). Pense nisso como as suas "preferências universais": estilo de código que você gosta, ferramentas que sempre usa, etc.

```typescript
function globalFiles() {
  const files = []
  files.push(path.join(Global.Path.config, "AGENTS.md"))
  // Compatibilidade: também busca CLAUDE.md do Claude Code
  if (!Flag.OPENCODE_DISABLE_CLAUDE_CODE_PROMPT) {
    files.push(path.join(os.homedir(), ".claude", "CLAUDE.md"))
  }
  return files
}
```

**Frente 2: Contexto do projeto.** Em paralelo, o sistema percorre a árvore de diretórios **de baixo pra cima** (do diretório de trabalho atual até a raiz do repositório), coletando todos os AGENTS.md que encontrar pelo caminho. Isso é poderoso em monorepos: se você está trabalhando em `packages/frontend/`, o sistema carrega tanto o `./packages/frontend/AGENTS.md` (regras do frontend) quanto o `./AGENTS.md` (regras globais do projeto).

```typescript
// Busca hierárquica: sobe do diretório atual até a raiz do worktree
const matches = await Filesystem.findUp(file, Instance.directory, Instance.worktree)
```

**Regra de prioridade**: quando há conflito, o arquivo mais próximo (mais específico) vence. E `AGENTS.md` tem prioridade sobre `CLAUDE.md`, que tem prioridade sobre `CONTEXT.md`.

### 2.2 Injeção: colocando no prompt de sistema

Uma vez descobertos, os arquivos são **lidos e injetados diretamente no system prompt** do LLM. No loop principal do agente (`prompt.ts`):

```typescript
// Linha 652 — Montagem do system prompt
const system = [
  ...await SystemPrompt.environment(model),  // Info do ambiente (OS, diretório, data)
  ...await InstructionPrompt.system()         // ← AGENTS.md entra aqui
]
```

O conteúdo do `AGENTS.md` é prefixado com `"Instructions from: /caminho/do/AGENTS.md"` e concatenado ao prompt de sistema. Isso significa que **toda mensagem** que o LLM processa já contém as instruções do seu repositório.

### 2.3 Lazy Loading: instruções aninhadas sob demanda

Aqui fica interessante. O OpenCode implementa um **carregamento contextual sob demanda**: quando o agente lê um arquivo em um subdiretório (por exemplo, `src/components/Button.tsx`), o sistema verifica se existe um `AGENTS.md` **naquele subdiretório** e, se existir, injeta seu conteúdo automaticamente:

```typescript
export async function resolve(messages, filepath, messageID) {
  let current = path.dirname(filepath)
  const root = path.resolve(Instance.directory)

  // Sobe do diretório do arquivo até a raiz
  while (current.startsWith(root) && current !== root) {
    const found = await find(current)
    // Injeta se: existe, não é o arquivo raiz, e não foi carregado antes
    if (found && !system.has(found) && !already.has(found)) {
      const content = await Filesystem.readText(found)
      results.push({ filepath: found, content })
    }
    current = path.dirname(current)
  }
}
```

Isso é elegante: o agente **só recebe as instruções de um subdiretório quando realmente está trabalhando nele**. Sem poluir o contexto desnecessariamente.

---

## 3. Agent Skills e Progressive Disclosure

Se o `AGENTS.md` é o "manual do projeto", as **Agent Skills** são um conceito mais sofisticado: **blocos modulares de comportamento** que o agente pode carregar sob demanda.

### 3.1 O que é uma Skill?

Uma skill é um diretório contendo um arquivo `SKILL.md` com instruções especializadas. O formato segue YAML frontmatter + corpo Markdown:

```markdown
---
name: git-release
description: Create consistent releases and changelogs
---

## What I do
- Draft release notes from merged PRs
- Propose a version bump
- Provide a copy-pasteable `gh release create` command

## When to use me
Use this when you are preparing a tagged release.
```

Skills podem conter **arquivos adicionais** como scripts, templates e referências:

```
my-skill/
├── SKILL.md           # Instruções principais (obrigatório)
├── reference.md       # Docs detalhados (carregados sob demanda)
├── examples/
│   └── sample.md      # Exemplos (carregados sob demanda)
└── scripts/
    └── helper.py      # Script utilitário (executado, não carregado)
```

### 3.2 O Padrão: Progressive Disclosure

Um recurso importante das Skills é como elas implementam o **progressive disclosure**, um padrão de design de UI emprestado para a arquitetura de agentes de IA. A ideia:

> Em vez de carregar tudo no contexto do LLM de uma vez (o que consome tokens, aumenta custo e pode confundir o modelo), revele informações **gradualmente**, conforme a necessidade.

O mecanismo funciona em **três camadas**:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   LAYER 1: INDEXAÇÃO (sempre no prompt)                 │
│   ┌───────────────────────────────────────────────┐     │
│   │ skill: "git-release"                          │     │
│   │ description: "Create consistent releases..."  │     │
│   └───────────────────────────────────────────────┘     │
│                                                         │
│   LAYER 2: ATIVAÇÃO (quando o modelo chama a tool)      │
│   ┌───────────────────────────────────────────────┐     │
│   │ Conteúdo completo do SKILL.md                 │     │
│   │ (instruções, workflows, regras)               │     │
│   └───────────────────────────────────────────────┘     │
│                                                         │
│   LAYER 3: REFERÊNCIA (sob demanda)                     │
│   ┌───────────────────────────────────────────────┐     │
│   │ Scripts, templates, exemplos, docs detalhados │     │
│   │ (lidos pelo agente quando necessário)          │     │
│   └───────────────────────────────────────────────┘     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Por que isso importa?** A janela de contexto de um LLM é um recurso finito e precioso. Cada token desperdiçado com informação irrelevante é um token a menos para raciocínio. Progressive disclosure resolve isso: o modelo sabe *o que* existe (Layer 1), carrega *como* usar quando decidir (Layer 2), e acessa detalhes profundos *se* precisar (Layer 3).

### 3.3 Deep Dive: Como o OpenCode Implementa Skills

#### Descoberta de Skills (`skill.ts`)

O algoritmo de descoberta é multi-diretório e multi-escopo:

```typescript
// Diretórios externos compatíveis
const EXTERNAL_DIRS = [".claude", ".agents"]
const EXTERNAL_SKILL_PATTERN = "skills/**/SKILL.md"
const OPENCODE_SKILL_PATTERN = "{skill,skills}/**/SKILL.md"
```

A ordem de busca garante que **projeto sobrescreve global**:

```
1. Global: ~/.claude/skills/*/SKILL.md
2. Global: ~/.agents/skills/*/SKILL.md
3. Projeto: .claude/skills/*/SKILL.md (sobe hierarquicamente)
4. Projeto: .agents/skills/*/SKILL.md (sobe hierarquicamente)
5. Config: .opencode/skills/*/SKILL.md
6. Custom: caminhos configurados em opencode.json
7. Remote: skills baixadas via URL (index.json)
```

A compatibilidade cross-tool é intencional: uma skill criada para Claude Code em `.claude/skills/` funciona automaticamente no OpenCode.

#### A Tool `skill` como mecanismo de disclosure (`tool/skill.ts`)

A mágica acontece na definição da **tool `skill`**. O OpenCode injeta o **índice de todas as skills disponíveis** diretamente na **descrição da ferramenta**:

```typescript
const description = [
  "Load a specialized skill that provides domain-specific instructions.",
  "",
  "Available skills:",
  "",
  "<available_skills>",
  ...accessibleSkills.flatMap((skill) => [
    `  <skill>`,
    `    <name>${skill.name}</name>`,
    `    <description>${skill.description}</description>`,
    `  </skill>`,
  ]),
  "</available_skills>",
].join("\n")
```

Isso é a **Layer 1**: o modelo vê na descrição da tool quais skills existem e o que fazem. Ele ainda não carregou nenhuma, isso consumiu apenas alguns tokens para os metadados.

Quando o modelo decide que precisa de uma skill, ele **chama a tool**:

```
skill({ name: "git-release" })
```

E a tool retorna o **conteúdo completo** (Layer 2) junto com uma lista dos **arquivos disponíveis** no diretório da skill (Layer 3):

```typescript
return {
  title: `Loaded skill: ${skill.name}`,
  output: [
    `<skill_content name="${skill.name}">`,
    `# Skill: ${skill.name}`,
    skill.content.trim(),                    // ← Layer 2: corpo do SKILL.md
    `Base directory: ${base}`,
    `<skill_files>`,
    files,                                    // ← Layer 3: arquivos disponíveis
    `</skill_files>`,
    `</skill_content>`,
  ].join("\n"),
}
```

Os arquivos da Layer 3 (scripts, exemplos, referências) são **listados** mas não carregados. O modelo pode então usar a tool de leitura de arquivos para acessá-los se necessário.

#### Skills Remotas (`discovery.ts`)

O OpenCode vai além e suporta **registries de skills remotas**. Você pode configurar uma URL que aponta para um `index.json`:

```json
{
  "skills": [
    {
      "name": "cloudflare",
      "description": "Cloudflare platform skill",
      "files": ["SKILL.md", "references/workers.md", "references/d1.md"]
    }
  ]
}
```

O sistema faz download, cache local, e as skills ficam disponíveis como qualquer outra. Isso abre a porta para **skill marketplaces** e compartilhamento entre times.

---

## 4. O Fluxo Completo: Da Inicialização à Execução

Vamos consolidar tudo em um fluxo narrativo. Quando você inicia uma sessão com um AI coding agent:

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. INICIALIZAÇÃO                                                 │
│    ┌──────────────────────────────────────────────────┐          │
│    │ Sistema descobre AGENTS.md (global + projeto)    │          │
│    │ Sistema descobre Skills (.opencode/, .claude/,   │          │
│    │   .agents/, URLs remotas)                        │          │
│    └──────────────────────────────────────────────────┘          │
│                         │                                        │
│                         ▼                                        │
│ 2. MONTAGEM DO PROMPT                                            │
│    ┌──────────────────────────────────────────────────┐          │
│    │ System Prompt =                                  │          │
│    │   Prompt base do modelo (Anthropic, GPT, etc.)   │          │
│    │   + Info do ambiente (OS, diretório, data)       │          │
│    │   + Conteúdo do AGENTS.md                        │          │
│    │                                                  │          │
│    │ Tools =                                          │          │
│    │   read, write, bash, grep, ...                   │          │
│    │   + skill (com índice das skills na descrição)   │          │
│    └──────────────────────────────────────────────────┘          │
│                         │                                        │
│                         ▼                                        │
│ 3. INTERAÇÃO DO USUÁRIO                                          │
│    ┌──────────────────────────────────────────────────┐          │
│    │ Usuário: "Deploy this to Cloudflare Workers"     │          │
│    └──────────────────────────────────────────────────┘          │
│                         │                                        │
│                         ▼                                        │
│ 4. DECISÃO DO MODELO                                             │
│    ┌──────────────────────────────────────────────────┐          │
│    │ Modelo vê: skill "cloudflare" disponível         │          │
│    │ → Chama: skill({ name: "cloudflare" })           │          │
│    │ → Recebe: instruções completas + arquivos        │          │
│    └──────────────────────────────────────────────────┘          │
│                         │                                        │
│                         ▼                                        │
│ 5. EXECUÇÃO INFORMADA                                            │
│    ┌──────────────────────────────────────────────────┐          │
│    │ Modelo executa a tarefa com as instruções da     │          │
│    │ skill, seguindo as convenções do AGENTS.md       │          │
│    └──────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────────┘
```

O ponto central: **o modelo opera em dois níveis de contexto simultâneos**. O `AGENTS.md` fornece o "como trabalhamos aqui" (sempre presente), enquanto as Skills fornecem o "como fazer isso especificamente" (sob demanda).

---

## 5. Panorama: Como Cada Ferramenta Implementa

Embora o conceito seja similar, cada ferramenta tem suas particularidades:

| Aspecto | Claude Code | OpenCode | Cursor | Codex | Windsurf |
|---------|-------------|----------|--------|-------|----------|
| **Arquivo de contexto** | `CLAUDE.md` | `AGENTS.md` (+ CLAUDE.md compat) | `.cursor/rules/` | `AGENTS.md` | Memories + Rules |
| **Skills (SKILL.md)** | Sim, completo | Sim, completo | Parcial (via regras) | Via MCP | Via Workflows |
| **Progressive Disclosure** | 3 camadas | 3 camadas | Direto no prompt | Parcial | Direto no prompt |
| **Hierarquia de arquivos** | Sim (findUp) | Sim (findUp) | Sim (file matchers) | Sim | Sim |
| **Geração automática** | `/init` | `/init` | Via LLM | `/init` | Via LLM |
| **Skills remotas** | Plugins | URLs (index.json) | N/A | N/A | N/A |
| **Compatibilidade AGENTS.md** | Lê como fallback | Nativo + CLAUDE.md compat | Sim | Nativo | Sim |

**Destaque**: o OpenCode implementa compatibilidade bidirecional: lê tanto `AGENTS.md` quanto `CLAUDE.md`, e busca skills em `.claude/skills/`, `.agents/skills/`, e `.opencode/skills/`. Isso significa que se você já tem uma config para Claude Code, ela funciona automaticamente no OpenCode.

---

## 6. O que a Literatura Científica Diz: O Caso Contra-Intuitivo

Em fevereiro de 2026, pesquisadores da ETH Zurich publicaram o paper **"Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?"** (Gloaguen, Mündler et al.), que é o primeiro estudo rigoroso sobre a efetividade real dos context files.

### O Benchmark: AGENT BENCH

Os pesquisadores criaram o **AGENT BENCH**, um benchmark com:
- **138 instâncias** de tarefas reais (bug fixes e features)
- **12 repositórios** recentes e de nicho (com context files escritos por devs reais)
- Cobertura média de testes de **75%**
- Avaliação em 4 modelos: Sonnet 4.5, GPT-5.2, GPT-5.1 Mini, e Qwen3-30B

### Os Achados (Surpreendentes)

Os resultados desafiam a intuição:

| Cenário | Efeito na taxa de sucesso | Custo |
|---------|---------------------------|-------|
| **Sem context file** | Baseline | Baseline |
| **Context file gerado por LLM** | **-3%** (piora) | **+20%** |
| **Context file escrito por humano** | **+4%** (melhora marginal) | **+20%** |

Sim, **context files gerados automaticamente pelo agente (via `/init`) tendem a piorar o desempenho**, enquanto os escritos por humanos oferecem apenas uma melhoria marginal de ~4%.

### Por que isso acontece?

A análise de traces revelou que:

1. **Mais exploração**: com context files, os agentes executam mais `grep`, leem mais arquivos, e rodam mais testes
2. **Mais raciocínio**: o número de tokens de raciocínio sobe 14-22% (o modelo "pensa mais" para seguir as instruções)
3. **Instruções seguidas**: os agentes realmente seguem as instruções (e.g., usam `uv` quando mencionado), mas isso nem sempre ajuda
4. **Redundância**: context files gerados por LLM são altamente redundantes com a documentação existente

O insight mais valioso: quando os pesquisadores **removeram toda a documentação** (README, docs/, exemplos) e deixaram apenas o context file, os arquivos gerados por LLM **passaram a ajudar** (+2.7%). Isso sugere que context files são mais úteis em **repositórios com pouca ou nenhuma documentação**.

### A Recomendação

Os autores concluem:

> "Omitir context files gerados por LLM por enquanto, contrariamente às recomendações dos desenvolvedores de agentes, e incluir apenas requisitos mínimos (e.g., tooling específico para usar com o repositório)."

Traduzindo: **menos é mais**. Um AGENTS.md com 10 linhas de build commands é mais útil que um de 200 linhas gerado automaticamente.

---

## Conclusão

O ecossistema de `AGENTS.md` e Agent Skills representa um momento fascinante na engenharia de software. Estamos essencialmente criando uma **camada de comunicação entre humanos e agentes de IA** no nível do repositório.

Os mecanismos por debaixo dos panos revelam engenharia sofisticada:

- **AGENTS.md** é injetado no system prompt como "regras fundamentais do projeto"
- **Skills** usam **progressive disclosure** em 3 camadas para otimizar o uso da janela de contexto
- **Compatibilidade cross-tool** permite que uma mesma configuração funcione em múltiplos agentes
- **Skills remotas** abrem caminho para marketplaces e compartilhamento entre times

O artigo da ETH Zurich nos lembra, porém, que **a qualidade importa mais que a existência**. Criar um AGENTS.md gigante e genérico pode ser contraproducente. A recomendação prática:

1. **Seja mínimo**: inclua apenas build commands, test commands, e as regras de estilo mais importantes
2. **Seja específico**: prefira "Use `ruff` para formatting" a "Formate seu código adequadamente"
3. **Use Skills para conhecimento especializado**: em vez de colocar tudo no AGENTS.md, crie skills separadas para workflows específicos
4. **Aplique progressive disclosure**: mantenha o SKILL.md conciso e mova referências detalhadas para arquivos separados
5. **Escreva manualmente**: context files escritos por humanos são mais efetivos que os gerados por `/init`

O futuro provavelmente envolverá **context files mais inteligentes**: gerados dinamicamente, adaptados à tarefa atual, e otimizados para minimizar o overhead cognitivo do modelo. Enquanto isso, o conselho mais valioso permanece simples: **conheça o seu projeto e diga ao agente apenas o essencial**.

---

## Fontes

### Especificação e Padrão
1. **AGENTS.md** - Site oficial da especificação — [agents.md](https://agents.md)
2. **AGENTS.md no GitHub** — [github.com/anthropics/agents-md](https://github.com/anthropics/agents-md)

### Documentação Oficial de Ferramentas
3. **Anthropic - Claude Code Skills** — [docs.anthropic.com/en/docs/claude-code/skills](https://docs.anthropic.com/en/docs/claude-code/skills)
4. **Anthropic - Claude Code Memory (CLAUDE.md)** — [docs.anthropic.com/en/docs/claude-code/memory](https://docs.anthropic.com/en/docs/claude-code/memory)
5. **OpenCode - Agents** — [opencode.ai/docs/agents](https://opencode.ai/docs/agents)
6. **OpenCode - Skills** — [opencode.ai/docs/skills](https://opencode.ai/docs/skills)
7. **OpenAI - Codex** — [github.com/openai/codex](https://github.com/openai/codex)

### Código-Fonte Analisado
8. **OpenCode (repositório)** — [github.com/anomalyco/opencode](https://github.com/anomalyco/opencode)
   - `packages/opencode/src/session/instruction.ts` — Carregamento de AGENTS.md
   - `packages/opencode/src/session/prompt.ts` — Loop do agente e montagem do prompt
   - `packages/opencode/src/skill/skill.ts` — Descoberta de skills
   - `packages/opencode/src/tool/skill.ts` — Tool de progressive disclosure
   - `packages/opencode/src/skill/discovery.ts` — Skills remotas

### Artigos Acadêmicos
9. **Gloaguen, T.; Mündler, N. et al.** "Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?" ETH Zurich, Preprint, fevereiro 2026. arXiv: `2602.11988v1`
10. **Chatlatanagulchai, W. et al.** "Agent READMEs: An Empirical Study of Context Files for Agentic Coding." arXiv: `2511.12884`, 2025.
11. **Mohsenimofidi, S. et al.** "Context Engineering for AI Agents in Open-Source Software." arXiv: `2510.21413`, 2025.

### Blog Posts e Artigos
12. **Boyina, G.** "Why I Created AGENTS.md: A Simple Solution to a Growing Problem" — [Medium](https://thegowtham.medium.com/why-i-created-agents-md)
13. **Sewell, S.** "Improve your AI code output with AGENTS.md" — [builder.io/blog/agents-md](https://www.builder.io/blog/agents-md)
14. **Nigh, M.** "How to write a great agents.md: Lessons from over 2,500 repositories" — [GitHub Blog](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
15. **Sawers, P.** "The Rise of Agents.md, an Open Standard" — [tessl.io](https://tessl.io/blog/the-rise-of-agents-md-an-open-standard-and-single-source-of-truth-for-ai-coding-agents/)
16. **InfoQ** — Cobertura sobre AGENTS.md e padronização
17. **Towards AI / Towards Data Science** — Artigos sobre progressive disclosure em AI agents
