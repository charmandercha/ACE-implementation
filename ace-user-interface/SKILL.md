---
name: ace-user-interface
description: Implementa interface de comandos slash para interação com o playbook. Use quando quiser que usuários possam gerenciar entries manualmente via comandos.
---

# ACE User Interface: Slash Commands

## Visão Geral

Esta skill adiciona uma interface de linha de comando para gerenciar o playbook:

- Listar entries
- Marcar entries como úteis/prejudiciais
- Ver detalhes de uma entry
- Limpar o playbook
- Ver estatísticas

## Pré-requisitos

- Playbook module completo (skills anteriores)
- Sistema de prompts/processamento existente

## Arquitetura

```
src/session/playbook/
└── commands.ts    # Handlers de comandos
```

### Step 1: Crie o Módulo de Comandos (commands.ts)

```typescript
import { PlaybookStorage } from "./storage"
import { PlaybookCurator } from "./curator"
import { PlaybookGenerator } from "./generator"

export interface CommandResult {
  response: string
  done: boolean // true = don't call LLM
}

export namespace PlaybookCommands {
  export async function handleCommand(input: string): Promise<CommandResult | null> {
    const parts = input.trim().split(/\s+/)
    const command = parts[0]?.toLowerCase()
    const args = parts.slice(1)

    switch (command) {
      case "/playbook":
        return await handlePlaybookCommand(args)
      case "/pb": // Shortcut
        return await handlePlaybookCommand(args)
      default:
        return null
    }
  }

  async function handlePlaybookCommand(args: string[]): Promise<CommandResult> {
    const subcommand = args[0]?.toLowerCase()

    switch (subcommand) {
      case "list":
        return await cmdList(args[1])
      case "show":
        return await cmdShow(args[1])
      case "helpful":
        return await cmdHelpful(args[1])
      case "harmful":
        return await cmdHarmful(args[1])
      case "stats":
        return await cmdStats()
      case "clear":
        return await cmdClear()
      case "help":
        return cmdHelp()
      default:
        return cmdHelp()
    }
  }

  async function cmdList(limitArg?: string): Promise<CommandResult> {
    const limit = parseInt(limitArg || "10", 10)
    const playbook = await PlaybookStorage.load()

    const entries = playbook.entries
      .sort((a, b) => b.helpfulCount - b.harmfulCount - (a.helpfulCount - a.harmfulCount))
      .slice(0, limit)

    if (entries.length === 0) {
      return {
        response: "📚 Playbook está vazio. Execute algumas tarefas para começar a aprender!",
        done: true,
      }
    }

    const lines = entries.map((e) => {
      const score = e.helpfulCount - e.harmfulCount
      const scoreEmoji = score > 0 ? "✅" : score < 0 ? "❌" : "⚪"
      const tags = e.tags.length > 0 ? ` [${e.tags.join(", ")}]` : ""
      return `${scoreEmoji} [\`${e.id}\`]${tags}\n   ${e.content.slice(0, 80)}...`
    })

    return {
      response: `📚 **Playbook** (${entries.length} entries)\n\n${lines.join("\n\n")}`,
      done: true,
    }
  }

  async function cmdShow(id?: string): Promise<CommandResult> {
    if (!id) {
      return { response: "❌ Usage: /playbook show <id>", done: true }
    }

    const playbook = await PlaybookStorage.load()
    const entry = playbook.entries.find((e) => e.id === id)

    if (!entry) {
      return { response: `❌ Entry não encontrada: ${id}`, done: true }
    }

    const score = entry.helpfulCount - entry.harmfulCount
    const created = new Date(entry.createdAt).toLocaleDateString()
    const updated = new Date(entry.updatedAt).toLocaleDateString()

    return {
      response: `**Entry**: \`${entry.id}\`

**Content**: ${entry.content}

**Score**: ${score} (✅ ${entry.helpfulCount} / ❌ ${entry.harmfulCount})
**Tags**: ${entry.tags.join(", ") || "none"}
**Created**: ${created}
**Updated**: ${updated}
**Retrievals**: ${entry.retrievalCount}`,
      done: true,
    }
  }

  async function cmdHelpful(id?: string): Promise<CommandResult> {
    if (!id) {
      return { response: "❌ Usage: /playbook helpful <id>", done: true }
    }

    try {
      await PlaybookCurator.updateWithFeedback([{ entryId: id, helpful: true }])
      return { response: `✅ Marcado como útil: \`${id}\``, done: true }
    } catch (e) {
      return { response: `❌ Erro: ${e instanceof Error ? e.message : "Unknown"}`, done: true }
    }
  }

  async function cmdHarmful(id?: string): Promise<CommandResult> {
    if (!id) {
      return { response: "❌ Usage: /playbook harmful <id>", done: true }
    }

    try {
      await PlaybookCurator.updateWithFeedback([{ entryId: id, helpful: false }])
      return { response: `❌ Marcado como prejudicial: \`${id}\``, done: true }
    } catch (e) {
      return { response: `❌ Erro: ${e instanceof Error ? e.message : "Unknown"}`, done: true }
    }
  }

  async function cmdStats(): Promise<CommandResult> {
    const playbook = await PlaybookStorage.load()

    const total = playbook.entries.length
    const positive = playbook.entries.filter((e) => e.helpfulCount > e.harmfulCount).length
    const negative = playbook.entries.filter((e) => e.harmfulCount > e.helpfulCount).length
    const neutral = playbook.entries.filter((e) => e.helpfulCount === e.harmfulCount).length
    const totalHelpful = playbook.entries.reduce((sum, e) => sum + e.helpfulCount, 0)
    const totalHarmful = playbook.entries.reduce((sum, e) => sum + e.harmfulCount, 0)

    return {
      response: `📊 **Playbook Stats**

- Total entries: ${total}
- ✅ Positivas: ${positive}
- ❌ Negativas: ${negative}
- ⚪ Neutras: ${neutral}
- 👍 Total helpful marks: ${totalHelpful}
- 👎 Total harmful marks: ${totalHarmful}`,
      done: true,
    }
  }

  async function cmdClear(): Promise<CommandResult> {
    await PlaybookStorage.clear()
    return { response: "🗑️ Playbook limpo!", done: true }
  }

  function cmdHelp(): CommandResult {
    return {
      response: `📚 **Playbook Commands**

/playbook list [limit] - Listar entries (padrão: 10)
/playbook show <id>    - Mostrar details de uma entry
/playbook helpful <id> - Marcar como útil
/playbook harmful <id> - Marcar como prejudicial
/playbook stats        - Ver estatísticas
/playbook clear       - Limpar tudo
/playbook help        - Esta ajuda`,
      done: true,
    }
  }
}
```

### Step 2: Integre no Processamento de Prompts

No arquivo `src/session/prompt.ts` (ou equivalente):

```typescript
import { PlaybookCommands } from "./playbook/commands"

