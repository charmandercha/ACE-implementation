---
name: ace-semantic-intelligence
description: Implementa busca semântica usando embeddings (model2vec) para deduplicação e recuperação de entries do playbook conforme Seção 3.2 do artigo ACE. Use quando precisar de correspondência semântica avançada além de keyword matching simples.
---

# ACE Semantic Intelligence: Embeddings (model2vec)

## Visão Geral

Esta skill adiciona capacidade de busca semântica ao sistema ACE, permitindo:

- **Deduplicação semântica** (Seção 3.2 do artigo): "comparing bullets via semantic embeddings"
- **Recuperação inteligente** por similaridade de significado
- **Fallback em cascata**: Rust nativo → Python → Levenshtein

O artigo ACE especifica que o sistema de embeddings é usado para o passo de deduplicação no mecanismo "Grow-and-Refine", permitindo que estratégias similares sejam mescladas em vez de duplicadas.

## O que o model2vec faz no Sistema ACE

### Relação com o Artigo (Seção 3.2)

> "A de-duplication step prunes redundancy by comparing bullets via semantic embeddings"

O model2vec é responsável por:

1. **Deduplicação semântica**: Detectar que "Usar Read ao invés de cat" e "Preferir Read em vez de bash" são similares
2. **Recuperação inteligente**: Encontrar entries relevantes mesmo sem palavras-chave exatas
3. **Similaridade vetorial**: Calcular quão similares são dois textos matematicamente

### Cadeia de Fallback

```
┌─────────────────────────────────────────────────────────────┐
│              EMBEDDINGS BACKEND CHAIN                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  embedTexts(texts[])                                        │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────┐  ← Mais rápido (~1.7x que Python)   │
│  │ 1. Rust/Native   │    Zero overhead de processo         │
│  │    (model2vec)  │    ~128MB modelo                     │
│  └────────┬─────────┘                                       │
│           │ Fallback                                        │
│           ▼                                                 │
│  ┌──────────────────┐  ← Script Python subprocess          │
│  │ 2. Python        │    Funciona sem Rust                 │
│  │    (subprocess) │    pip install model2vec             │
│  └────────┬─────────┘                                       │
│           │ Fallback                                        │
│           ▼                                                 │
│  ┌──────────────────┐  ← Similaridade textual simples     │
│  3. Levenshtein    │    Sempre funciona                   │
│  (JS native)       │    fallback final                    │
│  └──────────────────┘                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Pré-requisitos

- Sistema de embeddings disponível (model2vec, OpenAI, ou outro)
- Curator e Generator implementados (ver skill `ace-agentic-roles`)
- (Opcional) Rust toolchain para compilação nativa

## Estrutura de Arquivos

```
src/session/playbook/
├── embeddings/
│   ├── index.ts           # API TypeScript (fallback chain)
│   ├── model2vec.py      # Script Python (fallback 1)
│   └── model2vec-rs/     # Módulo Rust nativo (preferido)
│       ├── Cargo.toml
│       ├── src/lib.rs
│       └── package.json
```

## Implementação Completa

### Step 1: Módulo TypeScript com Fallback Chain (index.ts)

```typescript
import { spawn } from "bun"
import { join, dirname } from "path"
import { existsSync } from "fs"

function getNativeModulePath(): string {
  const url = new URL(import.meta.url)
  const platform = process.platform
  const arch = process.arch

  let platformName = platform
  if (platform === "win32") platformName = "win32"
  else if (platform === "darwin") platformName = "darwin"
  else if (platform === "linux") platformName = "linux"

  return join(url.pathname, "..", "model2vec-rs", `${platformName}-${arch}.node`)
}

let nativeModule: any = null
let nativeModuleLoadAttempted = false

