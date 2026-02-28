---
name: ace-evaluation-framework
description: Implementa sistema de avaliação e monitoramento: métricas de qualidade, A/B testing e ajuste dinâmico de thresholds. Use quando precisar validar se o playbook está melhorando resultados.
---

# ACE Evaluation Framework: Métricas e Validação

## Visão Geral

Sem avaliação, não sabemos se o sistema ACE está funcionando. Esta skill implementa:

1. **Métricas de Qualidade**: Quão boas são as lições extraídas
2. **Tracking de Recuperação**: As lições recuperadas foram úteis?
3. **A/B Testing**: Comparar playbook vs baseline
4. **Threshold Tuning**: Ajuste dinâmico de parâmetros

## Por Que Avaliação é Crítica

O artigo ACE lista nas limitações:

> "Context adaptation depends critically on feedback quality. In the absence of reliable feedback signals... the constructed context can be polluted by spurious or misleading signals."

Sem avaliação, o playbook pode:

- Acumular lições prejudiciais
- Ter qualidade degradada com o tempo
- Não melhorar (ou piorar) resultados

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                 EVALUATION FRAMEWORK                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    METRICS COLLECTION                     │  │
│  │  • Lesson Quality Score     • Retrieval Success Rate      │  │
│  │  • Deduplication Rate      • Context Improvement Delta   │  │
│  │  • False Positive Rate     • Latency Impact              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      A/B TESTING                          │  │
│  │                                                            │  │
│  │  Request A: +playbook    Request B: -playbook (baseline)  │  │
│  │        │                         │                         │  │
│  │        └────────────┬────────────┘                         │  │
│  │                     ▼                                      │  │
│  │            Compare Success Rate                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   THRESHOLD TUNING                         │  │
│  │                                                            │  │
│  │  Low quality → Increase threshold                         │  │
│  │  Too strict  → Decrease threshold                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Implementação

### Step 1: Métricas de Qualidade de Lições

```typescript
// lesson-metrics.ts
export interface LessonMetrics {
  lessonId: string
  extractedAt: number
  contentLength: number
  hasActionable: boolean
  hasGeneralization: boolean
  isTrivial: boolean
  qualityScore: number // 0-1
}

export interface RetrievalMetrics {
  lessonId: string
  retrievedAt: number
  wasRelevant: boolean
  wasUsed: boolean
  userFeedback?: "helpful" | "harmful" | "neutral"
}

export interface SessionMetrics {
  sessionId: string
  startedAt: number
  playbookEnabled: boolean
  lessonsExtracted: number
  lessonsRetrieved: number
  retrievalSuccessRate: number
  endAt?: number
}

export namespace LessonMetrics {
  const metrics: Map<string, LessonMetrics> = new Map()
  const retrievalMetrics: Map<string, RetrievalMetrics[]> = new Map()

  export function recordExtraction(lesson: ExecutionLesson): string {
    const id = `lm_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`
    const quality = assessQuality(lesson)

    metrics.set(id, {
      lessonId: id,
      extractedAt: Date.now(),
      contentLength: lesson.content.length,
      hasActionable: quality.isActionable,
      hasGeneralization: quality.isGeneralized,
      isTrivial: quality.isSpecific,
      qualityScore: quality.score,
    })

    return id
  }

  export function recordRetrieval(lessonId: string, sessionId: string, wasRelevant: boolean, wasUsed: boolean): void {
    const existing = retrievalMetrics.get(lessonId) || []
    existing.push({
      lessonId,
      retrievedAt: Date.now(),
      wasRelevant,
      wasUsed,
    })
    retrievalMetrics.set(lessonId, existing)
  }

  export function recordFeedback(lessonId: string, feedback: "helpful" | "harmful" | "neutral"): void {
    const existing = retrievalMetrics.get(lessonId)
    if (existing && existing.length > 0) {
      const last = existing[existing.length - 1]
      last.userFeedback = feedback
    }
  }

  function assessQuality(lesson: ExecutionLesson): {
    score: number
    isActionable: boolean
    isGeneralized: boolean
    isSpecific: boolean
  } {
    // Similar to LessonQuality in multi-epoch skill
    let score = 0.3
    const actionablePatterns = [/always remember/i, /make sure/i, /before you/i, /verify/i]
    const hasActionable = actionablePatterns.some((p) => p.test(lesson.content))
    if (hasActionable) score += 0.3

    const genericPatterns = [/\b(always|never|whenever)\b/i, /\b(any|every)\b/]
    const hasGeneric = genericPatterns.some((p) => p.test(lesson.content))
    if (hasGeneric) score += 0.2

    return {
      score: Math.min(1, score),
      isActionable: score > 0.5,
      isGeneralized: hasGeneric,
      isSpecific: false,
    }
  }

  export function getStats(): {
    totalLessons: number
    avgQuality: number
    actionableRate: number
    trivialRate: number
  } {
    const all = Array.from(metrics.values())
    if (all.length === 0) {
      return { totalLessons: 0, avgQuality: 0, actionableRate: 0, trivialRate: 0 }
    }

    return {
      totalLessons: all.length,
      avgQuality: all.reduce((a, m) => a + m.qualityScore, 0) / all.length,
      actionableRate: all.filter((m) => m.hasActionable).length / all.length,
      trivialRate: all.filter((m) => m.isTrivial).length / all.length,
    }
  }
}
```

