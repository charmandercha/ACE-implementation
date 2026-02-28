---
name: ace-foundation
description: Implementa a estrutura de dados persistente e o sistema de armazenamento para um Playbook evolutivo. Use quando precisar adicionar aprendizado de contexto em um agente de IA baseado em LLM, substituindo resumos monolíticos por entradas estruturadas e itemizadas.
---

# ACE Foundation: Schema & Storage

## Visão Geral

Esta skill implementa a base do sistema ACE (Agentic Context Engineering), criando a estrutura de dados que permite ao agente aprender e persistir estratégias entre sessões. O foco é substituir o "context collapse" (onde resumos monolíticos perdem informações valiosas) por um sistema de entradas estruturadas e itemizadas.

## Pré-requisitos

- Projeto TypeScript existente (Bun)
- Acesso ao sistema de arquivos para persistência
- Estrutura de sessão existente no projeto

## Arquitetura

### Estrutura de Arquivos

Crie o diretório `src/session/playbook/` com os seguintes arquivos:

```
src/session/playbook/
├── schema.ts      # Definição de tipos e validação
├── storage.ts    # Persistência em JSON
├── types.ts      # Tipos derivados
└── base.json     # Estratégias iniciais (seed)
```

### Step 1: Defina o Schema (schema.ts)

O schema usa Zod para validação robusta:

```typescript
import z from "zod"
import { randomBytes } from "crypto"

export const PlaybookEntrySchema = z.object({
  id: z.string(),
  content: z.string(),
  helpfulCount: z.number().int().min(0).default(0),
  harmfulCount: z.number().int().min(0).default(0),
  createdAt: z.number(),
  updatedAt: z.number(),
  tags: z.array(z.string()).default([]),
  projectId: z.string().nullable().optional().default(null),
  lastRetrievedAt: z.number().optional().default(0),
  retrievalCount: z.number().int().min(0).default(0),
})

export const PlaybookSchema = z.object({
  version: z.string().default("1.0"),
  entries: z.array(PlaybookEntrySchema).default([]),
  lastUpdated: z.number(),
})

export type PlaybookEntry = z.infer<typeof PlaybookEntrySchema>
export type Playbook = z.infer<typeof PlaybookSchema>

export namespace Playbook {
  export function createId(): string {
    const bytes = randomBytes(16)
    return "pb_" + bytes.toString("hex")
  }

  export function createEntry(content: string, tags: string[] = [], projectId: string | null = null): PlaybookEntry {
    const now = Date.now()
    return {
      id: createId(),
      content,
      helpfulCount: 0,
      harmfulCount: 0,
      createdAt: now,
      updatedAt: now,
      tags,
      projectId,
      lastRetrievedAt: 0,
      retrievalCount: 0,
    }
  }

  export function create(): Playbook {
    return {
      version: "1.0",
      entries: [],
      lastUpdated: Date.now(),
    }
  }

  export function markHelpful(entry: PlaybookEntry): PlaybookEntry {
    return {
      ...entry,
      helpfulCount: entry.helpfulCount + 1,
      updatedAt: Date.now(),
    }
  }

  export function markHarmful(entry: PlaybookEntry): PlaybookEntry {
    return {
      ...entry,
      harmfulCount: entry.harmfulCount + 1,
      updatedAt: Date.now(),
    }
  }
}
```

### Step 2: Defina Tipos Derivados (types.ts)

```typescript
import type { PlaybookEntry } from "./schema"

export interface PlaybookDelta {
  added: PlaybookEntry[]
  updated: Array<{
    id: string
    helpfulDelta?: number
    harmfulDelta?: number
    tagsToAdd?: string[]
  }>
  removed: string[]
}

export interface EntryFeedback {
  entryId: string
  helpful: boolean
}
```

### Step 3: Crie o Storage (storage.ts)

O storage deve suportar:

- Persistência em JSON
- Warmup (carregar estratégias seed na primeira execução)
- Suporte a escopo por projeto (opcional)
- Escrita atômica para evitar corrupção

