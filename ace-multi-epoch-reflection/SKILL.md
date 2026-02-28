---
name: ace-multi-epoch-reflection
description: Implementa reflexão iterativa (multi-epoch) que refina lições extraídas através de múltiplas passagens. Conforme o artigo ACE, esta técnica adiciona +3.7% de performance (de 55.7% para 59.4% no benchmark).
---

# ACE Multi-Epoch Reflection: Refinamento Iterativo de Lições

## Visão Geral

Esta skill implementa o mecanismo de **reflexão iterativa** que é crucial para a qualidade do sistema ACE. O artigo mostra que:

- **Sem multi-epoch**: 55.7% average no AppWorld
- **Com multi-epoch**: 59.4% average no AppWorld
- **Ganho**: +3.7%

A reflexão iterativa transforma lições brutas em lições acionáveis e generalizadas.

## Pré-requisitos

- Reflector básico implementado (ver skill `ace-agentic-roles`)
- Acesso ao modelo LLM para refinamento
- Curator para aplicar deltas refinadas

## O Problema que Resolve

### Extração Single-Pass (Original)

```
Tool: bash
Input: npm install
Output: ENOENT: package.json not found

→ "bash failed: ENOENT: package.json not found. Input: npm install"
```

Esta lição é:

- Específica demais (não generaliza)
- Não acionável (não diz o que fazer)
- Difícil de recuperar em contextos similares

### Com Multi-Epoch Reflection

```
Epoch 1 (Extract):
→ "bash failed: ENOENT: package.json not found. Input: npm install"

Epoch 2 (Refine):
→ "When 'npm install' fails with ENOENT: file not found,
   the package.json is missing. Check if file exists first."

Epoch 3 (Generalize):
→ "Before running npm commands, verify package.json exists.
   Use 'ls package.json' or 'test -f package.json && npm install'"
```

Esta lição é:

- Generalizada (aplica a qualquer npm command)
- Acionável (diz o que fazer)
- Facilmente recuperável

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                 MULTI-EPOCH REFLECTION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                                               │
│  │ Tool Result  │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │   EPOCH 1   │────▶│   EPOCH 2   │────▶│   EPOCH 3   │    │
│  │  Extract    │     │   Refine    │     │  Generalize  │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│  "bash failed          "When npm            "Before running    │
│   ENOENT..."             fails with           npm commands,    │
│                          ENOENT..."            verify file..."   │
│                                                                  │
│  Quality Score: 0.3     Quality Score: 0.6    Quality Score: 0.9│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Implementação

### Step 1: Defina Schema de Qualidade

```typescript
// quality.ts
export interface LessonQuality {
  score: number // 0-1
  isActionable: boolean // Pode ser seguido diretamente?
  isGeneralized: boolean // Aplica a outros contextos?
  isSpecific: boolean // Exemplo específico sem generalização?
}

export interface RefinedLesson {
  original: ExecutionLesson
  refined: ExecutionLesson
  quality: LessonQuality
  epochs: number
  cost: number // Tokens gastos
}

export namespace LessonQuality {
  const QUALITY_THRESHOLD = 0.7

  export function assess(lesson: ExecutionLesson): LessonQuality {
    let score = 0.3 // Base score

    // Check for actionable language
    const actionablePatterns = [
      /always remember to/i,
      /make sure to/i,
      /before you/i,
      /first check/i,
      /verify that/i,
      /use \w+ instead/i,
      /prefer \w+ over/i,
    ]

    const hasActionable = actionablePatterns.some((p) => p.test(lesson.content))
    if (hasActionable) score += 0.2

    // Check for generalization
    const genericPatterns = [
      /\b(always|never|whenever)\b/i,
      /\b(any|every)\b.*\b(file|command|path)\b/i,
      /\b(first|then)\b/i,
    ]

    const hasGeneric = genericPatterns.some((p) => p.test(lesson.content))
    if (hasGeneric) score += 0.2

    // Check for specificity (negative)
    const specificPatterns = [/\bproject-[a-z0-9]+\b/i, /\bfile-[a-z0-9]+\.ext\b/i, /path\/to\//i]

    const isSpecific = specificPatterns.some((p) => p.test(lesson.content))
    if (isSpecific) score -= 0.1

    // Penalize trivial lessons
    const trivialPatterns = [/^ls\s+/i, /^cat\s+/i, /^cd\s+\//i, /lists?\s+(the\s+)?files?$/i]

    const isTrivial = trivialPatterns.some((p) => p.test(lesson.content))
    if (isTrivial) score -= 0.2

    return {
      score: Math.max(0, Math.min(1, score)),
      isActionable: score >= QUALITY_THRESHOLD,
      isGeneralized: hasGeneric,
      isSpecific,
    }
  }

  export function shouldRefine(quality: LessonQuality): boolean {
    return quality.score < QUALITY_THRESHOLD && !quality.isActionable
  }
}
```

### Step 2: Implemente Multi-Epoch Reflector