### Step 2: Tracking de Recuperação

```typescript
// retrieval-tracker.ts
export interface RetrievalEvent {
  sessionId: string
  lessonId: string
  lessonContent: string
  timestamp: number
  context: string // What triggered retrieval
  usedInResponse: boolean
  userReaction?: "helpful" | "harmful" | "neutral"
}

export namespace RetrievalTracker {
  const events: RetrievalEvent[] = []

  export function trackRetrieval(
    sessionId: string,
    lessonId: string,
    lessonContent: string,
    context: string,
    usedInResponse: boolean,
  ): void {
    events.push({
      sessionId,
      lessonId,
      lessonContent,
      timestamp: Date.now(),
      context,
      usedInResponse,
    })
  }

  export function trackFeedback(
    lessonId: string,
    sessionId: string,
    reaction: "helpful" | "harmful" | "neutral",
  ): void {
    const event = events.find((e) => e.sessionId === sessionId && e.lessonId === lessonId)
    if (event) {
      event.userReaction = reaction
    }
  }

  export function getRetrievalStats(lessonId?: string): {
    totalRetrievals: number
    usedRate: number
    helpfulRate: number
    harmfulRate: number
    neutralRate: number
  } {
    const filtered = lessonId ? events.filter((e) => e.lessonId === lessonId) : events

    if (filtered.length === 0) {
      return { totalRetrievals: 0, usedRate: 0, helpfulRate: 0, harmfulRate: 0, neutralRate: 0 }
    }

    const used = filtered.filter((e) => e.usedInResponse).length
    const helpful = filtered.filter((e) => e.userReaction === "helpful").length
    const harmful = filtered.filter((e) => e.userReaction === "harmful").length
    const neutral = filtered.filter((e) => e.userReaction === "neutral").length
    const withFeedback = filtered.filter((e) => e.userReaction).length

    return {
      totalRetrievals: filtered.length,
      usedRate: used / filtered.length,
      helpfulRate: withFeedback > 0 ? helpful / withFeedback : 0,
      harmfulRate: withFeedback > 0 ? harmful / withFeedback : 0,
      neutralRate: withFeedback > 0 ? neutral / withFeedback : 0,
    }
  }

  /**
   * Gets lessons that are most effective
   */
  export function getTopPerformingLessons(limit = 10): Array<{
    lessonId: string
    retrievalCount: number
    helpfulRate: number
  }> {
    const byLesson = new Map<string, RetrievalEvent[]>()

    for (const event of events) {
      const existing = byLesson.get(event.lessonId) || []
      existing.push(event)
      byLesson.set(event.lessonId, existing)
    }

    const scored = Array.from(byLesson.entries()).map(([lessonId, evts]) => {
      const withFeedback = evts.filter((e) => e.userReaction)
      const helpful = withFeedback.filter((e) => e.userReaction === "helpful").length

      return {
        lessonId,
        retrievalCount: evts.length,
        helpfulRate: withFeedback.length > 0 ? helpful / withFeedback.length : 0,
      }
    })

    scored.sort((a, b) => b.helpfulRate - a.helpfulRate)
    return scored.slice(0, limit)
  }
}
```