async function loadNativeModule(): Promise<boolean> {
  if (nativeModuleLoadAttempted) return nativeModule !== null
  nativeModuleLoadAttempted = true

  const nativePath = getNativeModulePath()
  if (!existsSync(nativePath)) {
    console.warn("[playbook] Native module not found at:", nativePath)
    return false
  }

  try {
    nativeModule = await import(nativePath)
    console.log("[playbook] Loaded native model2vec module (Rust)")
    return true
  } catch (e) {
    console.warn("[playbook] Failed to load native module:", e)
    return false
  }
}

export interface EmbeddingResult {
  embeddings: number[][]
  error?: string
}

/**
 * Gera embeddings para uma lista de textos.
 * Tenta: Rust nativo → Python subprocess → retorna erro
 */
export async function embedTexts(texts: string[]): Promise<EmbeddingResult> {
  if (texts.length === 0) return { embeddings: [] }

  const hasNative = await loadNativeModule()

  if (hasNative && nativeModule) {
    try {
      const result = nativeModule.embedTexts(texts)
      return { embeddings: result, error: undefined }
    } catch (e) {
      console.warn("[playbook] Native module embedTexts failed:", e)
    }
  }

  // Fallback to Python
  return embedTextsWithPython(texts)
}

/**
 * Calcula similaridade de cosseno entre dois vetores.
 * Fórmula: cos(θ) = (A · B) / (||A|| × ||B||)
 */
export function cosineSimilarity(a: number[], b: number[]): number {
  if (nativeModule && nativeModule.cosineSimilarity) {
    try {
      return nativeModule.cosineSimilarity(a, b)
    } catch {
      // Fall through to JS implementation
    }
  }

  if (a.length !== b.length || a.length === 0) return 0

  let dot = 0
  let normA = 0
  let normB = 0

  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i]
    normA += a[i] * a[i]
    normB += b[i] * b[i]
  }

  const denominator = Math.sqrt(normA) * Math.sqrt(normB)
  if (denominator === 0) return 0

  return dot / denominator
}

export interface SimilarityResult {
  index: number
  similarity: number
}

/**
 * Encontra o texto mais similar a uma query.
 * Retorna null se nenhuma similaridade atingir o threshold.
 */
export async function findMostSimilar(
  query: string,
  candidates: string[],
  threshold = 0.75,
): Promise<SimilarityResult | null> {
  if (candidates.length === 0) return null

  const hasNative = await loadNativeModule()

  if (hasNative && nativeModule?.findMostSimilar) {
    try {
      const result = nativeModule.findMostSimilar(query, candidates, threshold)
      if (result) {
        return { index: result.index, similarity: result.similarity }
      }
      return null
    } catch (e) {
      console.warn("[playbook] Native findMostSimilar failed:", e)
    }
  }

  const { embeddings, error } = await embedTexts([query, ...candidates])

  if (error || embeddings.length < 2) {
    return null
  }

  const queryEmbedding = embeddings[0]
  let bestSimilarity = -1
  let bestIndex = -1

  for (let i = 0; i < candidates.length; i++) {
    const similarity = cosineSimilarity(queryEmbedding, embeddings[i + 1])
    if (similarity > bestSimilarity) {
      bestSimilarity = similarity
      bestIndex = i
    }
  }

  if (bestSimilarity >= threshold) {
    return { index: bestIndex, similarity: bestSimilarity }
  }

  return null
}

export function isNativeModuleLoaded(): boolean {
  return nativeModule !== null
}

export function getModelInfo(): string {
  if (nativeModule && nativeModule.getModelInfo) {
    try {
      return nativeModule.getModelInfo()
    } catch {
      // Fall through
    }
  }
  return "model2vec-rs (not loaded)"
}
```

### Step 2: Script Python Fallback (model2vec.py)

```python
#!/usr/bin/env python3
"""
Model2vec embedding script for playbook semantic search.
Este script é usado como fallback quando o módulo Rust nativo não está disponível.
"""

import sys
import json
from model2vec import StaticModel

