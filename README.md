<div align="center">

# UJ Advisor
### A Hybrid Agentic RAG-Based Chatbot for Arabic Academic Advising

*Unifying fragmented university records into a single, intelligent, agentic advising hub.*

[![Status](https://img.shields.io/badge/status-research%20prototype-blue)]()
[![Language](https://img.shields.io/badge/language-Arabic%20(MSA)-green)]()
[![LLM](https://img.shields.io/badge/response%20LLM-GPT--4-orange)]()
[![Embeddings](https://img.shields.io/badge/embeddings-multilingual--e5--large-purple)]()
[![Vector DB](https://img.shields.io/badge/vector%20db-ChromaDB-yellow)]()
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey)]()

</div>

---

## 📌 Overview

**UJ Advisor** is a hybrid **agentic Retrieval-Augmented Generation (RAG)** framework built to solve a very real, very painful problem at the **University of Jeddah**: academic advising data is scattered across disconnected sectors — student affairs, admissions and registration, faculty records, internship listings — forcing advisors to manually stitch together a student's academic picture before they can even start advising.

UJ Advisor replaces that manual process with **one secure, unified AI-powered portal** accessible to academic advisors, faculty, and students. It doesn't just answer questions from documents — it **reasons about which tool to use**, executes structured database functions, runs predictive ML models, retrieves institutional knowledge, and synthesizes everything into a single grounded Arabic response.

This repository contains the full implementation: the agentic pipeline, the RAG knowledge base, the SQL function-execution layer, two predictive classification models (academic risk + support needs), the evaluation notebooks, and the web interface.

---

##  Why This Exists

> Existing university student-record systems don't provide a fully integrated advising platform, and available tools are neither intelligent nor designed for cross-department integration.

The result: delayed decisions, inconsistent information, and advisors spending more time *searching* than *advising* — especially painful under high advisor-to-student ratios.

**UJ Advisor's objectives:**
- Unify academic-progress data into one decision-support framework
-  Centralize academic knowledge, regulations, and advising resources
-  Process structured student records through **controlled functions**, not free-form/unrestricted SQL generation
-  Flag students who may be **academically at-risk** or in need of support, early
-  Deliver context-aware, tool-grounded answers in Arabic within a single advising workflow

---

##  System Architecture

UJ Advisor is built around a **Tool-Augmented Agent** that sits between the user's query and four specialized sub-agents. It doesn't generate answers from memory — it plans, retrieves evidence, executes tools, and only then responds.

```
User Query
   │
   ▼
Autocorrect ──▶ LLM Query Rewriting (context-aware) ──▶ Updated Query
   │
   ▼
┌─────────────────────────── Tool-Augmented Agent ───────────────────────────┐
│                                                                              │
│   Router ──▶ Orchestrator ──▶ Response ──▶ Confidence ──(low conf.)──┐     │
│     │              │              │             │                    │     │
│     │              │              │             └─ High confidence ──┘     │
│     ▼              ▼                                                       │
│  routes to:    executes selected modules                                   │
└──────┬────────────────┬───────────────┬──────────────────┬────────────────┘
       │                │               │                  │
       ▼                ▼               ▼                  ▼
 ┌───────────┐   ┌─────────────┐  ┌────────────┐   ┌──────────────────┐
 │ RAG Agent │   │  SQL Agent  │  │ Classifiers│   │ External Fallback│
 │ (Vector DB)│  │(SQL Database)│ │(risk/support)│ │   (Internet)      │
 └───────────┘   └─────────────┘  └────────────┘   └──────────────────┘
       │                │               │                  │
       └────────────────┴───────────────┴──────────────────┘
                              │
                              ▼
                      Final Arabic Response
                              │
                              ▼
                        Update Memory
                  (chat history + structured state)
```

### 1️⃣ Tool-Augmented Agent (the brain)

The control layer. Four internal components:

| Component | Role |
|---|---|
| **Router** | Converts the rewritten query into a structured routing decision — intent, required modules (`rag`/`sql`/`risk`/`support`/`hybrid`), relevant collections/tables, retrieval strategy, and a confidence score. Uses LLM planning + deterministic safeguards (Arabic text normalization, student-ID extraction, path guarding). |
| **Orchestrator** | Executes whatever the Router decided — activates the RAG pipeline, resolves student context for classification models, runs SQL functions — and records execution feedback (`sql_ok`, `rag_has_answer`, etc.) for possible replanning. |
| **Response** | Synthesizes the final Arabic answer from collected evidence via a controlled prompt: answer only what was asked, never expose internal variables, never invent unsupported facts. |
| **Confidence** | Judges whether the routing confidence *and* the evidence quality are sufficient. If not, triggers **one bounded refinement attempt** (replan/re-route/re-execute) — capped to prevent infinite loops. |

Memory is two-layered:
- **Chat-history memory** — recent turns, used to rewrite follow-up questions into standalone queries.
- **Structured conversation-state memory** — active topic, last route, student ID/name, advisor name, course key, previous SQL/RAG outputs — critical because Arabic advising follow-ups routinely omit identifiers ("and does *he* need training?").

### 2️⃣ RAG Agent — Institutional Knowledge Retrieval

- **Router-aware retrieval**: starts in the collections the Router suggested; expands to all collections if evidence is weak.
- **Hybrid retrieval**: dense semantic search (embeddings) + **BM25 lexical search**, fused via **Reciprocal Rank Fusion**.
- **Deep KB Retrieval**: expands the query into multiple variants, reruns retrieval, merges/deduplicates — an internal fallback, *not* a web search.
- **3-tier evidence priority**: (1) internal knowledge base → (2) University of Jeddah domain search via Tavily → (3) general web search — only escalating a tier when evidence is genuinely insufficient.
- If the query is still ambiguous with no strong evidence → the system **asks a clarifying question** instead of guessing.
- Final generation via **GPT-4o-mini**, constrained to cite only from retrieved evidence.

### 3️⃣ SQL Agent — Controlled Structured-Data Access

Deliberately **does not generate free-form SQL**. Instead:

1. **Retrieval-first layer** — checks if the answer already exists in a vectorized academic-data context, judged by an LLM-based grounded judge returning `answered | needs_functions | no_answer`.
2. **Function-routing layer** — if functions are needed, a router selects from a **fixed registry of predefined advising functions** (never arbitrary queries), extracting signals like student ID, course key, advisor name, counts, rankings.
3. Execution only proceeds after argument validation.

<details>
<summary><b>📋 Full function registry (click to expand)</b></summary>

| Category | Functions |
|---|---|
| Student Profile | `resolve_student_profile_only`, `load_student_profile` |
| Academic Status | `student_academic_case`, `get_student_course_completion_summary` |
| Training Status | `training_students`, `get_student_training_status`, `training_needed_students_with_names` |
| Course Queries | `course_lookup`, `get_course_enrollment_count`, `students_not_passed_course` |
| Advisor Queries | `advisor_students`, `students_count_by_advisor`, `advisor_full_summary` |
| Ranking & Aggregation | `advisor_ranking`, `training_needed_count_by_advisor`, `backlog_students_count_by_advisor` |
| Special Cases | `exceptional_opening` |
| Model-Based Analysis | `student_model_analysis`, `student_factor_analysis` |

</details>

### 4️⃣ Classification Agent — Predictive Evidence

Two **pre-trained models**, invoked as tools (not retrained at inference time), triggered only when the Router detects a predictive/diagnostic intent:

| Model | Task | Architecture | Input Features |
|---|---|---|---|
| **At-Risk Classifier** | High / moderate / low academic-risk label | **TabNet v2** classifier | Attendance, participation, quizzes, assignments, midterm score, project performance |
| **Need-Support Classifier** | Support / no-support label + primary stress factor | Hybrid **Autoencoder reconstruction risk + Directional Univariate risk scoring** | Study hours, sleep duration, exercise, social-media usage, financial stress, peer pressure, mental-stress level, diet quality, family support, cognitive distortions |

Feature resolution priority: **database values > manually stated values** (DB is authoritative; user-provided values only fill gaps). If required features are missing, the tool returns a controlled failure — never a hallucinated prediction.

---

##  Knowledge Base & Data Pipeline

### Data Sources
| Category | Contents |
|---|---|
| Academic Regulations & Policies | Executive rules, academic calendar, registration rules, transfer/specialization guidelines, internship policies |
| Curriculum & Study Plans | Degree plans per major, prerequisite chains, electives, credit hours |
| Faculty Directory | Names, contact info, office locations |
| Internship & Training Data | Guidelines, requirements, company listings (survey-collected) |
| Student Activities & Clubs | Training opportunities, clubs, volunteering, peer-helper dataset |
| Expansive Student Records | Profiles, registration, performance indicators, backlog, training status, synthetic augmentation |

Collected from: PDFs, HTML pages, scanned images (OCR), surveys, manual data entry, and synthetic generation.

### Preprocessing Pipeline
`OCR (Tesseract) → HTML cleaning → Arabic text normalization → table reconstruction → dedup (exact + fuzzy) → JSON unification → metadata annotation`

### Indexing
- **Chunking**: boundary-aware, 1–3 paragraphs per chunk, preserving sentence integrity
- **Embedding model**: `intfloat/multilingual-e5-large` (1024-dim dense vectors)
- **Vector store**: **ChromaDB**, split into 14 collections — `regulations`, `transfer_rules`, `degree_plans`, `electives`, `student_helpers`, `faculty_offices`, `clubs`, `specialization`, `coop_replies`, `academic_calendar`, `coop_rules`, `activities`, `course_bundles`, `expansive_student_records`

---

##  Results

### 1. Routing Decision Performance
*(Evaluated on 50 labeled queries)*

| Metric | Score |
|---|---|
| Mode Accuracy | **0.90** |
| RAG Decision Accuracy | **0.92** |
| SQL Decision Accuracy | **0.94** |
| Model (Classification) Decision Accuracy | **0.88** |
| Exact Decision Match (all components agree) | **0.84** |

📌 Errors concentrate almost entirely in the **hybrid** category (queries requiring both SQL + RAG) — confirming that coordinating multiple evidence sources is the hardest routing case, not any single path.

### 2. Retrieval Layer Evaluation

| Metric | Score |
|---|---|
| Strict Router Accuracy | 88% |
| Router → Collection Recall | 100% |
| Global Fallback Rate | 12% |
| Collection Hit@1 / @3 / @5 / @10 | 100% |
| Collection Precision@1 | 100% |
| Collection Precision@3 | 92.33% |
| Collection Precision@5 | 88.6% |
| Collection Precision@10 | 82.7% |
| Collection MRR | 100% |
| Collection nDCG@10 | 97.99% |

### 3. Embedding Model Comparison
*(Hybrid metric: 80% cosine similarity + 20% keyword overlap)*

| Model | Avg. Performance |
|---|---|
| **multilingual-e5-large**  (selected) | **78–82%** |
| gte-large | 74–78% |
| Arabic Matryoshka | 70–74% |
| GATE | 68–72% |
| LaBSE | 60–66% |
| AraVec | 52–58% |
| Arab2vec | 50–55% |

### 4. LLM Comparison (Response Generation)

| LLM | F1 Score | BLEU Score | Cosine Similarity |
|---|---|---|---|
| **GPT-4**  (selected) | **0.6770** | **47.86** | **0.9435** |
| JAIS-13B-Chat | 0.6600 | 43.38 | 0.6993 |
| Llama-3.2-3B-Instruct | 0.2845 | 15.05 | 0.8854 |
| AraGPT | 0.0234 | 0.097 | 0.0418 |

*Mistral was evaluated but excluded from the final comparison due to inference latency.*

### 5. Classification Models

| Tool | Model | Reported Performance | Chatbot Output |
|---|---|---|---|
| At-Risk Classification | TabNet v2 | **96.53% test accuracy**, Macro F1 = **0.95** | Low / Moderate / High risk label |
| Need-Support Classification | Hybrid Autoencoder + Directional Univariate | Top-5% enrichment = **100%** high-risk, Top-10% = **98.5%** | Support / No-support + primary stress factor |

### 6. End-to-End Evaluation — RAGAS
*(Benchmark: 100 academic advising queries with ground-truth answers)*

| Metric | Score |
|---|---|
| **Faithfulness** | **0.893** |
| Context Recall | 0.805 |
| Context Precision | 0.730 |
| Answer Relevancy | 0.716 |

High faithfulness (0.89) confirms low hallucination — answers stay grounded in retrieved/executed evidence rather than model memory.

### 7. User Study
*(16 participants: 8 students, 5 faculty, 3 academic advisors)*

-  **93.8%** said they would use the system if it were available (15/16 "Yes", 1 "Maybe")
-  Ease of use was the single strongest-rated dimension (11/16 respondents most impressed)
-  10/16 felt it saved meaningful time and effort
-  8/16 reported at least one issue — most commonly *unclear answers* (4 reports), followed by unrelated responses, slowness, and lack of English support (1 each)

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Response Generation LLM | GPT-4 (selected after benchmarking vs. JAIS-13B, Llama-3.2-3B, AraGPT, Mistral) |
| RAG Answer Generation | GPT-4o-mini |
| Embedding Model | `intfloat/multilingual-e5-large` |
| Vector Database | ChromaDB (14 domain-specific collections) |
| Lexical Retrieval | BM25 (fused with dense retrieval via Reciprocal Rank Fusion) |
| Structured Data | SQL database + controlled function registry |
| OCR | Tesseract |
| At-Risk Model | TabNet v2 |
| Need-Support Model | Hybrid Autoencoder + Directional Univariate risk scoring |
| External Fallback Search | Tavily Search (UJ-domain-first, then general web) |
| Evaluation Framework | RAGAS methodology |
| Frontend | HTML / CSS / JS (student, advisor, employee dashboards) |

---

##  Repository Structure

```
.
├── interface/                              # Web front-end
│   ├── index (1).html
│   ├── student.html
│   ├── advisor.html
│   ├── employee.html
│   ├── app.js
│   ├── student.js
│   ├── styles.css
│   ├── logo-uj.svg
│   ├── right-shape.svg
│   ├── render.yaml
│   └── README.md
│
├── UJADV_final.ipynb                       # Main end-to-end pipeline notebook
├── UJADVISE (1).zip                        # Packaged system bundle
├── At-risk-SRP1.ipynb                      # At-Risk (TabNet v2) model notebook
├── FINAL_SUPPORT_WDEMO.ipynb               # Need-Support model + demo notebook
├── RetEVALUATION.ipynb                     # Retrieval/embedding evaluation notebook
│
├── student_db_updated_no_training (1) (1) (1).db   # Structured student-record SQLite DB
│
├── UJ_100_questions_decision_eval_dataset.xlsx     # 100-query RAGAS benchmark set
├── UJ_decision_eval_50_questions.xlsx              # 50-query routing evaluation set
├── decision_routing_eval_results (2).xlsx          # Routing confusion-matrix results
├── retrieval_eval_RESULT.xlsx                      # Retrieval metrics (Hit@K, MRR, nDCG)
├── ragas_results (1).xlsx                          # RAGAS full results
├── ragas_summary (2).csv                           # RAGAS aggregated metrics
├── ragas_details (2).csv                           # RAGAS per-query breakdown
│
├── .gitignore
└── README.md
```

---

##  Live Interface

The `interface/` folder contains three role-based dashboards, all connecting to the same agentic backend:

- **`student.html`** — student-facing chat interface for advising queries
- **`advisor.html`** — advisor dashboard with student-status overview
- **`employee.html`** — administrative/staff view

Deployment config is defined in `interface/render.yaml`.

---

##  Limitations & Honest Caveats

- **Hybrid queries** (requiring both SQL + RAG together) remain the weakest routing category — the confusion matrix shows most misclassifications land here.
- **No public Arabic academic-advising or student-record dataset exists** — a portion of the structured data used for training/testing is **synthetically generated**, which caps how confidently the predictive models generalize to real institutional data.
- **OCR quality directly affects retrieval quality** — scanned/fragmented Arabic tables required manual correction.
- Some users still reported unclear or partially unrelated answers on complex, multi-hop advising scenarios, consistent with known RAG hallucination risk under weak retrieval.
- This is a **decision-support tool for advisors**, not a replacement for human advising judgment.

---

##  Future Work

- Live integration with real university platforms and continuously updated institutional databases (not synthetic/static snapshots)
- Move toward a fully autonomous multi-agent architecture with continuous learning from advising outcomes
- Stronger Arabic-native embedding/LLM models
- Multilingual advising support
- Voice-based interaction

---

##  Team

Developed by Artificial Intelligence students, **College of Computer Science and Engineering, University of Jeddah**:

- Atheer Alsulami
- Lujain Alsuoilme
- Rahaf Jelan
- Ghala Alotaibi
- Shahad Alzahrani

**Project Advisor:** Dr. Hanen Himdi

---

## 📖 Citation

If you reference this work, please cite:

```bibtex
@article{alsulami2026ujadvisor,
  title   = {UJ Advisor: A Hybrid Agentic RAG Based Chatbot for Arabic Academic Advising},
  author  = {Alsulami, Atheer and Alsuoilme, Lujain and Jelan, Rahaf and Alotaibi, Ghala and Alzahrani, Shahad and Himdi, Hanen},
  journal = {Journal Not Specified},
  year    = {2026},
  note    = {Department of Artificial Intelligence, University of Jeddah}
}
```

---



---

<div align="center">

*Built with 🧠 agentic reasoning, 📚 retrieval grounding, and a genuine dislike of fragmented university systems.*

</div>
