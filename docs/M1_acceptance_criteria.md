# Milestone 1 — Acceptance Criteria & Scope Boundaries

Status: **Proposed** (pending three-agent review + user approval)
Applies to: Milestone 1 — *Scientific Decision RAG MVP* (see `RAG_DecisionsRX_specs.docx` §6)

This document makes Milestone 1 acceptance-testable. It defines what is in and out
of scope for M1, the corpus and query set used to evaluate it, and the pass/fail
thresholds an M1 build must meet before it is considered complete.

---

## 1. Purpose

The specification's M1 success criteria are qualitative ("users can recover rationale
faster," "trust the evidence trail"). This document replaces those with concrete,
checkable criteria so that feature work (dense embeddings, metadata, claim-level
output) can be built and verified against a fixed target instead of drifting.

---

## 2. Scope boundaries

### In scope (Milestone 1)
- Text ingestion of `.txt`, `.pdf`, `.docx`.
- Pluggable embeddings: dense provider (e.g. OpenAI / BGE) with a deterministic
  local **hash** fallback for offline/CI use.
- Source-type-aware metadata at ingestion (e.g. distinguishing notes vs. transcripts
  vs. files) sufficient to support metadata-filtered retrieval.
- Citation-aware answers with a claim-level **explicit vs. inferred** distinction and
  explicit uncertainty handling.
- Structured JSON output conforming to the `AskResponse` schema.

### Out of scope — deferred to Milestone 2 (multimodal)
- Ingestion/interpretation of images, gels/blots, slides, and flow-cytometry plots.
- OCR / figure segmentation / caption extraction.
- Ingestion of legacy `.doc` and `.xls` binary formats and image/`.pzf` files
  (this covers the bulk of `data/P1 Affinity Evolution/`; see §3).

### Out of scope — deferred to Milestone 3 (cross-functional)
- Decision Graph layer (Neo4j, multi-node relationships, traversal).
- Role-specific model **routing** and the automated comparative LLM harness
  (Science / General / Compare modes). M1 may expose a single structured-prompt
  path; the multi-mode router is M3.
- Timeline reconstruction and contradiction-detection features.

---

## 3. Benchmark corpus

**M1 is evaluated against `mock_documents/`** — seven ingestible synthetic `.txt`
files (plus a `README.md`, which the loader does **not** ingest and which is not a
benchmark target). The `.txt` files are deterministic, already ingestible, and span
the rationale buckets:

| File | Primary rationale bucket |
|---|---|
| `scientific_study.txt` | scientific |
| `business_analysis.txt` | business |
| `supply_chain_note.txt` | supply_chain |
| `timeline.txt` | timeline |
| `regulatory_summary.txt` | regulatory |
| `mixed_evidence.txt` | conflicting / mixed |
| `institutional_note.txt` | institutional (corpus-supporting; not a benchmark target) |
| `README.md` | not ingested; not a benchmark target |

**Why mock and not real data:** `data/P1 Affinity Evolution/` is the real corpus but
is overwhelmingly legacy `.doc`, `.xls`, `.jpg`, and `.pzf` files that the current
loader cannot ingest (`.txt/.pdf/.docx` only), and the spreadsheet/image portions are
M2 territory. A deterministic mock corpus also keeps the benchmark reproducible in CI.
**Real-data evaluation is a documented future item**, unblocked once ingestion gains
`.doc`/`.xls` support (M2-adjacent).

---

## 4. Benchmark query set

12 curated questions. Each has a target rationale bucket, the source document(s) that
should be retrieved, the expected evidence, the claim type the answer should assign,
and a pass condition. Questions **Q10–Q12** are deliberate hard cases (conflict,
unknown, and no-evidence/abstention).

**What these queries do and do not test.** Spanning the business, supply-chain,
regulatory, and timeline buckets exercises M1's *grounded text retrieval and answer
behaviour* over those document types. It does **not** constitute the cross-functional
*synthesis* layer (M3) or automated corpus-wide *contradiction detection* (M3). In
particular, Q10 only checks that conflicting retrieved snippets are both surfaced to
the user — it does not require the system to automatically detect or reconcile the
contradiction.

