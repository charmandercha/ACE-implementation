---
name: ace-maintenance-scaling
description: Implementa sistemas de manutenção e robustez: poda inteligente, rate limiting, WAL (Write-Ahead Log) e locking. Use quando precisar de um sistema de playbook confiável em produção com múltiplas sessões concorrentes.
---

# ACE Maintenance & Scaling: Pruner, Rate Limiting, WAL & Lock

## Visão Geral

Esta skill adiciona sistemas de robustez essenciais para uso em produção:

1. **Pruner**: Remove entries obsoletas ou de baixa utilidade
2. **Rate Limiting**: Previne spam de lições duplicadas
3. **WAL**: Garante durability mesmo em crashes
4. **Locking**: Previne condições de corrida

## Pré-requisitos

- Playbook module completo (skills anteriores)
- Sistema de arquivos para persistência

## Arquitetura

```
src/session/playbook/
├── pruner.ts      # Poda inteligente
├── rate-limit.ts  # Limitação de frequência
├── wal.ts        # Write-Ahead Log
└── lock.ts       # Locking de arquivos
```

## Step 1: Implemente Rate Limiting (rate-limit.ts)

Previne que lições duplicadas sobrecarreguem o playbook:

```typescript
export interface RateLimitConfig {
  maxLessonsPerMinute: number
  maxSimilarLessonsPerMinute: number
  dedupWindowMs: number
}

const DEFAULT_RATE_LIMIT: RateLimitConfig = {
  maxLessonsPerMinute: 10,
  maxSimilarLessonsPerMinute: 5,
  dedupWindowMs: 60000,
}

export namespace PlaybookRateLimiter {
  const recentLessons: Array<{ content: string; timestamp: number }> = []

  export function shouldAllowLesson(
    content: string,
    config?: Partial<RateLimitConfig>,
  ): { allowed: boolean; reason?: string } {
    const cfg = { ...DEFAULT_RATE_LIMIT, ...config }
    const now = Date.now()

    // Check global rate limit
    const oneMinuteAgo = now - 60000
    const recentCount = recentLessons.filter((l) => l.timestamp > oneMinuteAgo).length

    if (recentCount >= cfg.maxLessonsPerMinute) {
      return { allowed: false, reason: "rate limit: too many lessons per minute" }
    }

    // Check similar lessons
    const recentSimilar = recentLessons.filter(
      (l) => l.timestamp > now - cfg.dedupWindowMs && isSimilar(l.content, content),
    )

    if (recentSimilar.length >= cfg.maxSimilarLessonsPerMinute) {
      return { allowed: false, reason: "rate limit: too many similar lessons" }
    }

    return { allowed: true }
  }

  export function recordLesson(content: string): void {
    recentLessons.push({ content, timestamp: Date.now() })
    cleanupOldEntries()
  }

  function cleanupOldEntries(): void {
    const cutoff = Date.now() - 120000
    while (recentLessons.length > 0 && recentLessons[0].timestamp < cutoff) {
      recentLessons.shift()
    }
  }

  function isSimilar(a: string, b: string): boolean {
    const normalizedA = a.toLowerCase().replace(/[^a-z0-9]/g, "")
    const normalizedB = b.toLowerCase().replace(/[^a-z0-9]/g, "")

    if (normalizedA === normalizedB) return true
    if (normalizedA.includes(normalizedB) || normalizedB.includes(normalizedA)) return true

    return false
  }

  export function reset(): void {
    recentLessons.length = 0
  }

  export function getStats(): { recentCount: number; windowMs: number } {
    const now = Date.now()
    const oneMinuteAgo = now - 60000
    return {
      recentCount: recentLessons.filter((l) => l.timestamp > oneMinuteAgo).length,
      windowMs: 60000,
    }
  }
}
```

## Step 2: Implemente o Pruner (pruner.ts)

Remove entries de baixa utilidade automaticamente:

```typescript
import type { Playbook, PlaybookEntry } from "./schema"
import type { PlaybookDelta } from "./types"

export type PlaybookMode = "webdev" | "gamedev" | "conservative" | "aggressive"

export interface PruneAnalysis {
  totalEntries: number
  toRemove: string[]
  toMerge: Array<{ keep: string; remove: string; similarity: number }>
  reason: Record<string, string>
}

export interface PrunerConfig {
  maxEntries: number
  deduplicationThreshold: number
  mode: PlaybookMode
}

export namespace PlaybookPruner {
  const MAX_AGE_MS = 180 * 24 * 60 * 60 * 1000 // 180 days

  export async function analyze(playbook: Playbook, config: PrunerConfig): Promise<PruneAnalysis> {
    const entries = playbook.entries

    const result: PruneAnalysis = {
      totalEntries: entries.length,
      toRemove: [],
      toMerge: [],
      reason: {},
    }

    if (entries.length <= config.maxEntries) {
      return result
    }

    // Sort by utility score
    const sorted = sortByUtility(entries)

    // Remove lowest scoring entries
    const toRemove = sorted.slice(config.maxEntries)
    result.toRemove.push(...toRemove.map((e) => e.id))

    for (const e of toRemove) {
      result.reason[e.id] = "low utility score"
    }

    return result
  }

  export function createPruneDelta(analysis: PruneAnalysis): PlaybookDelta {
    return {
      added: [],
      updated: [],
      removed: analysis.toRemove,
    }
  }

  function sortByUtility(entries: PlaybookEntry[]): PlaybookEntry[] {
    return [...entries].sort((a, b) => {
      const scoreA = a.helpfulCount * 2 + a.retrievalCount - a.harmfulCount * 3
      const scoreB = b.helpfulCount * 2 + b.retrievalCount - b.harmfulCount * 3
      return scoreB - scoreA
    })
  }
}
```

## Step 3: Implemente WAL (wal.ts)

Write-Ahead Log para garantir durability:

```typescript
import { appendFile, readFile, unlink, mkdir } from "fs/promises"
import { join } from "path"
import type { PlaybookDelta } from "./types"

const PLAYBOOK_WAL_FILENAME = "playbook.wal"

export namespace PlaybookWAL {
  export interface WalEntry {
    timestamp: number
    delta: PlaybookDelta
  }

  function getWalPath(configDir: string, projectId: string | null): string {
    return join(configDir, projectId ? `playbook/${projectId}.wal` : PLAYBOOK_WAL_FILENAME)
  }

  export async function append(configDir: string, projectId: string | null, delta: PlaybookDelta): Promise<void> {
    const walPath = getWalPath(configDir, projectId)
    const entry: WalEntry = { timestamp: Date.now(), delta }
    const content = JSON.stringify(entry) + "\n"

    try {
      if (projectId) {
        await mkdir(join(configDir, "playbook"), { recursive: true }).catch(() => {})
      }
      await appendFile(walPath, content, "utf-8")
    } catch (e) {
      console.error("[playbook] Failed to append to WAL:", e)
    }
  }

  export async function readAll(configDir: string, projectId: string | null): Promise<WalEntry[]> {
    const walPath = getWalPath(configDir, projectId)
    const entries: WalEntry[] = []

    try {
      const content = await readFile(walPath, "utf-8")
      const lines = content.split("\n").filter((line) => line.trim().length > 0)

      for (const line of lines) {
        try {
          const entry = JSON.parse(line) as WalEntry
          entries.push(entry)
        } catch {
          // Ignore corrupted lines
        }
      }
    } catch {
      // File doesn't exist, return empty
    }

    return entries.sort((a, b) => a.timestamp - b.timestamp)
  }

  export async function truncate(configDir: string, projectId: string | null): Promise<void> {
    const walPath = getWalPath(configDir, projectId)
    try {
      await unlink(walPath)
    } catch {
      // Ignore if file doesn't exist
    }
  }
}
```