### Step 3: A/B Testing

```typescript
// ab-tester.ts
export interface ABExperiment {
  id: string
  name: string
  startedAt: number
  variantA: "playbook" | "no-playbook"
  variantB: "playbook" | "no-playbook"
  sessions: {
    variant: "A" | "B"
    sessionId: string
    success: boolean
    latencyMs: number
    lessonsLearned: number
  }[]
}

export namespace ABTester {
  const experiments: Map<string, ABExperiment> = new Map()
  let currentExperimentId: string | null = null

  /**
   * Starts a new A/B experiment
   */
  export function startExperiment(
    name: string,
    variantA: "playbook" | "no-playbook" = "playbook",
    variantB: "playbook" | "no-playbook" = "no-playbook",
  ): string {
    const id = `ab_${Date.now()}`

    experiments.set(id, {
      id,
      name,
      startedAt: Date.now(),
      variantA,
      variantB,
      sessions: [],
    })

    currentExperimentId = id
    return id
  }

  /**
   * Records a session result
   */
  export function recordSession(
    variant: "A" | "B",
    sessionId: string,
    success: boolean,
    latencyMs: number,
    lessonsLearned: number,
  ): void {
    if (!currentExperimentId) return

    const exp = experiments.get(currentExperimentId)
    if (!exp) return

    exp.sessions.push({
      variant,
      sessionId,
      success,
      latencyMs,
      lessonsLearned,
    })
  }

  /**
   * Gets experiment results
   */
  export function getResults(experimentId: string):
    | {
        variant: string
        sessions: number
        successRate: number
        avgLatency: number
        avgLessonsLearned: number
      }[]
    | null {
    const exp = experiments.get(experimentId)
    if (!exp || exp.sessions.length === 0) return null

    const results = ["A", "B"].map((v) => {
      const variantSessions = exp.sessions.filter((s) => s.variant === v)
      const successCount = variantSessions.filter((s) => s.success).length

      return {
        variant: v === "A" ? exp.variantA : exp.variantB,
        sessions: variantSessions.length,
        successRate: variantSessions.length > 0 ? successCount / variantSessions.length : 0,
        avgLatency:
          variantSessions.length > 0
            ? variantSessions.reduce((a, s) => a + s.latencyMs, 0) / variantSessions.length
            : 0,
        avgLessonsLearned:
          variantSessions.length > 0
            ? variantSessions.reduce((a, s) => a + s.lessonsLearned, 0) / variantSessions.length
            : 0,
      }
    })

    return results
  }

  /**
   * Determines which variant is better
   */
  export function getWinner(experimentId: string): "A" | "B" | "tie" | null {
    const results = getResults(experimentId)
    if (!results || results.length < 2) return null

    const aSuccess = results[0].successRate
    const bSuccess = results[1].successRate

    if (aSuccess > bSuccess) return "A"
    if (bSuccess > aSuccess) return "B"
    return "tie"
  }
}
```

### Step 4: Threshold Tuning