def main():
    try:
        # Carrega modelo uma vez (lazy loading)
        model = StaticModel.get()

        # Lê input do stdin
        input_texts = json.load(sys.stdin)

        if not isinstance(input_texts, list):
            input_texts = [input_texts]

        # Gera embeddings
        embeddings = []
        for text in input_texts:
            vector = model.encode(text)
            embeddings.append(vector.tolist())

        # Output como JSON
        print(json.dumps(embeddings))

    except Exception as e:
        sys.stderr.write(f"Error: {e}\n")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### Step 3: Módulo Rust Nativo (model2vec-rs) - O Mais Performático

Esta é a implementação preferida. O Rust oferece ~1.7x mais performance que Python.

#### Cargo.toml

```toml
[package]
name = "model2vec-embeddings"
version = "0.1.0"
edition = "2021"
authors = ["OpenCode AI"]
description = "Fast Model2Vec embeddings using Rust via NAPI-RS"
license = "MIT"

[lib]
crate-type = ["cdylib"]

[dependencies]
napi = { version = "2", features = ["async", "napi9"] }
napi-derive = "2"
model2vec = "0.3"
anyhow = "1.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
once_cell = "1.19"

[build-dependencies]
napi-build = "2"

[profile.release]
lto = true
strip = true
opt-level = 3
codegen-units = 1
```

#### src/lib.rs - Código Completo