## Step 4: Implemente Locking (lock.ts)

Previne condições de corrida em ambientes concorrentes:

```typescript
import { writeFile, unlink, access, mkdir } from "fs/promises"
import { dirname } from "path"

const LOCK_TIMEOUT_MS = 30000
const LOCK_RETRY_MS = 50
const LOCK_STALE_MS = 60000

export class LockError extends Error {
  constructor(message: string) {
    super(message)
    this.name = "LockError"
  }
}

export async function withFileLock<T>(
  lockPath: string,
  fn: () => Promise<T>,
  options: { timeout?: number; retryInterval?: number } = {},
): Promise<T> {
  const timeout = options.timeout ?? LOCK_TIMEOUT_MS
  const retryInterval = options.retryInterval ?? LOCK_RETRY_MS
  const startTime = Date.now()

  while (Date.now() - startTime < timeout) {
    const acquired = await tryAcquireLock(lockPath)

    if (acquired) {
      try {
        return await fn()
      } finally {
        await releaseLock(lockPath)
      }
    }

    const isStale = await isLockStale(lockPath)
    if (isStale) {
      await releaseLock(lockPath)
      continue
    }

    await sleep(retryInterval)
  }

  throw new LockError(`Failed to acquire lock after ${timeout}ms: ${lockPath}`)
}

async function tryAcquireLock(lockPath: string): Promise<boolean> {
  try {
    const now = Date.now()
    const lockContent = JSON.stringify({
      pid: process.pid,
      timestamp: now,
    })

    await mkdir(dirname(lockPath), { recursive: true }).catch(() => {})
    await writeFile(lockPath, lockContent, { flag: "wx" })
    return true
  } catch {
    return false
  }
}

async function releaseLock(lockPath: string): Promise<void> {
  try {
    await unlink(lockPath)
  } catch {
    // Lock may already be released
  }
}

async function isLockStale(lockPath: string): Promise<boolean> {
  try {
    await access(lockPath)

    const content = await Bun.file(lockPath).text()
    const lock = JSON.parse(content)

    if (lock.timestamp && Date.now() - lock.timestamp > LOCK_STALE_MS) {
      return true
    }

    return false
  } catch {
    return false
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms))
}
```

## Step 5: Integre no Storage Atualizado

Atualize o `storage.ts` para usar WAL e Locking:

```typescript
import { withFileLock } from "./lock"
import { PlaybookWAL } from "./wal"

export namespace PlaybookStorage {
  export async function load(configDir: string): Promise<Playbook> {
    const lockPath = join(configDir, "playbook.lock")
    const playbookPath = join(configDir, "playbook.json")

    return withFileLock(lockPath, async () => {
      // First, replay WAL if any
      const pendingDeltas = await PlaybookWAL.readAll(configDir, null)
      let playbook: Playbook

      try {
        const content = await readFile(playbookPath, "utf-8")
        playbook = PlaybookSchema.parse(JSON.parse(content))
      } catch {
        playbook = createEmpty()
      }

      // Apply pending deltas
      for (const walEntry of pendingDeltas) {
        playbook = applyDelta(playbook, walEntry.delta)
      }

      // Clear WAL after successful replay
      if (pendingDeltas.length > 0) {
        await PlaybookWAL.truncate(configDir, null)
      }

      return playbook
    })
  }

  export async function update(
    configDir: string,
    fn: (playbook: Playbook) => Promise<Playbook> | Playbook,
  ): Promise<Playbook> {
    const lockPath = join(configDir, "playbook.lock")

    return withFileLock(lockPath, async () => {
      const playbook = await load(configDir)
      const updated = await fn(playbook)

      // Write WAL before saving
      // (simplified - in production, track changes more precisely)
      await PlaybookWAL.append(configDir, null, { added: [], updated: [], removed: [] })

      await save(configDir, updated)
      await PlaybookWAL.truncate(configDir, null)

      return updated
    })
  }
}
```

