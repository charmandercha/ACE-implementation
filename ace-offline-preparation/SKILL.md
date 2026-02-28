---
name: ace-offline-preparation
description: Implementa preparação offline de playbooks: seed de documentação, inicialização de projetos e otimização de system prompts. Use quando quiser preparar o contexto antes de sessões começarem.
---

# ACE Offline Preparation: Preparação Pré-Sessão

## Visão Geral

Esta skill adiciona a capacidade de preparar playbooks **antes** de sessões começarem, diferenciando entre:

1. **Offline**: Preparação prévia (system prompt, seed de docs)
2. **Online**: Adaptação durante execução (execution feedback)

O artigo ACE menciona que pode ser usado tanto para "system prompt optimization (offline)" quanto "test-time memory adaptation (online)".

## Casos de Uso

| Caso de Uso            | Modo    | Descrição                           |
| ---------------------- | ------- | ----------------------------------- |
| Novo projeto           | Offline | Criar playbook seed automaticamente |
| Melhorar system prompt | Offline | Preparar contexto inicial offline   |
| Processar histórico    | Offline | Batch processar execuções passadas  |
| Adaptar durante sessão | Online  | Feedback em tempo real              |
| Recuperar de erros     | Online  | Aprender com falhas recentes        |

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                 OFFLINE PREPARATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                  │
│  │   DOCUMENT      │    │    PROJECT      │                  │
│  │   SEEDING       │    │   INITIALIZER   │                  │
│  ├─────────────────┤    ├─────────────────┤                  │
│  │ • README.md     │    │ • config.json   │                  │
│  │ • src/          │    │ • package.json  │                  │
│  │ • docs/         │    │ • .env.example  │                  │
│  │ • .gitignore    │    │ • tsconfig.json │                  │
│  └────────┬────────┘    └────────┬────────┘                  │
│           │                       │                             │
│           └───────────┬───────────┘                             │
│                       ▼                                          │
│              ┌─────────────────┐                                 │
│              │  PLAYBOOK SEED  │                                 │
│              └────────┬────────┘                                 │
│                       │                                          │
│                       ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │            SYSTEM PROMPT OPTIMIZATION                    │   │
│  │  • Combine playbook + static instructions               │   │
│  │  • Test in isolation                                     │   │
│  │  • Version for reuse                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Implementação

### Step 1: Document Seeding

Extrai estratégias automaticamente de documentação do projeto.

```typescript
// document-seeder.ts
import { readdir, readFile } from "fs/promises"
import { join, extname } from "path"
import { generateObject } from "ai"
import { z } from "zod"
import { Provider } from "@/provider/provider"

const StrategySchema = z.object({
  strategies: z.array(
    z.object({
      content: z.string().describe("Actionable strategy extracted from document"),
      source: z.string().describe("File or section where found"),
      confidence: z.number().describe("How confident we are in this strategy"),
      tags: z.array(z.string()).describe("Categorization"),
    }),
  ),
})

const DOCUMENT_EXTENSIONS = [".md", ".txt", ".json", ".yaml", ".yml"]
const IMPORTANT_FILES = ["README.md", "CONTRIBUTING.md", "SETUP.md", "docs/", "package.json", ".gitignore"]

export namespace DocumentSeeder {
  /**
   * Extrai estratégias de arquivos de documentação.
   */
  async function extractFromFile(filePath: string): Promise<string[]> {
    const ext = extname(filePath).toLowerCase()
    if (!DOCUMENT_EXTENSIONS.includes(ext)) return []

    try {
      const content = await readFile(filePath, "utf-8")

      // Skip very short or binary-like files
      if (content.length < 100 || content.includes("\0")) return []

      return [content]
    } catch {
      return []
    }
  }

  /**
   * Encontra arquivos relevantes para extração
   */
  async function findRelevantFiles(projectPath: string): Promise<string[]> {
    const files: string[] = []

    async function walk(dir: string, depth = 0): Promise<void> {
      if (depth > 3) return // Max depth

      try {
        const entries = await readdir(dir, { withFileTypes: true })

        for (const entry of entries) {
          const fullPath = join(dir, entry.name)

          // Skip node_modules, .git, etc
          if (entry.name.startsWith(".") || entry.name === "node_modules") continue
          if (entry.isDirectory() && ["dist", "build", "coverage"].includes(entry.name)) continue

          if (entry.isFile()) {
            const ext = extname(entry.name).toLowerCase()
            if (DOCUMENT_EXTENSIONS.includes(ext)) {
              files.push(fullPath)
            }
          } else if (entry.isDirectory()) {
            await walk(fullPath, depth + 1)
          }
        }
      } catch {
        // Skip inaccessible directories
      }
    }

    await walk(projectPath)
    return files
  }

  /**
   * Extrai estratégias de documentação usando LLM
   */
  async function extractStrategies(
    contents: string[],
    model: Provider.Model,
  ): Promise<Array<{ content: string; source: string; confidence: number; tags: string[] }>> {
    if (contents.length === 0) return []

    const languageModel = await Provider.getLanguage(model)

    // Combine relevant contents
    const combined = contents.slice(0, 10).join("\n\n--- Document ---\n\n")
    const truncated = combined.slice(0, 8000) // Limit context

    try {
      const result = await generateObject({
        model: languageModel,
        schema: StrategySchema,
        prompt: `Extract actionable development strategies from these project documents.

