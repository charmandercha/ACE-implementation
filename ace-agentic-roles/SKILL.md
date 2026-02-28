---
name: ace-agentic-roles
description: Implementa os três papéis agênticos do ACE: Reflector (extração de lições), Curator (aplicação de deltas) e Generator (recuperação de contexto). Use quando precisar que o agente aprenda automaticamente com execuções de ferramentas.
---

# ACE Agentic Roles: Reflector, Curator & Generator

## Visão Geral

Esta skill implementa o núcleo do sistema ACE, seguindo a arquitetura de três componentes descritos no paper:

1. **Generator**: Produz reasoning trajectories e busca entradas relevantes
2. **Reflector**: Analisa execuções e extrai lições estruturadas
3. **Curator**: Aplica deltas incrementais ao playbook

## Pré-requisitos

- Schema e Storage implementados (ver skill `ace-foundation`)
- Acesso ao modelo LLM para extração de lições (opcional, com fallback)
- Biblioteca `zod` instalada

## Arquitetura

### Estrutura de Arquivos

```
src/session/playbook/
├── reflector.ts    # Extração de lições
├── curator.ts     # Aplicação de deltas
└── generator.ts   # Recuperação de contexto
```

### Step 1: Implemente o Reflector (reflector.ts)

O Reflector analisa os resultados de ferramentas e extrai lições estruturadas.

```typescript
import { MessageV2 } from "../message-v2" // Ajuste para seu projeto

export interface ExecutionLesson {
  content: string
  tags: string[]
  isPositive: boolean
  tool?: string
  context?: string
}

export namespace PlaybookReflector {
  // Padrões de erro para detecção heurística
  const ERROR_PATTERNS = [
    /^error:/i,
    /^Error:/i,
    /failed/i,
    /fatal/i,
    /exception/i,
    /traceback/i,
    /^Command.+/i,
    /exited with code [1-9]/i,
    /SyntaxError/i,
    /TypeError/i,
    /ReferenceError/i,
    /^ENOENT/i,
    /^EADDRINUSE/i,
    /permission denied/i,
    /not found/i,
    /no such file/i,
    /cannot find/i,
    /undefined is not a/i,
    /cannot read properties of/i,
  ]

  export function extractLessonFromPart(part: MessageV2.Part): ExecutionLesson | null {
    if (part.type !== "tool") return null

    const toolName = part.tool
    const state = part.state

    // Handle completed execution
    if (state.status === "completed") {
      const output = state.output ?? ""
      const input = state.input

      if (looksLikeError(output)) {
        return {
          content: formatErrorLesson(toolName, input, output),
          tags: ["error", toolName, "execution"],
          isPositive: false,
          tool: toolName,
          context: extractContext(input),
        }
      } else if (looksLikeSuccessPattern(output, toolName)) {
        return {
          content: formatSuccessLesson(toolName, input, output),
          tags: ["success", toolName, "execution"],
          isPositive: true,
          tool: toolName,
          context: extractContext(input),
        }
      }
    }
    // Handle error status
    else if (state.status === "error") {
      const error = state.error ?? "Unknown error"
      return {
        content: formatErrorLesson(toolName, state.input, error),
        tags: ["error", toolName, "failure"],
        isPositive: false,
        tool: toolName,
        context: extractContext(state.input),
      }
    }

    return null
  }

  function looksLikeError(output: string): boolean {
    return ERROR_PATTERNS.some((pattern) => pattern.test(output))
  }

  function looksLikeSuccessPattern(output: string, tool: string): boolean {
    if (tool === "bash") {
      return output.startsWith("0") || output.includes("completed successfully")
    }
    return output.length > 0 && !looksLikeError(output)
  }

  function formatErrorLesson(tool: string, input: Record<string, unknown>, error: string): string {
    const cmd = extractCommand(input)
    const shortError = truncate(error.split("\n")[0], 100)
    return `${tool} failed: ${shortError}. Input: ${cmd}`
  }

  function formatSuccessLesson(tool: string, input: Record<string, unknown>, output: string): string {
    const cmd = extractCommand(input)
    const shortOutput = truncate(output.split("\n").slice(0, 3).join("; "), 80)
    return `${tool} succeeded: ${shortOutput}. Command: ${cmd}`
  }

  function extractCommand(input: Record<string, unknown>): string {
    if (input.command && typeof input.command === "string") {
      return truncate(input.command, 50)
    }
    if (input.args && Array.isArray(input.args)) {
      return truncate(input.args.join(" "), 50)
    }
    return "[complex input]"
  }

  function extractContext(input: Record<string, unknown>): string {
    const contextParts: string[] = []
    if (input.path) contextParts.push(String(input.path))
    if (input.file) contextParts.push(String(input.file))
    if (input.cwd) contextParts.push(String(input.cwd))
    return contextParts.slice(0, 2).join(", ") || "unknown"
  }

  function truncate(str: string, maxLen: number): string {
    if (str.length <= maxLen) return str
    return str.slice(0, maxLen - 3) + "..."
  }
}
```