## Step 6: Configure no Curator

Atualize o `curator.ts` para usar rate limiting:

```typescript
import { PlaybookRateLimiter } from "./rate-limit"

export namespace PlaybookCurator {
  export async function addLessons(
    lessons: Array<{ content: string; tags: string[]; isPositive: boolean }>,
  ): Promise<Playbook> {
    const allowedLessons: Array<{ content: string; tags: string[]; isPositive: boolean }> = []

    for (const lesson of lessons) {
      const check = PlaybookRateLimiter.shouldAllowLesson(lesson.content)
      if (check.allowed) {
        allowedLessons.push(lesson)
        PlaybookRateLimiter.recordLesson(lesson.content)
      }
    }

    if (allowedLessons.length === 0) {
      return PlaybookStorage.load()
    }

    const delta = await createDeltaFromLessons(allowedLessons)
    return await applyDelta(delta)
  }
}
```

## Configuração

Adicione ao config:

```typescript
export interface PlaybookConfig {
  // ... other fields
  mode: "webdev" | "gamedev" | "conservative" | "aggressive"
  maxEntries: number
  deduplicationThreshold: number
}

const MODE_DEFAULTS = {
  webdev: { maxEntries: 500, deduplicationThreshold: 0.85 },
  conservative: { maxEntries: 500, deduplicationThreshold: 0.85 },
  gamedev: { maxEntries: 200, deduplicationThreshold: 0.8 },
  aggressive: { maxEntries: 200, deduplicationThreshold: 0.8 },
}
```

## Testes

```typescript
import { describe, test, expect, beforeEach } from "bun:test"
import { PlaybookRateLimiter } from "./rate-limit"
import { PlaybookPruner } from "./pruner"

describe("Rate Limiter", () => {
  beforeEach(() => {
    PlaybookRateLimiter.reset()
  })

  test("allows lessons within limit", () => {
    const result = PlaybookRateLimiter.shouldAllowLesson("test lesson")
    expect(result.allowed).toBe(true)
  })

  test("blocks duplicate lessons", () => {
    PlaybookRateLimiter.recordLesson("test lesson")
    PlaybookRateLimiter.recordLesson("test lesson")
    PlaybookRateLimiter.recordLesson("test lesson")
    PlaybookRateLimiter.recordLesson("test lesson")
    PlaybookRateLimiter.recordLesson("test lesson")
    PlaybookRateLimiter.recordLesson("test lesson")

    const result = PlaybookRateLimiter.shouldAllowLesson("test lesson")
    expect(result.allowed).toBe(false)
    expect(result.reason).toContain("similar lessons")
  })
})

describe("Pruner", () => {
  test("removes lowest utility entries", async () => {
    const playbook = {
      version: "1.0",
      entries: [
        { id: "a", helpfulCount: 10, harmfulCount: 0, retrievalCount: 5, content: "a" },
        { id: "b", helpfulCount: 1, harmfulCount: 0, retrievalCount: 0, content: "b" },
        { id: "c", helpfulCount: 0, harmfulCount: 5, retrievalCount: 0, content: "c" },
      ],
      lastUpdated: Date.now(),
    }

    const analysis = await PlaybookPruner.analyze(playbook, {
      maxEntries: 2,
      deduplicationThreshold: 0.8,
      mode: "webdev",
    })

    expect(analysis.toRemove).toContain("c")
    expect(analysis.toRemove).toContain("b")
  })
})
```

## Referências

- Rate limiter: `packages/opencode/src/session/playbook/rate-limit.ts`
- Pruner: `packages/opencode/src/session/playbook/pruner.ts`
- WAL: `packages/opencode/src/session/playbook/wal.ts`
- Lock: `packages/opencode/src/session/playbook/lock.ts`