```typescript
// threshold-optimizer.ts
export interface ThresholdConfig {
  deduplication: number
  retrieval: number
  quality: number
}

const DEFAULT_THRESHOLDS: ThresholdConfig = {
  deduplication: 0.75,
  retrieval: 0.5,
  quality: 0.7,
}

export namespace ThresholdOptimizer {
  let currentThresholds = { ...DEFAULT_THRESHOLDS }

  /**
   * Adjusts thresholds based on metrics
   */
  export function adjustThresholds(metrics: {
    deduplicationRate: number // Too high = too strict
    retrievalSuccessRate: number // Too low = too strict
    lowQualityRate: number // Too high = threshold too low
  }): ThresholdConfig {
    // Deduplication threshold
    if (metrics.deduplicationRate > 0.8) {
      // Too aggressive, decrease threshold
      currentThresholds.deduplication = Math.max(0.5, currentThresholds.deduplication - 0.05)
    } else if (metrics.deduplicationRate < 0.3) {
      // Too lenient, increase threshold
      currentThresholds.deduplication = Math.min(0.95, currentThresholds.deduplication + 0.05)
    }

    // Quality threshold
    if (metrics.lowQualityRate > 0.3) {
      currentThresholds.quality = Math.min(0.9, currentThresholds.quality + 0.05)
    } else if (metrics.lowQualityRate < 0.1) {
      currentThresholds.quality = Math.max(0.5, currentThresholds.quality - 0.05)
    }

    return currentThresholds
  }

  /**
   * Gets current thresholds
   */
  export function getThresholds(): ThresholdConfig {
    return { ...currentThresholds }
  }

  /**
   * Resets to defaults
   */
  export function reset(): void {
    currentThresholds = { ...DEFAULT_THRESHOLDS }
  }
}
```

### Step 5: Dashboard de Métricas

```typescript
// dashboard.ts
export async function generateReport(): Promise<string> {
  const lessonStats = LessonMetrics.getStats()
  const retrievalStats = RetrievalTracker.getRetrievalStats()
  const thresholds = ThresholdOptimizer.getThresholds()

  const report = `# ACE Evaluation Report

Generated: ${new Date().toISOString()}

## Lesson Quality

- Total extracted: ${lessonStats.totalLessons}
- Average quality: ${(lessonStats.avgQuality * 100).toFixed(1)}%
- Actionable rate: ${(lessonStats.actionableRate * 100).toFixed(1)}%
- Trivial rate: ${(lessonStats.trivialRate * 100).toFixed(1)}%

## Retrieval Performance

- Total retrievals: ${retrievalStats.totalRetrievals}
- Used in response: ${(retrievalStats.usedRate * 100).toFixed(1)}%
- User marked helpful: ${(retrievalStats.helpfulRate * 100).toFixed(1)}%
- User marked harmful: ${(retrievalStats.harmfulRate * 100).toFixed(1)}%

## Thresholds

- Deduplication: ${thresholds.deduplication}
- Retrieval: ${thresholds.retrieval}
- Quality: ${thresholds.quality}

## Top Performing Lessons

${getTopLessonsTable()}
`

  return report
}

function getTopLessonsTable(): string {
  const top = RetrievalTracker.getTopPerformingLessons(5)

  if (top.length === 0) return "No data yet"

  return top
    .map(
      (t, i) => `${i + 1}. ${t.lessonId}: ${t.retrievalCount} retrievals, ${(t.helpfulRate * 100).toFixed(0)}% helpful`,
    )
    .join("\n")
}
```

## Integração com Comandos

Adicione ao `commands.ts`:

```typescript
case "eval":
  return await cmdEval()

async function cmdEval(): Promise<CommandResult> {
  const report = await generateReport()
  return { response: report, done: true }
}
```

## Métricas Importantes para Monitorar

| Métrica                | Alerta se...                     |
| ---------------------- | -------------------------------- |
| Lesson Quality Score   | < 0.5 por mais de 50% das lições |
| Retrieval Success Rate | < 30%                            |
| Harmful Rate           | > 20%                            |
| Deduplication Rate     | > 90% (muito agressivo)          |
| Latency Impact         | > 100ms adicional                |

## Configuração

```typescript
// config.ts
export interface EvaluationConfig {
  enabled: boolean
  trackRetrieval: boolean
  trackFeedback: boolean
  abTestingEnabled: boolean
  autoThresholdAdjustment: boolean
  alertThresholds: {
    lowQualityRate: number
    highHarmfulRate: number
    highLatencyMs: number
  }
}
```

## Referências

- Artigo ACE, Appendix B (Limitations): "Context adaptation depends critically on feedback quality"
- Código relacionado: Métricas em `packages/opencode/src/session/playbook/`