### Step 2: Implemente o Curator (curator.ts)

O Curator aplica deltas ao playbook com deduplicação:

```typescript
import type { Playbook, PlaybookEntry } from "./schema"
import type { PlaybookDelta, EntryFeedback } from "./types"
import { Playbook as PlaybookHelpers } from "./schema"
import { PlaybookStorage } from "./storage"

export namespace PlaybookCurator {
  export async function applyDelta(delta: PlaybookDelta): Promise<Playbook> {
    return PlaybookStorage.update(async (playbook) => {
      const updated = { ...playbook, entries: [...playbook.entries] }

      // Remove entries
      for (const removedId of delta.removed) {
        updated.entries = updated.entries.filter((e) => e.id !== removedId)
      }

      // Add new entries
      for (const newEntry of delta.added) {
        if (!updated.entries.find((e) => e.id === newEntry.id)) {
          updated.entries.push(newEntry)
        }
      }

      // Update existing entries
      for (const update of delta.updated) {
        updated.entries = updated.entries.map((e) => {
          if (e.id !== update.id) return e
          const updatedEntry = { ...e, updatedAt: Date.now() }
          if (update.helpfulDelta) {
            updatedEntry.helpfulCount += update.helpfulDelta
          }
          if (update.harmfulDelta) {
            updatedEntry.harmfulCount += update.harmfulDelta
          }
          if (update.tagsToAdd && update.tagsToAdd.length > 0) {
            updatedEntry.tags = Array.from(new Set([...updatedEntry.tags, ...update.tagsToAdd]))
          }
          return updatedEntry
        })
      }

      updated.lastUpdated = Date.now()
      return updated
    })
  }

  export function createDeltaFromFeedback(playbook: Playbook, feedback: EntryFeedback[]): PlaybookDelta {
    const updates: PlaybookDelta["updated"] = []
    const removed: string[] = []

    for (const fb of feedback) {
      const entry = playbook.entries.find((e) => e.id === fb.entryId)
      if (!entry) continue

      if (fb.helpful) {
        updates.push({ id: fb.entryId, helpfulDelta: 1 })
      } else {
        updates.push({ id: fb.entryId, harmfulDelta: 1 })
        // Remove if too many harmful marks
        if (entry.harmfulCount + 1 >= 3) {
          removed.push(entry.id)
        }
      }
    }

    return { added: [], updated: updates, removed }
  }

  export async function createDeltaFromLessons(
    lessons: Array<{ content: string; tags: string[]; isPositive: boolean }>,
  ): Promise<PlaybookDelta> {
    const added: PlaybookEntry[] = []

    for (const lesson of lessons) {
      // Only add positive lessons
      if (lesson.isPositive) {
        added.push(PlaybookHelpers.createEntry(lesson.content, lesson.tags, null))
      }
    }

    return { added, updated: [], removed: [] }
  }

  export async function addLessons(
    lessons: Array<{ content: string; tags: string[]; isPositive: boolean }>,
  ): Promise<Playbook> {
    const delta = await createDeltaFromLessons(lessons)
    return await applyDelta(delta)
  }

  export async function updateWithFeedback(feedback: EntryFeedback[]): Promise<Playbook> {
    const playbook = await PlaybookStorage.load()
    const delta = createDeltaFromFeedback(playbook, feedback)
    return await applyDelta(delta)
  }
}
```

### Step 3: Implemente o Generator (generator.ts)

O Generator busca entradas relevantes para o contexto atual:

