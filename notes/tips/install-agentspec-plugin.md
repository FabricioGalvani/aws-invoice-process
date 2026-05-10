# Como instalar o plugin AgentSpec no Claude Code (VSCode Extension)

Guia para instalar o plugin **AgentSpec** (Spec-Driven Development for Data Engineering — 58 agents, 23 KB domains, 30 commands, 5 skills) no Claude Code rodando dentro da extensão do VSCode, **sem usar `/plugin install`** (que não está disponível nesse ambiente).

## Contexto

- O comando `/plugin` da CLI **não funciona** dentro da extensão VSCode do Claude Code.
- A solução é registrar o plugin manualmente via `settings.json`.
- O plugin precisa ter a estrutura padrão do Claude Code:
  - `.claude-plugin/plugin.json`
  - `.claude-plugin/marketplace.json`

## Passo a passo

### 1. Confirme a estrutura do plugin

O diretório do plugin deve conter:

```
<caminho-do-plugin>/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── agents/
├── commands/
├── hooks/
├── skills/
└── ...
```

Anote o **nome do marketplace** que está dentro do `marketplace.json` (campo `name`). No caso do AgentSpec, é `agentspec`.

### 2. Edite o arquivo `settings.json` global

Caminho: `C:\Users\<seu-usuario>\.claude\settings.json`

Adicione os blocos `extraKnownMarketplaces` e `enabledPlugins`:

```json
{
  "autoUpdatesChannel": "latest",
  "theme": "dark-ansi",
  "model": "opus",
  "extraKnownMarketplaces": {
    "agentspec": {
      "source": {
        "source": "directory",
        "path": "C:\\Workspace\\Github\\GenAI\\agentspec\\plugin"
      }
    }
  },
  "enabledPlugins": {
    "agentspec@agentspec": true
  }
}
```

**Pontos críticos** (estes são os erros mais comuns):

1. **`source.source` deve ser `"directory"`**, não `"local"`. O schema só aceita os valores: `url`, `github`, `git`, `npm`, `file`, `directory`, `hostPattern`, `pathPattern`, `settings`.
2. **A chave em `extraKnownMarketplaces` precisa bater com o `name` que está dentro do `marketplace.json`** do plugin. Se o `marketplace.json` tem `"name": "agentspec"`, a chave aqui também precisa ser `"agentspec"`. Se usar nome diferente (ex.: `"agentspec-local"`), o Claude Code registra o marketplace com o nome real (`agentspec`) e o `enabledPlugins` não casa — o plugin nunca é habilitado.
3. **Formato do `enabledPlugins`**: `"<nome-do-plugin>@<nome-do-marketplace>": true`. Para o AgentSpec: `"agentspec@agentspec": true`.
4. **Use barras duplas `\\`** no path (JSON escape) ou barras normais `/`.

### 3. Reinicie o VSCode

Feche e reabra a janela/aba do Claude Code. O plugin é carregado apenas no startup.

### 4. Verifique a instalação

Após o restart, abra:

`C:\Users\<seu-usuario>\.claude\plugins\installed_plugins.json`

Deve listar o `agentspec` em `plugins`. Se ainda estiver `{}`, algo no settings está errado — provavelmente o problema #2 acima (mismatch de nome).

Você também pode confirmar pelos comandos disponíveis na extensão:
- `/agentspec:brainstorm`, `/agentspec:define`, `/agentspec:design`, `/agentspec:build`, `/agentspec:ship`
- `/agentspec:pipeline`, `/agentspec:schema`, `/agentspec:data-quality`, `/agentspec:sql-review`, etc.

E pelos agents auto-invocáveis (ex.: `agentspec:architect:schema-designer`, `agentspec:data-engineering:dbt-specialist`).

## Diagnóstico rápido

Se algo não funcionar, verifique nessa ordem:

| Sintoma | Causa provável | Solução |
|---------|----------------|---------|
| Erro de validação ao salvar `settings.json` | `source.source` com valor inválido (ex.: `"local"`) | Trocar para `"directory"` |
| `installed_plugins.json` continua `{}` após restart | Chave do marketplace não casa com `name` do `marketplace.json` | Renomear a chave em `extraKnownMarketplaces` para o mesmo valor do `name` |
| Comandos não aparecem mas plugin está instalado | Não reiniciou o VSCode | Fechar e reabrir |
| `claude plugin install` não funciona | Comando não disponível na extensão VSCode | Usar este método manual via `settings.json` |

## Alternativa via CLI (fora do VSCode)

Se você tiver o Claude Code CLI instalado em terminal externo (PowerShell normal):

```powershell
claude plugin marketplace add C:\Workspace\Github\GenAI\agentspec\plugin
claude plugin install agentspec@agentspec
```

Depois reabra o VSCode.

## Referências

- Plugin oficial: https://github.com/luanmorenommaciel/agentspec
- Instalação oficial via marketplace remoto:
  ```powershell
  claude plugin marketplace add luanmorenommaciel/agentspec
  claude plugin install agentspec
  ```