```rust
use napi::bindgen_prelude::*;
use napi_derive::napi;
use once_cell::sync::OnceCell;
use model2vec::Model2Vec;
use std::sync::Mutex;

/**
 * Modelo singleton - carregado uma vez na memória.
 * Thread-safe via Mutex.
 */
static MODEL: OnceCell<Mutex<Model2Vec>> = OnceCell::new();

const DEFAULT_MODEL: &str = "minishlab/potion-multilingual-128M";

/**
 * Obtém ou inicializa o modelo.
 * O modelo é carregado lazy (na primeira chamada).
 */
fn get_model() -> Result<&'static Mutex<Model2Vec>> {
    MODEL.get_or_try_init(|| {
        let model = Model2Vec::from_pretrained(DEFAULT_MODEL, None, None)
            .map_err(|e| Error::new(
                Status::GenericFailure,
                format!("Failed to load model '{}': {}", DEFAULT_MODEL, e)
            ))?;
        Ok(Mutex::new(model))
    })
}

/**
 * Gera embeddings para uma lista de textos.
 *
 * Este é o coração do sistema de busca semântica.
 * Converte texto → vetor numérico de 128 dimensões.
 *
 * @param texts - Lista de textos para embeddar
 * @return Vec<Vec<f64>> - Array de vetores de embedding
 */
#[napi]
pub fn embed_texts(texts: Vec<String>) -> Result<Vec<Vec<f64>>> {
    if texts.is_empty() {
        return Ok(vec![]);
    }

    let model = get_model()?;
    let model = model.lock().map_err(|_| Error::new(Status::GenericFailure, "Failed to lock model"))?;

    let embeddings = model.encode(&texts).map_err(|e| {
        Error::new(Status::GenericFailure, format!("Failed to encode: {}", e))
    })?;

    let nrows = embeddings.nrows();
    let ncols = embeddings.ncols();
    let mut result: Vec<Vec<f64>> = Vec::with_capacity(nrows);

    for i in 0..nrows {
        let mut row = Vec::with_capacity(ncols);
        for j in 0..ncols {
            row.push(embeddings[[i, j]] as f64);
        }
        result.push(row);
    }

    Ok(result)
}

/**
 * Calcula similaridade de cosseno entre dois vetores.
 *
 * Fórmula: cos(θ) = (A · B) / (||A|| × ||B||)
 *
 * @param a - Primeiro vetor
 * @param b - Segundo vetor
 * @return f64 - Similaridade entre -1 e 1
 */
#[napi]
pub fn cosine_similarity(a: Vec<f64>, b: Vec<f64>) -> f64 {
    if a.len() != b.len() || a.is_empty() {
        return 0.0;
    }

    let mut dot = 0.0_f64;
    let mut norm_a = 0.0_f64;
    let mut norm_b = 0.0_f64;

    for i in 0..a.len() {
        dot += (a[i] as f64) * (b[i] as f64);
        norm_a += (a[i] as f64) * (a[i] as f64);
        norm_b += (b[i] as f64) * (b[i] as f64);
    }

    let denominator = norm_a.sqrt() * norm_b.sqrt();
    if denominator == 0.0 {
        0.0
    } else {
        dot / denominator
    }
}

/**
 * Resultado de busca de similaridade.
 */
#[napi]
pub struct SimilarityResult {
    pub index: i32,
    pub similarity: f64,
}

/**
 * Encontra o texto mais similar a uma query.
 *
 * Usado para deduplicação (Seção 3.2 do artigo ACE).
 * Compara a query com todos os candidatos e retorna
 * o mais similar se ultrapassar o threshold.
 *
 * @param query - Texto de busca
 * @param candidates - Textos candidatos
 * @param threshold - Limiar de similaridade (padrão: 0.75)
 * @return Option<SimilarityResult> - Melhor match ou None
 */
#[napi]
pub fn find_most_similar(query: String, candidates: Vec<String>, threshold: f64) -> Result<Option<SimilarityResult>> {
    if candidates.is_empty() {
        return Ok(None);
    }

    let threshold = if threshold > 0.0 { threshold } else { 0.75 };

    let mut all_texts = vec![query];
    all_texts.extend(candidates);

    let embeddings = embed_texts(all_texts)?;

    if embeddings.len() < 2 {
        return Ok(None);
    }

    let query_embedding = &embeddings[0];

    let mut best_similarity = -1.0_f64;
    let mut best_index = -1_i32;

    for i in 0..(embeddings.len() - 1) {
        let candidate = &embeddings[i + 1];
        let similarity = cosine_similarity(candidate.clone(), query_embedding.clone());

        if similarity > best_similarity {
            best_similarity = similarity;
            best_index = i as i32;
        }
    }

    if best_similarity >= threshold {
        Ok(Some(SimilarityResult {
            index: best_index,
            similarity: best_similarity,
        }))
    } else {
        Ok(None)
    }
}

/**
 * Calcula matriz de similaridade entre todos os textos.
 * Útil para análise de clusters de Entries.
 */
#[napi]
pub fn compute_similarity_matrix(texts: Vec<String>) -> Result<Option<Vec<Vec<f64>>>> {
    if texts.is_empty() {
        return Ok(None);
    }

    let embeddings = embed_texts(texts)?;

    let n = embeddings.len();
    let mut matrix: Vec<Vec<f64>> = Vec::with_capacity(n);

    for i in 0..n {
        let mut row = Vec::with_capacity(n);
        for j in 0..n {
            let similarity = cosine_similarity(embeddings[i].clone(), embeddings[j].clone());
            row.push(similarity);
        }
        matrix.push(row);
    }

    Ok(Some(matrix))
}

/**
 * Verifica se o modelo está carregado.
 */
#[napi]
pub fn is_model_loaded() -> bool {
    MODEL.get().is_some()
}

/**
 * Retorna informações do modelo.
 */
#[napi]
pub fn get_model_info() -> String {
    format!("model2vec-rs v0.1.0 | Model: {}", DEFAULT_MODEL)
}
```

### Step 4: Setup Automático (setup.js)

Para instalar automaticamente:

```javascript
const { execSync } = require("child_process")
const fs = require("fs")
const path = require("path")

const platform = process.platform
const arch = process.arch

const platformMap = {
  "linux-x64": "linux-x64",
  "darwin-x64": "darwin-x64",
  "darwin-arm64": "darwin-arm64",
  "win32-x64": "win32-x64",
}

const binaryName = `${platformMap[`${platform}-${arch}`]}.node`
const binaryPath = path.join(__dirname, binaryName)

async function setup() {
  // Check if binary exists
  if (fs.existsSync(binaryPath)) {
    console.log("✅ Native module already exists")
    return
  }

  // Try to build with Rust
  try {
    console.log("🔨 Attempting to build with Rust...")
    execSync("npm run build", { stdio: "inherit" })
    console.log("✅ Built successfully")
    return
  } catch (e) {
    console.log("⚠️  Rust build failed, trying to download prebuilt...")
  }

  // Download prebuilt binary (simplified)
  // In production, would download from GitHub Releases
}

setup()
```