| # | Question | Bucket | Expected source | Expected evidence | Claim type | Pass condition |
|---|---|---|---|---|---|---|
| Q1 | What were Compound X's first-year adoption rates? | scientific | `scientific_study.txt` | "8% to 32% in year 1" | explicit | Cites the range from the study |
| Q2 | Were any safety signals detected for Compound X? | scientific | `scientific_study.txt` | "mild in 3.2%… no safety signal detected" | explicit | States no safety signal, cited |
| Q3 | Why did adherence to Compound X drop off? | scientific | `scientific_study.txt` | "month 3… supply interruptions and unclear dosing guidance" | explicit | Names both drivers, cited |
| Q4 | Which segments are the best early adopters for Product Y? | business | `business_analysis.txt` | "specialty clinics and integrated health systems" | explicit | Names both segments, cited |
| Q5 | What is the estimated market size (TAM) for Product Y? | business | `business_analysis.txt` | "$1.2B TAM" | explicit | Cites the TAM figure |
| Q6 | What is the lead time for primary component Z? | supply_chain | `supply_chain_note.txt` | "10-14 weeks" | explicit | Cites the lead-time range |
| Q7 | How can single-source supply risk for component Z be mitigated? | supply_chain | `supply_chain_note.txt` | "safety stock (8-12 weeks)… qualify secondary supplier… air freight" | explicit | Lists ≥2 mitigations, cited |
| Q8 | What device classification is expected, and what is uncertain? | regulatory | `regulatory_summary.txt` | "Device classification unclear; likely falls under Class II equivalent" | explicit + uncertain | Gives class **and** flags that it is unclear |
| Q9 | When is full commercial launch planned, and on what is it contingent? | timeline | `timeline.txt` | "Month 9-12… contingent on payer engagement" | explicit | Cites window + contingency |
| Q10 | Is early adoption strong or weak? | conflicting | `mixed_evidence.txt` | Source A: "strong early uptake in urban centers" **vs.** Source B: "low patient willingness due to cost concerns" | inferred / conflicting | Surfaces **both** sides; does not assert one as settled |
| Q11 | Do accelerated approval pathways apply to this product? | regulatory | `regulatory_summary.txt` | "Outstanding questions: whether accelerated pathways apply" | unknown | Labels as unresolved/unknown; does **not** fabricate an answer |
| Q12 | What clinical evidence supports phage-display vector family X? | (none) | — none in corpus — | — | abstention | Returns no supporting evidence / states it cannot be found; **must not hallucinate** |

### Synonym / semantic variants (for §5 synonym metric)
Phrasings chosen to **minimise lexical overlap** with the target source text, so that
retrieval must rely on semantic similarity rather than shared keywords. Each is paired
with the term it deliberately avoids:
- Q3-syn: "Why did patients stop taking the treatment over time?" → `scientific_study.txt` (avoids "adherence"/"adoption").
- Q5-syn: "How large is the commercial opportunity in dollars?" → `business_analysis.txt`, "$1.2B TAM" (avoids "market"/"TAM"/"revenue").
- Q6-syn: "How long before the main part arrives?" → `supply_chain_note.txt`, "10-14 weeks" (avoids "component"/"lead time"/"supplier").
- Q8-syn: "Which oversight category applies to this product?" → `regulatory_summary.txt`, "Class II" (avoids "device"/"classification"/"regulatory").
- Q9-syn: "What is the rollout schedule?" → `timeline.txt` (avoids "timeline"/"deployment"/"launch").

**Small-set caveat:** with 5 variants, the ≥85% threshold rounds to "all 5 must pass"
(4/5 = 80% fails). This is acceptable for the MVP but statistically coarse; the variant
set should grow as the corpus expands so the percentage becomes meaningful.

---

## 5. Metrics & thresholds (Balanced)

Explicit-claim fabrication is fixed at **0** and structured-output compliance at
**100%** — a single fabricated explicit claim or schema-invalid response is an
automatic M1 fail, regardless of other scores.

| Metric | Threshold | Definition |
|---|---|---|
| **Groundedness rate** | **≥ 90%** | Of all assertions the system makes across the benchmark, the share traceable to a cited retrieved chunk. |
| **Fabricated explicit claims** | **0** | No explicit (non-inferred, non-"unknown") claim may lack supporting retrieved evidence. Hard gate. |
| **Retrieval recall@5** | **≥ 85%** | Share of *answerable* questions (Q1–Q11) whose target source appears in the top-5 retrieved chunks. |
| **Synonym / semantic success** | **≥ 85%** | Share of synonym-variant queries that retrieve the correct source. Evaluated **after dense embeddings land** (roadmap item 3); under hash embeddings this is expected to underperform, which is the motivation for that item. |
| **Structured-output compliance** | **100%** | Every `/ask` returns a schema-valid `AskResponse`. Hard gate. |
| **Abstention correctness** | **Q12 must pass** | The no-evidence query returns no supporting evidence rather than a fabricated answer. |

### Definitions
- **Explicit claim** — a statement directly supported by retrieved text.
- **Inferred claim** — a reasonable synthesis the model labels as inferred, not stated verbatim.
- **Unknown** — the corpus poses but does not resolve the question (e.g. Q11); the system must say so.
- **Recall@k** — the target source document appears among the top-k retrieved chunks.

---

## 6. Evaluation method

1. Ingest `mock_documents/` into a clean index.
2. Run Q1–Q12 (plus synonym variants) through `/ask` with `top_k = 5`.
3. Score each response against the table in §4 (groundedness, recall, claim typing,
   abstention). The synonym metric is scored once dense embeddings are configured.
4. Compute the §5 metrics; M1 **passes** only if every threshold and every hard gate
   is met.

Automating this scoring (a benchmark runner) is an implementation task tracked with
the embedding/metadata/schema work — not part of this acceptance document.

---

## 7. Known limitations & future work
- Real-data (`data/P1 Affinity Evolution/`) evaluation is deferred until `.doc`/`.xls`
  ingestion exists (M2-adjacent).
- Synonym/semantic metric is only meaningful once dense embeddings replace the hash
  fallback.
- Groundedness and claim-typing scoring is initially manual; a runner can automate it.