```typescript
// multi-epoch-reflector.ts
import { generateObject } from "ai"
import { z } from "zod"
import { Provider } from "@/provider/provider"
import { type ExecutionLesson } from "./reflector"
import { LessonQuality, type RefinedLesson, type LessonQuality as LessonQualityType } from "./quality"

const RefinementSchema = z.object({
  refinedLesson: z.object({
    content: z.string().describe("Generalized, actionable lesson"),
    tags: z.array(z.string()).describe("Updated tags"),
    improvement: z.string().describe("What was improved from previous version"),
  }),
  quality: z.object({
    score: z.number().describe("Quality score 0-1"),
    isActionable: z.boolean().describe("Can be followed directly?"),
    isGeneralized: z.boolean().describe("Applies to multiple contexts?"),
  }),
})

const MAX_EPOCHS = 3
const QUALITY_THRESHOLD = 0.7

export namespace MultiEpochReflector {
  /**
   * Extrai e refina uma lição através de múltiplas passagens.
   *
   * Epoch 1: Extração inicial (do Reflector original)
   * Epoch 2: Refinamento - torna acionável
   * Epoch 3: Generalização - amplia escopo
   */
  export async function extractWithRefinement(
    part: MessageV2.ToolPart,
    model: Provider.Model,
  ): Promise<RefinedLesson | null> {
    // Epoch 1: Initial extraction
    const initialLesson = extractLessonFromPart(part)
    if (!initialLesson) return null

    let currentLesson = initialLesson
    let currentQuality = LessonQuality.assess(currentLesson)
    let epochs = 1

    // Early exit if already good
    if (currentQuality.score >= QUALITY_THRESHOLD && currentQuality.isActionable) {
      return {
        original: initialLesson,
        refined: currentLesson,
        quality: currentQuality,
        epochs: 1,
        cost: 0,
      }
    }

    // Epoch 2: Refinement (make actionable)
    if (LessonQuality.shouldRefine(currentQuality)) {
      const refined2 = await refineLesson(currentLesson, currentQuality, 2, model)
      if (refined2) {
        currentLesson = refined2.lesson
        currentQuality = refined2.quality
        epochs = 2
      }
    }

    // Epoch 3: Generalization (if still not good)
    if (LessonQuality.shouldRefine(currentQuality) && epochs < MAX_EPOCHS) {
      const refined3 = await refineLesson(currentLesson, currentQuality, 3, model)
      if (refined3) {
        currentLesson = refined3.lesson
        currentQuality = refined3.quality
        epochs = 3
      }
    }

    return {
      original: initialLesson,
      refined: currentLesson,
      quality: currentQuality,
      epochs,
      cost: epochs > 1 ? epochs * 500 : 0, // Estimate
    }
  }

  async function refineLesson(
    lesson: ExecutionLesson,
    quality: LessonQualityType,
    epoch: number,
    model: Provider.Model,
  ): Promise<{ lesson: ExecutionLesson; quality: LessonQualityType } | null> {
    const prompts = {
      2: `Refine this lesson to make it ACTIONABLE.
Make it a clear instruction that can be followed directly.

Current: ${lesson.content}
Tags: ${lesson.tags.join(", ")}

Provide a refined version that:
- Starts with "Always...", "Make sure to...", "Before you..."
- Specifies the exact action to take
- Keeps it concise but clear`,
      3: `Generalize this lesson to apply to MORE contexts.
Keep the actionable part but make it apply broadly.

Current: ${lesson.content}

Generalize by:
- Replacing specific file/command names with patterns
- Using "any" or "whenever" appropriately
- Keeping the core wisdom intact`,
    }

    try {
      const languageModel = await Provider.getLanguage(model)

      const result = await generateObject({
        model: languageModel,
        schema: RefinementSchema,
        prompt: prompts[epoch as 2 | 3],
        temperature: 0.2,
      })

      if (result.object.quality.score < 0.3) {
        return null // Skip if worse
      }

      return {
        lesson: {
          content: result.object.refinedLesson.content,
          tags: result.object.refinedLesson.tags,
          isPositive: lesson.isPositive,
          tool: lesson.tool,
          context: lesson.context,
        },
        quality: {
          score: result.object.quality.score,
          isActionable: result.object.quality.isActionable,
          isGeneralized: result.object.quality.isGeneralized,
          isSpecific: false,
        },
      }
    } catch (e) {
      console.warn("[playbook] Refinement epoch failed:", e)
      return null
    }
  }

  /**
   * Batch process for efficiency
   */
  export async function extractBatchWithRefinement(
    parts: MessageV2.ToolPart[],
    model: Provider.Model,
    options: { maxEpochs?: number; parallel?: boolean } = {},
  ): Promise<RefinedLesson[]> {
    const results: RefinedLesson[] = []

    if (options.parallel) {
      const promises = parts.map((p) => extractWithRefinement(p, model))
      const resolved = await Promise.all(promises)
      results.push(...resolved.filter((r): r is RefinedLesson => r !== null))
    } else {
      for (const part of parts) {
        const result = await extractWithRefinement(part, model)
        if (result) results.push(result)
      }
    }

    return results
  }
}
```

### Step 3: Integração com Curator

