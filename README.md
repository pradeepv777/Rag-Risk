# TollgateRAG: Auditable Pre-Generation Epistemic Gating for RAG

[![IEEE Paper](https://img.shields.io/badge/Paper-IEEE%20Format%20PDF-red?style=for-the-badge&logo=adobeacrobatreader)](docs/paper/TollgateRAG_IEEE_Conference_Paper.pdf)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)
[![Python: 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue?style=for-the-badge&logo=python)](https://python.org)
[![Benchmark: 351 Docs](https://img.shields.io/badge/Benchmark-351%20Docs%20%7C%206%20Domains-green?style=for-the-badge)](data/sample_docs/)

> **TollgateRAG** is an auditable pre-generation epistemic risk gating framework for Retrieval-Augmented Generation (RAG) systems. Operating between dense/sparse retrieval and LLM context assembly, TollgateRAG computes transparent multidimensional risk scores per candidate document, enforcing **Accept / Warn / Reject** gating to eliminate indirect prompt injection, policy override attacks, and corpus poisoning without sacrificing benign knowledge utility.

---

## 🎯 Research Highlights & Key Results

Evaluated on **Tollgate-Bench** (351 enterprise policy documents across 6 domains and 60 targeted queries), alongside **PoisonedRAG** (USENIX Security 2025) and **Microsoft BIPIA**:

| Defense Mechanism | Adversarial Reject Rate (ARR ↑) | Adversarial Flag Rate (Warn+Reject ↑) | Benign False Positive Rate (FPR ↓) | Context Contamination Rate (CCR ↓) | Contaminated Docs in Context ↓ |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Plain RAG (Unprotected)** | 0.0% | 0.0% | **0.0%** | 61.7% | 73 |
| **Strict Whitelist (Zero Trust)** | 100.0% | 100.0% | 100.0% *(destroys utility)* | **0.0%** | **0** |
| **Injection-Keyword Filter** | 0.6% | 0.6% | **0.0%** | 61.7% | 73 |
| **Prototype Regex Gater** | 3.5% | 33.9% | **0.0%** | 31.7% | 23 |
| **TollgateRAG (Neural NLI - Ours)** | **89.7%** | **94.2%** | **0.0%** | **3.3%** | **2** *(97.3% reduction)* |

### 🛡️ Resistance Against Adaptive Red-Teaming

Against an automated LLM adaptive attacker employing 4 adversarial evasion strategies (*authority mimicking*, *synonym obfuscation*, *hypothetical framing*, *markdown steganography*):
- **Traditional Keyword Filter:** Collapses to **0.0%** detection.
- **TollgateRAG (Ours):** Preserves **70.0% strict rejection** and **80.0% quarantine rate** by evaluating semantic premise-hypothesis contradiction against immutable policy anchors.

---

## 🏛️ Architecture & Epistemic Gating Pipeline

```
┌─────────────────┐       ┌────────────────────────┐       ┌──────────────────────────┐
│ User Query      │ ───►  │ Retrieval Engine       │ ───►  │ Candidate Documents (k)  │
└─────────────────┘       │ (Dense / Sparse Index) │       └────────────┬─────────────┘
                          └────────────────────────┘                    │
                                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                       TOLLGATERAG 5-SIGNAL RISK SCORING ENGINE                              │
│                                                                                             │
│  1. Provenance Trust Score        (S_trust)  ──► Weight: 0.20                              │
│  2. Temporal Freshness Decay      (S_fresh)  ──► Weight: 0.05                              │
│  3. Indirect Injection Classifier (S_inj)    ──► Weight: 0.25                              │
│  4. Authority Claim Score         (S_auth)   ──► Weight: 0.15                              │
│  5. Neural NLI Anchor Conflict    (S_nli)    ──► Weight: 0.35 (DeBERTa-v3 Cross-Encoder)   │
└───────────────────────────────────────────────┬─────────────────────────────────────────────┘
                                                │
                                                ▼
                                    ┌───────────────────────┐
                                    │ Composite Risk Metric │
                                    │ R(d) = Σ (w_i · S_i)  │
                                    └───────────┬───────────┘
                                                │
                     ┌──────────────────────────┼──────────────────────────┐
                     ▼                          ▼                          ▼
            R(d) < θ_warn (0.25)    θ_warn ≤ R(d) < θ_rej (0.45)    R(d) ≥ θ_rej (0.45)
            ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
            │     ACCEPT      │        │      WARN       │        │     REJECT      │
            └────────┬────────┘        └────────┬────────┘        └────────┬────────┘
                     │                          │                          │
                     ▼                          ▼                          ▼
              Full LLM Context         Quarantined Context        Dropped from Context
                                       + Epistemic Warning        + Audit Review Queue
```

---

## 📊 5-Signal Mathematical Formulation

For each retrieved candidate document $d$, the composite risk $R(d) \in [0, 1]$ is defined as:

$$R(d) = w_{\text{nli}} S_{\text{nli}}(d, \mathcal{A}) + w_{\text{inj}} S_{\text{inj}}(d) + w_{\text{trust}} S_{\text{trust}}(d) + w_{\text{auth}} S_{\text{auth}}(d) + w_{\text{fresh}} S_{\text{fresh}}(d)$$

Where:
1. **$S_{\text{nli}}(d, \mathcal{A})$ ($w=0.35$):** Semantic contradiction probability computed via DeBERTa-v3 cross-encoder between unverified candidate $d$ and verified domain anchor documents $\mathcal{A}$.
2. **$S_{\text{inj}}(d)$ ($w=0.25$):** Heuristic and embedding-based classifier detecting instruction overrides, delimiter tampering, and prompt jailbreaks.
3. **$S_{\text{trust}}(d)$ ($w=0.20$):** Provenance penalty ($0.0$ for signed/verified corporate sources, $0.8$ for unverified crowdsourced tips, $1.0$ for untrusted endpoints).
4. **$S_{\text{auth}}(d)$ ($w=0.15$):** Linguistic penalty detecting unauthorized normative assertions (*"supersedes official policy"*, *"effective immediately"*).
5. **$S_{\text{fresh}}(d)$ ($w=0.05$):** Exponential temporal decay penalty penalizing stale documentation.

---

## 🔬 Benchmark Dataset (Tollgate-Bench)

Tollgate-Bench provides **351 structured policy documents** across 6 high-stakes enterprise domains:

| Enterprise Domain | Verified Anchors | Benign Unverified | Direct Injections | Subtle Adversarial | Evasive Adversarial | Total |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **HR & Employee Benefits** | 15 | 15 | 10 | 10 | 10 | **60** |
| **IT & Identity Access (IAM)** | 15 | 15 | 10 | 10 | 10 | **60** |
| **Financial & Travel Accounting**| 14 | 14 | 10 | 10 | 10 | **58** |
| **Healthcare & HIPAA Compliance**| 15 | 15 | 10 | 10 | 10 | **60** |
| **Cloud Infrastructure (DevOps)**| 15 | 14 | 10 | 10 | 10 | **59** |
| **Legal & Non-Disclosure (NDA)** | 14 | 14 | 10 | 10 | 10 | **58** |
| **Total Corpus** | **88** | **87** | **60** | **60** | **56** | **351** |

Ground truth annotations: `data/labels.json`  
Evaluation queries (60 queries across all 6 domains): `data/eval_queries.json`

---

## 🚀 Quick Start Guide

### 1. Installation

```bash
git clone https://github.com/Pavanyata999/Rag-Risk.git
cd Rag-Risk

# Create and activate virtual environment
python -m venv venv
# On Windows PowerShell:
.\venv\Scripts\Activate.ps1
# On macOS/Linux:
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Ingest the Benchmark Corpus

```bash
# Ingests all 351 documents into vector store and sets up verified anchors
python -m tollgaterag.ingest
```

### 3. Run Pipeline on Any Query

```bash
python -m tollgaterag.pipeline "What is the corporate international travel reimbursement limit?"
```

### 4. Run SOTA Baselines & Benchmarks

```bash
# Evaluate TollgateRAG vs Plain RAG, Whitelisting, and Keyword Filters
python -m tollgaterag.eval_baselines

# Run automated Adaptive Red-Teaming loop
python -m tollgaterag.red_team

# Run Leave-One-Out Ablation & Threshold Sweeps
python -m tollgaterag.ablation

# Run USENIX PoisonedRAG evaluation suite
python -m tollgaterag.eval_poisonedrag
```

### 5. Launch Interactive Web UI

```bash
streamlit run app.py
```

---

## 📄 IEEE Conference Paper & Publication Artifacts

The complete camera-ready academic paper is available in the repository:
- **PDF Manuscript:** [`docs/paper/TollgateRAG_IEEE_Conference_Paper.pdf`](docs/paper/TollgateRAG_IEEE_Conference_Paper.pdf) (4 pages, camera-ready)
- **LaTeX Source:** [`docs/paper/main.tex`](docs/paper/main.tex) and [`docs/paper/references.bib`](docs/paper/references.bib)
- **Publication Figures:** [`docs/paper/figures/`](docs/paper/figures/) (300 DPI vector charts)
- **Project Specifications & Transcript:** [`PROJECT_DETAILS_AND_IEEE_PAPER.txt`](PROJECT_DETAILS_AND_IEEE_PAPER.txt)

### Citation (BibTeX)

```bibtex
@inproceedings{yatapavankumar2026tollgaterag,
  author    = {Yatapavankumar and Pradeep V},
  title     = {TollgateRAG: Auditable Pre-Generation Epistemic Risk Gating for Agentic Retrieval-Augmented Generation},
  booktitle = {Proceedings of the IEEE International Conference on Secure AI Systems},
  year      = {2026},
  pages     = {1--4}
}
```

---

## 📁 Repository Structure

```
├── .planning/                  # GSD planning specs, roadmaps, and execution state
├── data/
│   ├── sample_docs/            # 351 benchmark documents across 6 enterprise domains
│   ├── benchmarks/             # USENIX PoisonedRAG (750 samples) and Microsoft BIPIA
│   ├── eval_queries.json       # 60 targeted multi-domain evaluation queries
│   └── labels.json             # Ground truth classification labels
├── docs/
│   └── paper/
│       ├── TollgateRAG_IEEE_Conference_Paper.pdf # Compiled camera-ready PDF
│       ├── main.tex            # IEEE LaTeX conference manuscript
│       ├── references.bib      # Academic bibliography
│       └── figures/            # 300 DPI vector plots (SOTA, RedTeam, Pareto, Ablation)
├── scripts/
│   ├── build_paper_pdf.py      # Automated LaTeX/PDF compilation tool
│   ├── generate_large_benchmark.py # 351-doc enterprise benchmark generator
│   └── generate_paper_figures.py   # Publication figures generator
├── tollgaterag/
│   ├── nli_engine.py           # Neural NLI cross-encoder with LRU caching
│   ├── risk_scorer.py          # 5-signal composite risk scoring engine
│   ├── baselines.py            # SOTA defense baselines (Plain, Whitelist, Keyword, Guard)
│   ├── red_team.py             # Automated LLM adaptive red-teaming loop
│   ├── ablation.py             # Leave-one-out ablation & threshold sensitivity sweeps
│   ├── eval_baselines.py       # Comparative benchmarking evaluation script
│   ├── eval_poisonedrag.py     # PoisonedRAG test suite
│   ├── retriever.py            # Dense & sparse candidate retrieval engine
│   ├── action_gate.py          # 3-tier action gating engine (Accept/Warn/Reject)
│   ├── provenance.py           # Immutable audit log and human review queue dispatch
│   └── pipeline.py             # End-to-end RAG gating orchestration
├── app.py                      # Streamlit interactive demonstration UI
├── PROJECT_DETAILS_AND_IEEE_PAPER.txt # Full project spec and text paper transcript
└── requirements.txt            # Python dependencies
```

---

## 📜 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