### Step 5: Integração no Curator (Deduplicação)

```typescript
import { findMostSimilar } from "./embeddings"

export namespace PlaybookCurator {
  async function deduplicate(newEntry: PlaybookEntry): Promise<boolean> {
    const existing = playbook.entries.map((e) => e.content)

    // Usa embeddings para encontrar similar
    const result = await findMostSimilar(newEntry.content, existing, 0.85)

    if (result) {
      // Encontrou entry similar - incrementa helpfulCount
      await applyDelta({
        added: [],
        updated: [
          {
            id: playbook.entries[result.index].id,
            helpfulDelta: 1,
            tagsToAdd: newEntry.tags.filter((t) => !playbook.entries[result.index].tags.includes(t)),
          },
        ],
        removed: [],
      })
      console.log(
        `[playbook] Deduplicated: ${newEntry.content.substring(0, 30)}... similar to ${existing[result.index].substring(0, 30)}... (similarity: ${result.similarity.toFixed(2)})`,
      )
      return true
    }

    return false
  }
}
```

### Step 6: Integração no Generator (Ranking)

```typescript
import { embedTexts, cosineSimilarity } from "./embeddings"

export namespace PlaybookGenerator {
  async function rankEntries(entries: PlaybookEntry[], query: string, limit: number): Promise<PlaybookEntry[]> {
    const texts = [query, ...entries.map((e) => e.content)]
    const { embeddings, error } = await embedTexts(texts)

    if (error || embeddings.length < 2) {
      // Fallback: keyword matching
      return rankByKeywords(entries, query, limit)
    }

    const queryEmbedding = embeddings[0]

    const scored = entries.map((entry, i) => {
      const embedding = embeddings[i + 1]

      // Similaridade semântica (0-100)
      const semanticScore = cosineSimilarity(queryEmbedding, embedding) * 100

      // Score de utilidade
      const utilityScore = entry.helpfulCount * 2 - entry.harmfulCount * 3

      // Score total
      const totalScore = semanticScore + utilityScore

      return { entry, score: totalScore }
    })

    scored.sort((a, b) => b.score - a.score)
    return scored.slice(0, limit).map((s) => s.entry)
  }
}
```

## Configuração

```typescript
export interface PlaybookConfig {
  embeddingThreshold: number // 0.75-0.9 recomendado
  embeddingModel?: "model2vec" | "openai" | "disabled"
}
```

## Testes

```typescript
import { describe, test, expect } from "bun:test"
import { embedTexts, cosineSimilarity, findMostSimilar } from "./embeddings"

describe("Embeddings", () => {
  test("computes cosine similarity", () => {
    const a = [1, 0, 0]
    const b = [1, 0, 0]
    const c = [0, 1, 0]

    expect(cosineSimilarity(a, b)).toBe(1)
    expect(cosineSimilarity(a, c)).toBe(0)
  })

  test("finds similar text", async () => {
    const query = "how to read files"
    const candidates = ["use Read tool for files", "bash is for commands", "edit files with Edit tool"]

    const result = await findMostSimilar(query, candidates, 0.5)
    expect(result).not.toBeNull()
    // Deve encontrar "use Read tool for files" como mais similar
  }, 10000)
})
```

## Referências

- Embeddings module: `packages/opencode/src/session/playbook/embeddings/index.ts`
- Model2Vec Python: `packages/opencode/src/session/playbook/embeddings/model2vec.py`
- Rust module: `packages/opencode/src/session/playbook/embeddings/model2vec-rs/`