Look for:
- Setup/installation instructions
- Common errors and how to avoid them
- Best practices mentioned
- Testing requirements
- Build/deployment commands
- Code style conventions

Documents:
${truncated}

Return strategies that would help a developer working on this project.`,
        temperature: 0.2,
      })

      return result.object.strategies.map((s) => ({
        content: s.content,
        source: s.source,
        confidence: s.confidence,
        tags: s.tags,
      }))
    } catch (e) {
      console.warn("[playbook] Document extraction failed:", e)
      return []
    }
  }

  /**
   * Main entry point: seed playbook from project
   */
  export async function seedFromProject(projectPath: string, model: Provider.Model): Promise<PlaybookEntry[]> {
    console.log("[playbook] Seeding from project:", projectPath)

    const files = await findRelevantFiles(projectPath)
    console.log("[playbook] Found", files.length, "relevant files")

    const contents: string[] = []
    for (const file of files.slice(0, 20)) {
      const content = await extractFromFile(file)
      contents.push(...content)
    }

    const strategies = await extractStrategies(contents, model)

    return strategies
      .filter((s) => s.confidence > 0.5)
      .map((s) => Playbook.createEntry(s.content, [...s.tags, "seed", "documentation"], null))
  }

  /**
   * Seed from single file (e.g., README.md)
   */
  export async function seedFromFile(filePath: string, model: Provider.Model): Promise<PlaybookEntry[]> {
    const content = await readFile(filePath, "utf-8")
    const strategies = await extractStrategies([content], model)

    return strategies.map((s) => Playbook.createEntry(s.content, [...s.tags, "seed"], null))
  }
}
```

### Step 2: Project Initializer

Inicializa playbook para novos projetos automaticamente.