```typescript
import { readFile, writeFile, unlink, access, mkdir, rename } from "fs/promises"
import { join, dirname } from "path"
import { PlaybookSchema, type Playbook, type PlaybookEntry } from "./schema"

const PLAYBOOK_FILENAME = "playbook.json"

const BASE_PLAYBOOK: Playbook = {
  version: "1.0",
  entries: [],
  lastUpdated: Date.now(),
}

export namespace PlaybookStorage {
  // Returns the config directory path for your project
  function globalFilepath(): string {
    // Adjust this based on your project's config path
    return join(process.env.HOME || "", ".config", "your-project", PLAYBOOK_FILENAME)
  }

  export async function load(): Promise<Playbook> {
    try {
      const content = await readFile(globalFilepath(), "utf-8")
      const parsed = JSON.parse(content)
      return PlaybookSchema.parse(parsed)
    } catch {
      // Try to load base playbook
      return initializeWithBase()
    }
  }

  async function initializeWithBase(): Promise<Playbook> {
    try {
      const basePath = join(import.meta.dir, "base.json")
      const content = await readFile(basePath, "utf-8")
      const parsed = JSON.parse(content)
      const playbook = PlaybookSchema.parse(parsed)

      // Reset timestamps for base entries
      const now = Date.now()
      for (const entry of playbook.entries) {
        entry.createdAt = now
        entry.updatedAt = now
      }
      playbook.lastUpdated = now

      // Save the initialized playbook
      await save(playbook)
      return playbook
    } catch {
      return { ...BASE_PLAYBOOK, lastUpdated: Date.now() }
    }
  }

  export async function save(playbook: Playbook): Promise<void> {
    const file = globalFilepath()
    const dir = dirname(file)

    await mkdir(dir, { recursive: true }).catch(() => {})

    const tmpFile = `${file}.tmp`
    const content = JSON.stringify(playbook, null, 2)

    // Atomic write
    await writeFile(tmpFile, content, "utf-8")
    await rename(tmpFile, file)
  }

  export async function exists(): Promise<boolean> {
    try {
      await access(globalFilepath())
      return true
    } catch {
      return false
    }
  }

  export async function clear(): Promise<void> {
    try {
      await unlink(globalFilepath())
    } catch {
      // Ignore if file doesn't exist
    }
  }
}
```

### Step 4: Crie o Base Playbook (base.json)

Este arquivo contém estratégias seed que serão carregadas na primeira execução:

```json
{
  "version": "1.0",
  "entries": [
    {
      "id": "pb_seed_001",
      "content": "Use specialized tools (Read, Edit, Write) instead of bash for file operations. Avoid using 'cat', 'head', 'tail', 'sed', or 'awk' for file manipulation.",
      "tags": ["best-practice", "file-operations", "tools"],
      "helpfulCount": 0,
      "harmfulCount": 0,
      "createdAt": 0,
      "updatedAt": 0,
      "projectId": null,
      "lastRetrievedAt": 0,
      "retrievalCount": 0
    },
    {
      "id": "pb_seed_002",
      "content": "Always verify files exist before editing. Use Read tool first to confirm the file path and content.",
      "tags": ["best-practice", "file-operations", "safety"],
      "helpfulCount": 0,
      "harmfulCount": 0,
      "createdAt": 0,
      "updatedAt": 0,
      "projectId": null,
      "lastRetrievedAt": 0,
      "retrievalCount": 0
    },
    {
      "id": "pb_seed_003",
      "content": "When using bash, use '&&' to chain commands that depend on each other.",
      "tags": ["bash", "command-line", "best-practice"],
      "helpfulCount": 0,
      "harmfulCount": 0,
      "createdAt": 0,
      "updatedAt": 0,
      "projectId": null,
      "lastRetrievedAt": 0,
      "retrievalCount": 0
    }
  ],
  "lastUpdated": 0
}
```

### Step 5: Crie o Index (index.ts)

```typescript
export { PlaybookSchema, PlaybookEntrySchema, type PlaybookEntry, type Playbook, Playbook } from "./schema"
export type { PlaybookDelta, EntryFeedback } from "./types"
export { PlaybookStorage } from "./storage"
```

## Testes

Crie testes básicos para validar o schema e storage:

```typescript
import { describe, test, expect } from "bun:test"
import { Playbook, type PlaybookEntry } from "./schema"
import { PlaybookStorage } from "./storage"

describe("Playbook Schema", () => {
  test("creates entry with correct structure", () => {
    const entry = Playbook.createEntry("Test strategy", ["test", "example"])
    expect(entry.id).toStartWith("pb_")
    expect(entry.content).toBe("Test strategy")
    expect(entry.helpfulCount).toBe(0)
    expect(entry.tags).toEqual(["test", "example"])
  })

  test("marks entry as helpful", () => {
    const entry = Playbook.createEntry("Test", [])
    const marked = Playbook.markHelpful(entry)
    expect(marked.helpfulCount).toBe(1)
  })

  test("creates playbook", () => {
    const playbook = Playbook.create()
    expect(playbook.version).toBe("1.0")
    expect(playbook.entries).toEqual([])
  })
})
```

## Configuração

Adicione ao seu arquivo de configuração (`opencode.json` ou similar):

```json
{
  "playbook": {
    "enabled": true,
    "mode": "webdev",
    "deduplicationThreshold": 0.85,
    "maxEntries": 500
  }
}
```

## Integração com o Projeto

1. Importe o módulo no arquivo de sessão principal
2. Carregue o playbook durante a inicialização da sessão
3. Use `PlaybookStorage.load()` para obter as entradas

## Referências

- Schema completo: `packages/opencode/src/session/playbook/schema.ts`
- Storage completo: `packages/opencode/src/session/playbook/storage.ts`
- Base playbook: `packages/opencode/src/session/playbook/base.json`