```typescript
import { PlaybookStorage } from "./storage"
import type { PlaybookEntry } from "./schema"

export namespace PlaybookGenerator {
  export async function getRelevantEntries(query: string, limit = 5): Promise<PlaybookEntry[]> {
    const playbook = await PlaybookStorage.load()
    const entries = playbook.entries

    if (entries.length === 0) return []

    // Simple keyword matching fallback
    const queryLower = query.toLowerCase()
    const queryWords = queryLower.split(/\s+/).filter((w) => w.length > 2)

    const scored = entries.map((entry) => {
      let score = 0
      const entryContent = entry.content.toLowerCase()

      // Keyword matching
      for (const word of queryWords) {
        if (entryContent.includes(word)) {
          score += 10
        }
      }

      // Tag matching
      for (const tag of entry.tags) {
        if (queryLower.includes(tag.toLowerCase())) {
          score += 15
        }
      }

      // Utility score
      score += entry.helpfulCount * 2
      score -= entry.harmfulCount * 3

      return { entry, score }
    })

    scored.sort((a, b) => b.score - a.score)
    return scored.slice(0, limit).map((s) => s.entry)
  }

  export async function getPositiveEntries(minScore = 0): Promise<PlaybookEntry[]> {
    const playbook = await PlaybookStorage.load()
    return playbook.entries
      .filter((e) => e.helpfulCount - e.harmfulCount >= minScore)
      .sort((a, b) => b.helpfulCount - a.harmfulCount - (a.helpfulCount - a.harmfulCount))
  }

  export function formatForPrompt(entries: PlaybookEntry[]): string {
    if (entries.length === 0) return ""

    const formatted = entries.map((e) => {
      const status = e.helpfulCount > e.harmfulCount ? "[HELPFUL]" : "[MAYBE HARMFUL]"
      const tags = e.tags.length > 0 ? ` (${e.tags.join(", ")})` : ""
      return `${status}${tags} [${e.id}]\n- ${e.content}`
    })

    return `## Relevant Playbook Entries\n${formatted.join("\n\n")}`
  }

  export async function generateContextForTask(task: string): Promise<string> {
    const relevant = await getRelevantEntries(task, 5)
    return formatForPrompt(relevant)
  }

  export async function generateCompactionContext(): Promise<{ usePlaybook: boolean; context: string }> {
    const playbook = await PlaybookStorage.load()
    const positiveEntries = playbook.entries.filter((e) => e.helpfulCount > e.harmfulCount)

    if (positiveEntries.length > 0) {
      const formatted = positiveEntries
        .slice(0, 10)
        .map((e) => {
          return `• ${e.content}`
        })
        .join("\n")

      return {
        usePlaybook: true,
        context: `## Learned Strategies\n${formatted}`,
      }
    }

    return {
      usePlaybook: false,
      context: "",
    }
  }
}
```

### Step 4: Atualize o Index

Adicione os novos exports ao `index.ts`:

```typescript
export { PlaybookSchema, PlaybookEntrySchema, type PlaybookEntry, type Playbook, Playbook } from "./schema"
export type { PlaybookDelta, EntryFeedback } from "./types"
export { PlaybookStorage } from "./storage"
export { PlaybookReflector, type ExecutionLesson } from "./reflector"
export { PlaybookCurator } from "./curator"
export { PlaybookGenerator } from "./generator"
```

## Integração com Processamento de Mensagens

No seu arquivo de processamento de mensagens (`processor.ts`), adicione hooks para extrair lições após cada execução de ferramenta:

```typescript
// After tool execution (pseudo-code)
async function handleToolResult(part: MessageV2.ToolPart) {
  // Extract lesson from the execution
  const lesson = PlaybookReflector.extractLessonFromPart(part)

  if (lesson) {
    // Save lesson asynchronously (fire-and-forget)
    PlaybookCurator.addLessons([lesson]).catch((e) => {
      console.warn("playbook: failed to save lesson", e)
    })
  }
}
```

## Testes

```typescript
import { describe, test, expect } from "bun:test"
import { PlaybookReflector } from "./reflector"
import { PlaybookCurator } from "./curator"
import { PlaybookGenerator } from "./generator"

describe("Reflector", () => {
  test("extracts lesson from error output", () => {
    // Mock the tool part with error status
    const part = createMockToolPart("bash", { status: "error", error: "ENOENT: no such file or directory" })
    const lesson = PlaybookReflector.extractLessonFromPart(part)
    expect(lesson).not.toBeNull()
    expect(lesson?.isPositive).toBe(false)
  })

  test("extracts lesson from success output", () => {
    const part = createMockToolPart("bash", { status: "completed", output: "0" })
    const lesson = PlaybookReflector.extractLessonFromPart(part)
    expect(lesson).not.toBeNull()
    expect(lesson?.isPositive).toBe(true)
  })
})

describe("Curator", () => {
  test("creates delta from lessons", async () => {
    const lessons = [{ content: "Test lesson", tags: ["test"], isPositive: true }]
    const delta = await PlaybookCurator.createDeltaFromLessons(lessons)
    expect(delta.added.length).toBe(1)
    expect(delta.added[0].content).toBe("Test lesson")
  })
})

describe("Generator", () => {
  test("formats entries for prompt", () => {
    const entries = [
      {
        id: "pb_test",
        content: "Use Read instead of cat",
        tags: ["best-practice"],
        helpfulCount: 5,
        harmfulCount: 0,
        createdAt: Date.now(),
        updatedAt: Date.now(),
        projectId: null,
        lastRetrievedAt: 0,
        retrievalCount: 0,
      },
    ]
    const formatted = PlaybookGenerator.formatForPrompt(entries)
    expect(formatted).toContain("pb_test")
    expect(formatted).toContain("[HELPFUL]")
  })
})
```

## Referências

- Reflector completo: `packages/opencode/src/session/playbook/reflector.ts`
- Curator completo: `packages/opencode/src/session/playbook/curator.ts`
- Generator completo: `packages/opencode/src/session/playbook/generator.ts`