```typescript
// project-initializer.ts
import { readFile } from "fs/promises"
import { join } from "path"
import { DocumentSeeder } from "./document-seeder"
import { PlaybookStorage } from "./storage"
import { Playbook } from "./schema"

const PROJECT_CONFIG_FILES = [
  "package.json",
  "Cargo.toml",
  "pyproject.toml",
  "go.mod",
  "pom.xml",
  "Makefile",
  "docker-compose.yml",
]

export namespace ProjectInitializer {
  export interface InitOptions {
    projectPath: string
    projectName: string
    useDocumentation: boolean
    useDefaults: boolean
  }

  /**
   * Detecta o tipo de projeto
   */
  async function detectProjectType(projectPath: string): Promise<string[]> {
    const types: string[] = []

    for (const file of PROJECT_CONFIG_FILES) {
      try {
        await Bun.file(join(projectPath, file)).text()
        types.push(file.replace(/\.(json|toml|yml|yaml)/, ""))
      } catch {
        // File doesn't exist
      }
    }

    return types
  }

  /**
   * Gera estratégias padrão baseadas no tipo de projeto
   */
  function getDefaultStrategies(projectType: string[]): PlaybookEntry[] {
    const strategies: string[] = []

    if (projectType.includes("package.json")) {
      strategies.push(
        "Run 'npm install' before 'npm run dev' to ensure dependencies are installed",
        "Use 'npm run lint' or 'npm run typecheck' before committing",
        "Check node_modules exists before running npm scripts that depend on packages",
      )
    }

    if (projectType.includes("Cargo.toml")) {
      strategies.push(
        "Run 'cargo build' to compile the project before testing",
        "Use 'cargo clippy' for linting recommendations",
        "Check Cargo.lock is committed for reproducible builds",
      )
    }

    if (projectType.includes("Makefile")) {
      strategies.push(
        "Check Makefile exists and read targets before running make",
        "Use 'make help' or 'make -n' to preview commands",
      )
    }

    return strategies.map((s) => Playbook.createEntry(s, ["seed", "default", ...projectType], null))
  }

  /**
   * Inicializa playbook para novo projeto
   */
  export async function initialize(options: InitOptions, model: Provider.Model): Promise<Playbook> {
    const entries: PlaybookEntry[] = []

    // Detect project type
    const types = await detectProjectType(options.projectPath)

    // Add default strategies
    if (options.useDefaults) {
      const defaults = getDefaultStrategies(types)
      entries.push(...defaults)
    }

    // Extract from documentation
    if (options.useDocumentation) {
      const docStrategies = await DocumentSeeder.seedFromProject(options.projectPath, model)
      entries.push(...docStrategies)
    }

    // Save initial playbook
    const playbook = Playbook.create()
    playbook.entries = entries

    await PlaybookStorage.save(playbook)

    return playbook
  }
}
```

### Step 3: System Prompt Optimization

Prepara contexto otimizado para system prompts.

```typescript
// system-prompt-optimizer.ts
import { PlaybookStorage } from "./storage"
import { PlaybookGenerator } from "./generator"

const SYSTEM_PROMPT_TEMPLATE = `# Project Context

{{PLAYBOOK}}

---

## Static Instructions

You are an expert coding assistant.

## Guidelines

- Always verify files exist before editing
- Use specialized tools (Read, Edit, Write) instead of bash for file operations
- When you make changes, verify they work with appropriate tests or commands
`

export namespace SystemPromptOptimizer {
  /**
   * Gera system prompt otimizado combinando playbook + instruções estáticas
   */
  export async function generateOptimizedPrompt(
    staticInstructions: string,
    options: {
      includePositiveOnly?: boolean
      maxEntries?: number
      customTemplate?: string
    } = {},
  ): Promise<string> {
    const { includePositiveOnly = true, maxEntries = 10 } = options

    const playbook = await PlaybookStorage.load()

    // Filter entries
    let entries = playbook.entries

    if (includePositiveOnly) {
      entries = entries.filter((e) => e.helpfulCount > e.harmfulCount)
    }

    // Sort by utility
    entries = entries
      .sort((a, b) => {
        const scoreA = a.helpfulCount * 2 - a.harmfulCount * 3
        const scoreB = b.helpfulCount * 2 - b.harmfulCount * 3
        return scoreB - scoreA
      })
      .slice(0, maxEntries)

    // Format playbook
    const playbookSection = entries
      .map((e) => {
        const score = e.helpfulCount - e.harmfulCount
        const tags = e.tags.length > 0 ? ` [${e.tags.join(", ")}]` : ""
        return `•${tags} ${e.content} (score: ${score})`
      })
      .join("\n")

    const playbookBlock = playbookSection
      ? `## Learned Strategies\n\n${playbookSection}`
      : "## Learned Strategies\n\n(No strategies learned yet)"

    // Use template
    const template = options.customTemplate || SYSTEM_PROMPT_TEMPLATE

    return template
      .replace("{{PLAYBOOK}}", playbookBlock)
      .replace("## Static Instructions\n\n{{STATIC}}", staticInstructions)
  }

  /**
   * Versiona system prompts para comparação
   */
  export interface PromptVersion {
    id: string
    timestamp: number
    playbookSnapshot: Playbook
    prompt: string
    metrics?: {
      successRate?: number
      avgResponseTime?: number
    }
  }

  const versions: PromptVersion[] = []

  export async function createVersion(
    playbook: Playbook,
    prompt: string,
    metrics?: PromptVersion["metrics"],
  ): Promise<string> {
    const id = `v_${Date.now()}`

    versions.push({
      id,
      timestamp: Date.now(),
      playbookSnapshot: JSON.parse(JSON.stringify(playbook)),
      prompt,
      metrics,
    })

    return id
  }

  /**
   * Compara versões
   */
  export function compareVersions(
    idA: string,
    idB: string,
  ): {
    playbookDelta: number
    promptDelta: number
  } | null {
    const vA = versions.find((v) => v.id === idA)
    const vB = versions.find((v) => v.id === idB)

    if (!vA || !vB) return null

    return {
      playbookDelta: vB.playbookSnapshot.entries.length - vA.playbookSnapshot.entries.length,
      promptDelta: vB.prompt.length - vA.prompt.length,
    }
  }
}
```

### Step 4: Batch Processing

Processa histórico de execuções offline.

```typescript
// batch-processor.ts
import { PlaybookReflector } from "./reflector"
import { PlaybookCurator } from "./curator"
import { MultiEpochReflector } from "./multi-epoch-reflection"

