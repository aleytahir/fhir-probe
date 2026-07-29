```
╔═══════════════════════════════════════════════════════╗
║                                                       ║
║   ⬡  @flexpa/llm-fhir-eval                           ║
║                                                       ║
║   The missing benchmark for clinical AI.              ║
║   Test LLMs on real FHIR tasks before they            ║
║   touch real patient data.                            ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝
```

> **Because clinical AI deserves more than a vibe check.**

![FHIR R4](https://img.shields.io/badge/FHIR-R4-e11d48?style=flat-square)
![Version](https://img.shields.io/badge/version-0.0.4-0d9488?style=flat-square)
![promptfoo](https://img.shields.io/badge/powered_by-promptfoo-f97316?style=flat-square)
![Models](https://img.shields.io/badge/models_tested-14-4f46e5?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-64748b?style=flat-square)

`@flexpa/llm-fhir-eval` is an open, reproducible benchmark for evaluating large language models on real FHIR R4 healthcare tasks — generation, validation, extraction, and recursive tool use. Implements and extends prior art from [FHIR-GPT (NEJM AI, 2023)](https://ai.nejm.org/doi/10.1056/AIcs2300301).

> [!NOTE]
> Follow the development discussion on [FHIR Chat](https://chat.fhir.org/#narrow/channel/323443-Artificial-Intelligence.2FMachine-Learning-.28AI.2FML.29/topic/LLM.20FHIR.20Eval.20Preview/near/483998202).

---

## Why this exists

Every LLM leaderboard tests coding, reasoning, math. None of them tell you whether a model can:

- Generate a valid `MedicationRequest` bundle from an unstructured clinical note
- Extract an `Observation` value with the correct LOINC code
- Call `$validate`, parse the `OperationOutcome`, and fix its own output

This framework closes that gap. Vendor-neutral. Reproducible. Built for healthcare engineers.

---

## What gets benchmarked

| Task | Description | Eval path |
|---|---|---|
| **Resource Generation** | Unstructured note → valid FHIR R4 Bundle | `evals/generation/` |
| **Resource Validation** | Live `$validate` + `OperationOutcome` parsing | `assertions/validateOperation.mjs` |
| **Data Extraction** | FHIR JSON → precise clinical data points | `evals/extraction/` |
| **Recursive Tool Use** | Multi-turn `$validate` self-correction loop | `providers/*RecursiveToolCalls*.ts` |

---

## Models tested

**Anthropic** — Claude 3.5 Haiku · Claude 3.5 Sonnet · Claude Sonnet 4 · Claude Opus 4

**OpenAI** — GPT-3.5-turbo · GPT-4.1 · O3 (low reasoning) · O3 (high reasoning)

All models support recursive tool calling up to depth 10 via the custom providers.

---

## How the pipeline works

```
Clinical Note / FHIR Resource
        │
        ▼
  promptfoo eval engine
        │
        ├──► Provider (Anthropic / OpenAI)
        │           │
        │           ▼
        │      LLM Draft Bundle
        │           │
        │    [multi-turn only]
        │           ▼
        │    validateFhirBundle.mjs   ◄─────────┐
        │           │                           │
        │    OperationOutcome errors            │
        │           │                           │
        │    LLM Revision ────────────── (depth 1..10)
        │
        ▼
  Assertion Layer
        ├── isBundle              structural gate
        ├── metaElementMissing    spec compliance
        ├── fhirPathEquals        clinical precision
        └── validateOperation     live server check
        │
        ▼
  Pass@1 score per model · per category
```

---

## Generation modes

### Zero-shot

```
Clinical note ──► Single-pass LLM ──► FHIR Bundle ──► Assertions
```

Baseline capability. No feedback loop. Tests raw FHIR comprehension.

### Multi-turn with `$validate` (recursive)

```
Clinical note ──► LLM Draft
                      │
                      ▼
               validateFhirBundle  ──► OperationOutcome
                      │                      │
                      └──────────────────────┘
                         LLM sees errors, revises
                         repeats up to 10 times
                              │
                              ▼
                        Final Bundle
```

Models that use `$validate` to self-correct produce measurably better FHIR output. The recursive providers make that delta visible and comparable across vendors.

---

## Custom assertions

Four layers of correctness, from syntax to semantics:

```
1. isBundle.mjs
   └── resourceType === 'Bundle'
       Fast structural gate. Fails immediately on prose, partial JSON,
       or wrong resource types.

2. metaElementMissing.mjs
   └── !resource.meta
       Spec compliance. FHIR R4 prompts prohibit meta elements.
       Catches models that hallucinate schema fields.

3. fhirPathEquals.mjs
   └── evalFhirPath(fhirPath, result).length > 0
       Clinical precision. Uses @medplum/core — the same FHIRPath
       engine used in production FHIR servers. Tests LOINC bindings,
       name fields, dosage values, and more.

4. validateOperation.mjs
   └── $validate → OperationOutcome → structured errors
       Live server validation. Separates syntactically correct from
       semantically valid FHIR. This is the hardest gate to pass.
```

---

## Extraction suite

Two prompt strategies benchmarked side-by-side:

| Strategy | Approach | Use case |
|---|---|---|
| **Minimalist** | Bare instruction, no domain context | Baseline capability check |
| **Specialist** | Full clinical informaticist persona | Production-grade extraction |

Six test categories:

```
evals/extraction/tests/
├── basic-demographics.yaml       name, DOB, gender, MRN, address
├── conditions.yaml               ICD codes, onset dates, status
├── observations.yaml             lab values, units, LOINC codes
├── explanations-of-benefit.yaml  payer, claim amounts, CPT service lines
├── medication-requests.yaml      drug names, dosage, RxNorm codes
└── patient-history.yaml          longitudinal summary across resources
```

---

## Project structure

```
llm-fhir-eval/
│
├── assertions/
│   ├── fhirPathEquals.mjs                  FHIRPath evaluator (@medplum/core)
│   ├── isBundle.mjs                        structural gate
│   ├── metaElementMissing.mjs              spec compliance gate
│   └── validateOperation.mjs              live $validate assertion
│
├── evals/
│   ├── extraction/
│   │   ├── config-minimalist.yaml          bare-prompt strategy
│   │   ├── config-specialist.yaml          persona-driven strategy
│   │   └── tests/                          6 clinical categories
│   └── generation/
│       ├── config-zero-shot-bundle.yaml    single-pass generation
│       ├── config-multi-turn-tool-use.js   recursive tool calling
│       └── tests.yaml                      14 clinical scenarios
│
├── providers/
│   ├── AnthropicMessagesWithRecursiveToolCallsProvider.ts
│   └── OpenAiResponsesWithRecursiveToolCallsProvider.ts
│
├── tools/
│   └── validateFhirBundle.mjs              $validate tool exposed to LLMs
│
├── etc/
│   └── fhir-gpt.yaml                       NEJM AI paper replication config
│
└── .env.template
```

---

## Quickstart

**Prerequisites:** Node.js · Yarn · API keys for the models you want to test

```bash
# 1. Clone and install
git clone https://github.com/flexpa/llm-fhir-eval.git
cd llm-fhir-eval
yarn install

# 2. Configure keys
cp .env.template .env
# Add ANTHROPIC_API_KEY, OPENAI_API_KEY, etc.

# 3. Run an evaluation
# Extraction — minimalist
promptfoo eval -c evals/extraction/config-minimalist.yaml

# Extraction — specialist
promptfoo eval -c evals/extraction/config-specialist.yaml

# Generation — zero-shot
promptfoo eval -c evals/generation/config-zero-shot-bundle.yaml

# Generation — multi-turn recursive tool use
promptfoo eval -c evals/generation/config-multi-turn-tool-use.js
```

Results print as a Pass@1 score table, broken down by model and test category.

---

## Contributing

Open areas with the most surface area:

- **New extraction categories** — SMART-on-FHIR tokens, CQL expressions, DICOM metadata
- **Additional providers** — Gemini 2.5, Llama 3, Mistral config
- **HL7 v2.x tasks** — extend beyond FHIR R4 to v2 message parsing
- **RAG retrieval benchmarks** — evaluation over FHIR Knowledge Base corpora
- **Partial credit scoring** — for nearly-correct terminology bindings

Open an issue before a large PR to align on approach.
Join the discussion on [FHIR Chat](https://chat.fhir.org/#narrow/channel/323443-Artificial-Intelligence.2FMachine-Learning-.28AI.2FML.29/topic/LLM.20FHIR.20Eval.20Preview).

---

## Citation

If you use `@flexpa/llm-fhir-eval` in research, please cite:

```bibtex
@software{kelly2024fhirllmeval,
  author  = {Kelly, Joshua},
  title   = {FHIR LLM Eval},
  version = {0.0.1},
  date    = {2024-11-22},
  url     = {https://github.com/flexpa/fhir-llm-evals},
  orcid   = {0009-0000-7191-0595}
}
```

This framework implements evaluations from:
**FHIR-GPT** — Li, Y. et al. *NEJM AI* (2023). [doi:10.1056/AIcs2300301](https://ai.nejm.org/doi/10.1056/AIcs2300301)

---

*Built by [Flexpa](https://flexpa.com) — the patient-consented claims data platform connecting 300+ health plans.*
*FHIR® is a registered trademark of HL7 and is used with permission.*
