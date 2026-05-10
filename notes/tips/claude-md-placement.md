# Onde colocar o `CLAUDE.md` (raiz do projeto vs `.claude/`)

Dica rápida sobre a convenção do Claude Code para o arquivo de memória/contexto persistente do projeto. Resposta curta: **`CLAUDE.md` vai na raiz do projeto, NÃO dentro de `.claude/`**.

## Contexto

O `CLAUDE.md` é o arquivo que o Claude Code carrega automaticamente em toda sessão para servir de memória/contexto persistente do projeto (decisões arquiteturais, convenções, regras, escopo). Por ser tão central, é comum confundir onde ele deve ficar.

A pasta `.claude/` existe **para configuração executável** (settings, hooks, subagents, slash commands customizados) — não para documentação de contexto.

## Convenção

| Caminho | Propósito | Vai pro git? |
|---|---|---|
| `./CLAUDE.md` (raiz do projeto) | **Memória do projeto** — carregada em toda sessão aberta nesse diretório. Compartilhada com o time inteiro. | **Sim** — commitar |
| `~/.claude/CLAUDE.md` | Memória **pessoal global** — preferências do usuário em todos os projetos da máquina. | Não — fica só na sua máquina |
| `./.claude/settings.json` | Configuração do Claude Code para este projeto (hooks, permissões, env vars, modelo). | Sim (com cuidado em segredos) |
| `./.claude/agents/`, `./.claude/commands/`, `./.claude/skills/` | Subagents, slash commands e skills customizados deste projeto. | Sim |
| `./.claude/settings.local.json` | Overrides locais de configuração (gitignored por padrão). | Não |

## Por que na raiz e não em `.claude/`?

1. **Visibilidade humana** — `CLAUDE.md` na raiz fica ao lado do `README.md`, é lido por humanos também e aparece imediatamente quando alguém abre o repo.
2. **Versionamento natural** — fica óbvio que é parte do código/documentação do projeto e deve ser revisado em PR como qualquer outro arquivo.
3. **Convenção do Claude Code** — a CLI procura `CLAUDE.md` subindo a partir do diretório atual; a raiz do projeto é o ponto canônico.
4. **Separação de responsabilidades** — `.claude/` é "como o Claude Code roda neste projeto" (executável); `CLAUDE.md` é "o que o Claude precisa saber sobre este projeto" (declarativo).

## O que colocar em `CLAUDE.md`

- Visão geral do projeto (1 parágrafo)
- Decisões arquiteturais persistentes (com data e justificativa)
- Regras impostas por essas decisões (o que fazer / o que evitar)
- Itens deprecados (não usar em código novo)
- Estrutura de pastas esperada
- Apontadores para documentação detalhada em outras pastas (`notes/`, `docs/`)

**Mantenha conciso.** Acima de ~200 linhas o conteúdo final pode ser truncado pelo Claude Code. Detalhes longos vão para arquivos linkados.

## O que NÃO colocar em `CLAUDE.md`

- Tarefas em andamento (use plano/todo da sessão)
- Histórico de mudanças passadas que não impõem regra atual (use git log)
- Documentação de API/código (essa fica no próprio código)
- Segredos, credenciais, tokens (nunca em arquivo versionado)

## Diagnóstico rápido

| Sintoma | Causa | Solução |
|---|---|---|
| Claude não parece lembrar do contexto entre sessões | `CLAUDE.md` não está na raiz ou está vazio | Criar/mover para a raiz do projeto |
| Conteúdo do `CLAUDE.md` aparece truncado | Arquivo grande demais | Reduzir; mover detalhes para `notes/` ou `docs/` linkados |
| Equipe não vê as regras arquiteturais | `CLAUDE.md` está em `.claude/` (frequentemente sub-explorado) | Mover para a raiz e commitar |
| Configuração de hooks/permissões em `CLAUDE.md` | Confusão de propósito | Mover para `.claude/settings.json` |

## Referências

- Documentação Claude Code — Memory: https://docs.claude.com/en/docs/claude-code/memory
- Estrutura de plugin/configuração: ver `notes/tips/install-agentspec-plugin.md`
