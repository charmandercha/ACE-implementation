---
name: ace-core-integration
description: Integra o playbook com o sistema de processamento de mensagens, compaction e prompts. Use quando precisar que o aprendizado ocorra automaticamente durante a execução do agente.
---

# ACE Core Integration: Processor, Compaction & System

## Visão Geral

Esta skill integra o sistema de playbook com o core do agente, conectando:

1. **Processor**: Extração automática de lições após execuções de ferramentas
2. **Compaction**: Substituição do resumo monolítico por entries estruturadas
3. **System**: Injeção do playbook no system prompt

## Pré-requisitos

- Playbook module completo (skills anteriores)
- Arquivos de sessão existentes: `processor.ts`, `compaction.ts`, `system.ts`

## Integração no Processor

### Step 1: Adicione Imports

No arquivo `src/session/processor.ts`:

```typescript
import { PlaybookReflector } from "./playbook/reflector"
import { PlaybookCurator } from "./playbook/curator"
```

### Step 2: Adicione Hook após Tool Result

Encontre onde o `tool-result` é processado e adicione:

```typescript
// Inside your message processing loop
case "tool-result": {
  const completedPart = part as MessageV2.ToolPart

  // ACE: Extract lesson from completed tool execution
  const lesson = PlaybookReflector.extractLessonFromPart(completedPart)
  if (lesson) {
    // Fire-and-forget: don't block the main flow
    PlaybookCurator.addLessons([lesson]).catch((e) => {
      console.warn("playbook: failed to save lesson", { error: e.message })
    })
  }

  // ... existing tool-result handling
}
```

### Step 3: Adicione Hook após Tool Error

```typescript
case "tool-error": {
  const errorPart = part as MessageV2.ToolPart

  // ACE: Extract lesson from error
  const lesson = PlaybookReflector.extractLessonFromPart(errorPart)
  if (lesson) {
    PlaybookCurator.addLessons([lesson]).catch((e) => {
      console.warn("playbook: failed to save lesson", { error: e.message })
    })
  }

  // ... existing tool-error handling
}
```

## Integração no Compaction

### Step 1: Adicione Imports

No arquivo `src/session/compaction.ts`:

```typescript
import { PlaybookGenerator } from "./playbook/generator"
```

### Step 2: Modifique a Função de Compaction

Substitua ou modifique a função que gera o resumo:

```typescript
export async function compact(input: CompactInput): Promise<CompactOutput> {
  // ACE: Check if playbook has positive entries to use instead of LLM summary
  const playbookData = await PlaybookGenerator.generateCompactionContext()

  let promptText: string

  if (playbookData.usePlaybook) {
    // ACE: Use structured playbook entries instead of monolithic summary
    promptText = `Continue the conversation using these learned strategies:

${playbookData.context}

If these strategies are relevant to the current task, apply them. Otherwise, proceed normally.`
  } else {
    // Fallback: Use traditional LLM summary
    promptText = defaultPrompt // Your existing summary prompt
  }

  // ... rest of compaction logic
}
```

### Step 3: Alternative: Adicione Contexto ao Prompt

Se preferir injetar o contexto sem substituir o resumo:

```typescript
async function buildCompactPrompt(input: CompactInput): Promise<ModelMessage[]> {
  const playbookContext = await PlaybookGenerator.generateContextForTask(
    `session ${input.sessionID} - continuing conversation`,
  )

  const messages = [
    ...MessageV2.toModelMessages(input.messages, model),
    {
      role: "user" as const,
      content: [
        {
          type: "text" as const,
          text: playbookContext ? `${promptText}\n\n${playbookContext}` : promptText,
        },
      ],
    },
  ]

  return messages
}
```

## Integração no System Prompt

### Step 1: Adicione Imports

No arquivo `src/session/system.ts`:

```typescript
import { PlaybookGenerator } from "./playbook/generator"
import { PlaybookCache } from "./playbook/cache"
```

### Step 2: Crie Função para Injetar Playbook