export async function processPrompt(input: PromptInput): Promise<PromptOutput> {
  // Check for playbook commands first
  const commandResult = await PlaybookCommands.handleCommand(input.text)

  if (commandResult?.done) {
    return {
      content: commandResult.response,
      done: true,
    }
  }

  // ... normal prompt processing
}
```

### Step 3: Alternative - Integre via Chat Handler

Se você tem um handler de chat/mensagens:

```typescript
import { PlaybookCommands } from "./playbook/commands"

async function handleUserMessage(message: string): Promise<string | null> {
  // Trim and check for command
  const trimmed = message.trim()

  if (trimmed.startsWith("/playbook ") || trimmed === "/playbook") {
    const result = await PlaybookCommands.handleCommand(trimmed)
    return result.response
  }

  // Not a playbook command, continue normal processing
  return null
}
```

## Comandos Disponíveis

| Comando                  | Descrição                       |
| ------------------------ | ------------------------------- |
| `/playbook list [n]`     | Lista as n entries mais úteis   |
| `/playbook show <id>`    | Mostra detalhes de uma entry    |
| `/playbook helpful <id>` | Marca entry como útil           |
| `/playbook harmful <id>` | Marca entry como prejudicial    |
| `/playbook stats`        | Mostra estatísticas do playbook |
| `/playbook clear`        | Limpa todas as entries          |
| `/playbook help`         | Mostra ajuda                    |

## Exemplo de Uso

```
User: /playbook list
Bot: 📚 Playbook (10 entries)

✅ [pb_abc123] [best-practice]
   Use Read instead of cat for file operations...

✅ [pb_def456] [bash]
   Use && to chain dependent commands...

❌ [pb_ghi789] [error]
   Don't ignore error output...


User: /playbook helpful pb_abc123
Bot: ✅ Marcado como útil: pb_abc123


User: /playbook stats
Bot: 📊 Playbook Stats

- Total entries: 47
- ✅ Positivas: 38
- ❌ Negativas: 4
- ⚪ Neutras: 5
- 👍 Total helpful marks: 156
- 👎 Total harmful marks: 12
```

## Atualização no Generator

O generator deve incluir IDs no output para facilitar feedback:

```typescript
export function formatForPrompt(entries: PlaybookEntry[]): string {
  if (entries.length === 0) return ""

  const formatted = entries.map((e) => {
    const status = e.helpfulCount > e.harmfulCount ? "[HELPFUL]" : "[MAYBE HARMFUL]"
    const tags = e.tags.length > 0 ? ` (${e.tags.join(", ")})` : ""
    return `${status}${tags} [${e.id}]\n- ${e.content}`
  })

  return `## Relevant Playbook Entries
${formatted.join("\n\n")}

Use /playbook helpful [id] or /playbook harmful [id] to provide feedback.`
}
```

## Testes

```typescript
import { describe, test, expect, beforeEach } from "bun:test"
import { PlaybookCommands } from "./commands"
import { PlaybookStorage } from "./storage"

describe("Playbook Commands", () => {
  beforeEach(async () => {
    await PlaybookStorage.clear()
  })

  test("handles help command", async () => {
    const result = await PlaybookCommands.handleCommand("/playbook help")
    expect(result?.done).toBe(true)
    expect(result?.response).toContain("Playbook Commands")
  })

  test("handles stats command", async () => {
    const result = await PlaybookCommands.handleCommand("/playbook stats")
    expect(result?.done).toBe(true)
    expect(result?.response).toContain("Total entries: 0")
  })

  test("handles list command", async () => {
    const result = await PlaybookCommands.handleCommand("/playbook list")
    expect(result?.done).toBe(true)
    expect(result?.response).toContain("entries")
  })

  test("returns null for unknown commands", async () => {
    const result = await PlaybookCommands.handleCommand("/unknown")
    expect(result).toBeNull()
  })
})
```

## Referências

- Commands module: `packages/opencode/src/session/playbook/commands.ts`
- Generator update: `packages/opencode/src/session/playbook/generator.ts`