```typescript
// curator.ts (update)
import { MultiEpochReflector, type RefinedLesson } from "./multi-epoch-reflector"

export namespace PlaybookCurator {
  /**
   * Versão que usa multi-epoch reflection para lições de alta importância
   */
  export async function addLessonsWithRefinement(
    lessons: Array<{ content: string; tags: string[]; isPositive: boolean }>,
    options: { useRefinement?: boolean; importance?: "low" | "medium" | "high" } = {},
  ): Promise<Playbook> {
    const { useRefinement = true, importance = "medium" } = options

    // Only refine high importance lessons
    const toRefine: ExecutionLesson[] = []
    const toAddDirectly: ExecutionLesson[] = []

    for (const lesson of lessons) {
      if (useRefinement && importance === "high" && !lesson.isPositive) {
        toRefine.push(lesson)
      } else {
        toAddDirectly.push(lesson)
      }
    }

    // Add direct lessons first
    let playbook = await addLessons(toAddDirectly)

    // Then add refined lessons
    // (In production, would call MultiEpochReflector here)

    return playbook
  }
}
```

### Step 4: Configuração

```typescript
// config.ts
export interface PlaybookConfig {
  // ... existing config
  multiEpochReflection: {
    enabled: boolean
    maxEpochs: number
    qualityThreshold: number
    onlyPositive: boolean // Only refine positive lessons
    importanceThreshold: "low" | "medium" | "high"
  }
}

export const DEFAULT_MULTI_EPOCH_CONFIG = {
  enabled: true,
  maxEpochs: 3,
  qualityThreshold: 0.7,
  onlyPositive: false,
  importanceThreshold: "medium",
}
```

### Step 5: Logging e Métricas

```typescript
// metrics.ts
export namespace ReflectionMetrics {
  const metrics: Array<{
    timestamp: number
    originalLength: number
    refinedLength: number
    epochs: number
    qualityBefore: number
    qualityAfter: number
    cost: number
  }> = []

  export function record(refined: RefinedLesson): void {
    metrics.push({
      timestamp: Date.now(),
      originalLength: refined.original.content.length,
      refinedLength: refined.refined.content.length,
      epochs: refined.epochs,
      qualityBefore: LessonQuality.assess(refined.original).score,
      qualityAfter: refined.quality.score,
      cost: refined.cost,
    })
  }

  export function getStats() {
    if (metrics.length === 0) return null

    const avgQualityBefore = metrics.reduce((a, m) => a + m.qualityBefore, 0) / metrics.length
    const avgQualityAfter = metrics.reduce((a, m) => a + m.qualityAfter, 0) / metrics.length
    const avgEpochs = metrics.reduce((a, m) => a + m.epochs, 0) / metrics.length

    return {
      totalRefinements: metrics.length,
      avgQualityImprovement: avgQualityAfter - avgQualityBefore,
      avgEpochs,
      qualityBefore: avgQualityBefore,
      qualityAfter: avgQualityAfter,
    }
  }
}
```

## Quando Usar Multi-Epoch

| Cenário                 | Usar Multi-Epoch? | Epochs |
| ----------------------- | ----------------- | ------ |
| Erros críticos          | ✅ Sim            | 2-3    |
| Success patterns        | ✅ Sim            | 2      |
| Lições simples (ls, cd) | ❌ Não            | 1      |
| Performance crítica     | ❌ Não            | 1      |
| Primeira vez no projeto | ✅ Sim            | 3      |

## Trade-offs

| Aspecto      | Single-Pass | Multi-Epoch       |
| ------------ | ----------- | ----------------- |
| Latência     | ~50ms       | ~200-500ms        |
| Custo LLM    | $0          | $0.001-0.005/call |
| Qualidade    | 55.7%       | 59.4% (+3.7%)     |
| Complexidade | Simples     | Moderada          |

## Testes

```typescript
import { describe, test, expect } from "bun:test"
import { LessonQuality } from "./quality"

describe("Lesson Quality", () => {
  test("scores actionable lesson high", () => {
    const lesson = {
      content: "Always verify package.json exists before running npm install",
      tags: ["npm", "setup"],
      isPositive: true,
    }
    const quality = LessonQuality.assess(lesson as any)
    expect(quality.score).toBeGreaterThan(0.5)
    expect(quality.isActionable).toBe(true)
  })

  test("scores trivial lesson low", () => {
    const lesson = {
      content: "ls lists files",
      tags: ["bash"],
      isPositive: true,
    }
    const quality = LessonQuality.assess(lesson as any)
    expect(quality.score).toBeLessThan(0.3)
  })

  test("detects specific lesson", () => {
    const lesson = {
      content: "When running npm in /Users/marcos/project-abc123, check package.json",
      tags: ["npm"],
      isPositive: true,
    }
    const quality = LessonQuality.assess(lesson as any)
    expect(quality.isSpecific).toBe(true)
  })
})
```

## Referências

- Artigo ACE, Seção 4 (Ablation Study): "ACE w/o multi-epoch: 55.7% vs ACE (full): 59.4%"
- Código relacionado: `packages/opencode/src/session/playbook/reflector.ts`