```typescript
export namespace SystemPrompt {
  export async function playbook(): Promise<string> {
    try {
      const entries = await PlaybookGenerator.getPositiveEntries(5)

      if (entries.length === 0) {
        return ""
      }

      const formatted = entries
        .map((e) => {
          const score = e.helpfulCount - e.harmfulCount
          const tags = e.tags.length > 0 ? ` (${e.tags.join(", ")})` : ""
          return `•${tags} ${e.content} [score: ${score}]`
        })
        .join("\n")

      return `## Learned Strategies
These strategies have been learned from previous interactions:
${formatted}`
    } catch (e) {
      console.warn("playbook: failed to generate system context", e)
      return ""
    }
  }
}
```

### Step 3: Integre na Construção do System Prompt

```typescript
export async function buildSystemPrompt(context: SystemContext): Promise<string> {
  const parts: string[] = []

  // Environment info
  parts.push(await SystemPrompt.environment(context))

  // Provider info
  parts.push(await SystemPrompt.provider(context))

  // Instructions
  parts.push(await SystemPrompt.instructions(context))

  // ACE: Add playbook context
  const playbookContext = await SystemPrompt.playbook()
  if (playbookContext) {
    parts.push(playbookContext)
  }

  return parts.filter(Boolean).join("\n\n---\n\n")
}
```

## Configuração de Cache (Opcional)

Para otimizar o tamanho do contexto:

```typescript
// cache.ts
export namespace PlaybookCache {
  const TOKEN_LIMIT = 8000

  export function determineStrategy(entriesCount: number): "full" | "selective" | "none" {
    if (entriesCount === 0) return "none"
    // Rough token estimate: ~20 chars per entry
    if (entriesCount * 20 < TOKEN_LIMIT) return "full"
    return "selective"
  }

  export function formatStableContext(entries: PlaybookEntry[]): string {
    return entries
      .map((e) => {
        const score = e.helpfulCount - e.harmfulCount
        return `• ${e.content} [score: ${score}]`
      })
      .join("\n")
  }

  export function estimateTokens(entries: PlaybookEntry[]): number {
    return entries.reduce((sum, e) => sum + e.content.length, 0)
  }
}
```

## Fluxo Completo

```
┌─────────────────────────────────────────────────────────────────┐
│                    MESSAGE PROCESSING                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Tool executes → tool-result / tool-error                   │
│                     ↓                                           │
│  2. Reflector.extractLessonFromPart() → ExecutionLesson        │
│                     ↓                                           │
│  3. Curator.addLessons() → Persisted to playbook.json          │
│                     ↓                                           │
│  4. Session continues...                                       │
│                     ↓                                           │
│  5. Compaction triggered (context window full)                 │
│                     ↓                                           │
│  6. Generator.generateCompactionContext() → playbook entries   │
│                     ↓                                           │
│  7. Use playbook entries INSTEAD OF LLM summary                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Testes de Integração

```typescript
import { describe, test, expect, beforeEach } from "bun:test"
import { compact } from "./compaction"
import { PlaybookStorage } from "./playbook/storage"

describe("Core Integration", () => {
  beforeEach(async () => {
    await PlaybookStorage.clear()
  })

  test("compaction uses playbook when available", async () => {
    // Add a positive entry
    const playbook = await PlaybookStorage.load()
    playbook.entries.push({
      id: "pb_test",
      content: "Use Read instead of cat",
      helpfulCount: 5,
      harmfulCount: 0,
      createdAt: Date.now(),
      updatedAt: Date.now(),
      tags: ["best-practice"],
      projectId: null,
      lastRetrievedAt: 0,
      retrievalCount: 0,
    })
    await PlaybookStorage.save(playbook)

    // Compact should use playbook
    const result = await compact({
      sessionID: "test",
      messages: [],
      model: "test-model",
    })

    expect(result.usePlaybook).toBe(true)
    expect(result.context).toContain("Use Read instead of cat")
  })

  test("compaction falls back when no positive entries", async () => {
    const result = await compact({
      sessionID: "test",
      messages: [],
      model: "test-model",
    })

    expect(result.usePlaybook).toBe(false)
  })
})
```

## Referências

- Processor integration: `packages/opencode/src/session/processor.ts` (lines ~202, ~234)
- Compaction integration: `packages/opencode/src/session/compaction.ts` (lines ~154-175)
- System prompt integration: `packages/opencode/src/session/system.ts`