export namespace BatchProcessor {
  export interface ExecutionRecord {
    timestamp: number
    tool: string
    input: Record<string, unknown>
    output: string
    status: "completed" | "error"
  }

  /**
   * Processa histórico de execuções em batch
   */
  export async function processHistory(
    records: ExecutionRecord[],
    options: {
      useMultiEpoch?: boolean
      parallel?: boolean
      model?: Provider.Model
    } = {},
  ): Promise<{ lessonsAdded: number; duplicatesSkipped: number }> {
    const { useMultiEpoch = false, parallel = true } = options

    const lessons: ExecutionLesson[] = []

    for (const record of records) {
      // Convert to tool part format
      const part: MessageV2.ToolPart = {
        type: "tool",
        tool: record.tool,
        state: {
          status: record.status,
          input: record.input,
          output: record.status === "completed" ? record.output : undefined,
          error: record.status === "error" ? record.output : undefined,
        },
      }

      if (useMultiEpoch && options.model) {
        const refined = await MultiEpochReflector.extractWithRefinement(part, options.model)
        if (refined) {
          lessons.push(refined.refined)
        }
      } else {
        const lesson = PlaybookReflector.extractLessonFromPart(part)
        if (lesson) lessons.push(lesson)
      }
    }

    // Add to playbook
    const playbook = await PlaybookCurator.addLessons(lessons)

    return {
      lessonsAdded: lessons.length,
      duplicatesSkipped: 0, // Would need to track this
    }
  }
}
```

## Configuração

```typescript
// config.ts
export interface OfflineConfig {
  documentSeeding: {
    enabled: boolean
    fileExtensions: string[]
    maxDepth: number
    maxFiles: number
  }
  projectInitializer: {
    enabled: boolean
    useDefaults: boolean
    useDocumentation: boolean
  }
  systemPromptOptimization: {
    enabled: boolean
    maxEntries: number
    includePositiveOnly: boolean
  }
  batchProcessing: {
    enabled: boolean
    useMultiEpoch: boolean
  }
}
```

## Fluxo de Uso

```
┌─────────────────────────────────────────────────────────────────┐
│                    USAGE FLOW                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. NEW PROJECT                                                  │
│     ┌─────────────────┐                                         │
│     │ ProjectInit.    │ → Creates initial playbook             │
│     │ initialize()    │   with defaults + docs                 │
│     └─────────────────┘                                         │
│                                                                  │
│  2. BEFORE SESSION                                              │
│     ┌─────────────────┐                                         │
│     │ SystemPrompt    │ → Generates optimized                  │
│     │ generate()      │   system prompt                         │
│     └─────────────────┘                                         │
│                                                                  │
│  3. AFTER PROJECT CHANGES                                       │
│     ┌─────────────────┐                                         │
│     │ DocumentSeeder  │ → Updates playbook                     │
│     │ seedFromProject │   with new strategies                   │
│     └─────────────────┘                                         │
│                                                                  │
│  4. OFFLINE BATCH                                               │
│     ┌─────────────────┐                                         │
│     │ BatchProcessor  │ → Process historical                    │
│     │ processHistory()│   executions                            │
│     └─────────────────┘                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Referências

- Artigo ACE, Abstract: "ACE optimizes contexts both offline (e.g., system prompts) and online (e.g., agent memory)"
- Código relacionado: `packages/opencode/src/session/playbook/`
