# TollgateRAG: Epistemic Risk Gating for Retrieval-Augmented Generation

[![Tests](https://img.shields.io/badge/pytest-24%20passed-brightgreen.svg)]()
[![Paper](https://img.shields.io/badge/IEEE%20Paper-6%20Pages%20PDF-blue.svg)](research_paper/main.pdf)
[![Python](https://img.shields.io/badge/Python-3.11%20%7C%203.12%20%7C%203.13-blue.svg)]()
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Empirically%20Verified-brightgreen.svg)]()

**TollgateRAG** is a retrieval-time defense framework designed to protect enterprise Retrieval-Augmented Generation (RAG) systems against **Corpus Constraint-Relaxation Attacks (CRAs)**. 

While existing safety classifiers (e.g., Meta Prompt Guard) detect explicit prompt injections, they fail against polite, professionally phrased documents that surreptitiously relax compliance boundaries (e.g., falsely stating that corporate travel limits or security clearances are waived). TollgateRAG intercepts retrieved passages before context assembly, verifying them against authoritative policy anchors using sentence-level Natural Language Inference (NLI) and multi-cue heuristics.

---

## 🏛️ System Architecture

TollgateRAG positions an **Epistemic Risk Gate** directly between candidate retrieval and generation context construction:

![TollgateRAG System Architecture](docs/paper/figures/fig_architecture.png)

### Three-Stage Gating Pipeline
1. **Dual-Index Retrieval:** Retrieves top-$k$ uncurated candidate passages ($\mathcal{C}_q$, $k=5$) and pairs each candidate with authoritative policy anchors ($\mathcal{A}_q$, $m=2$) retrieved from a cryptographically authenticated corporate knowledge base.
2. **Dual-Evidence Scoring:** 
   - **Neural NLI Verification:** A cross-encoder (`DeBERTa-v3-large`) computes fine-grained contradiction probabilities ($s_i^{\text{nli}}$) between candidate claims and policy anchors.
   - **Multi-Cue Heuristics:** A pattern engine extracts non-linear risk signals ($s_i^{\text{cue}}$) across authority assertions, urgency markers, and policy evasion cues.
3. **Risk Aggregation & Action Gate:** A Noisy-OR aggregator combines epistemic signals into an overall risk score ($r_i$). The tri-state gate assigns a deterministic verdict:
   - **ACCEPT** ($r_i < 0.30$): Admitted into the final context pool $\mathcal{G}_q$.
   - **WARN** ($0.30 \le r_i < 0.50$): Admitted with explicit uncertainty caveats.
   - **REJECT** ($r_i \ge 0.50$): Quarantined and replaced via deterministic clean retrieval backfill.

---

## 📊 Empirical Benchmark Results

Evaluated on a held-out test split of the 480-document Enterprise Policy Benchmark across 6 corporate domains (Finance, Travel, HR, Procurement, Legal, IT/Security) with 79 adversarial exposures, 13 clean benign exposures, and 12 benign-hard exposures.

![SOTA Baseline Comparison](docs/paper/figures/fig_sota_comparison.png)

### Performance vs. Baselines (Held-Out Test Split)

| Defense Mechanism | Adv. Rejection Rate (ARR) | Clean Benign FPR | Benign-Hard FPR | Query Contamination | Admitted Adv. Passages |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Plain RAG (No Defense)** | 0.0% (0/79) | 0.0% (0/13) | 0.0% (0/12) | 91.7% (22/24) | 31 (26 unique docs) |
| **Meta Prompt Guard 86M** | 3.8% (3/79) | 0.0% (0/13) | 0.0% (0/12) | 91.7% (22/24) | 31 (27 unique docs) |
| **BIPIA Keyword Filter** | 5.1% (4/79) | 0.0% (0/13) | 0.0% (0/12) | 91.7% (22/24) | 31 (26 unique docs) |
| **Strict Whitelist (Zero Trust)** | 100.0% (79/79) | 100.0% (13/13) | 100.0% (12/12) | 0.0% (0/24) | 0 (0 unique docs) |
| **TollgateRAG (Weighted Sum)** | 87.3% (69/79) | 0.0% (0/13) | 66.7% (8/12) | 29.2% (7/24) | 8 (7 unique docs) |
| **TollgateRAG (Noisy-OR - Default)** | **88.6% (70/79)** | **0.0% (0/13)** | **66.7% (8/12)** | **29.2% (7/24)** | **8 (7 unique docs)** |

### Key Highlights
- **Adversarial Interception:** Rejects **88.6%** of adversarial exposures and drops query context contamination from **91.7% down to 29.2%**.
- **Zero Benign Utility Penalty:** Achieves **0.0% False Positive Rate** on standard uncurated enterprise documents, whereas strict whitelisting blocks 100% of non-anchor enterprise knowledge.
- **Resplit Stability:** 4-seed topic-disjoint evaluation confirms consistent rejection (mean ARR **92.5%**, range 90.9%–94.1%, std **1.59%**).

<details>
<summary><b>📈 Additional Visualizations (Threshold Pareto Sweep & Adaptive Red-Teaming)</b></summary>
<br>

| Threshold Sensitivity & Operating Trade-Off | Closed-Loop Adaptive Red-Teaming (10 Rounds) |
| :---: | :---: |
| ![Threshold Sweep](docs/paper/figures/fig_pareto_threshold.png) | ![Adaptive Red Team](docs/paper/figures/fig_adaptive_redteam.png) |
| *Pareto sweep across threshold choices $\tau \in [0.1, 0.9]$ showing the chosen operating point at $\tau = 0.50$.* | *Evolutionary optimizer testing black-box (6.7% valid ASR) vs white-box (13.3% valid ASR) semantic evasion.* |

</details>

---

## 🚀 Quickstart

### 1. Installation

```bash
git clone https://github.com/pradeepv777/Rag-Risk.git
cd Rag-Risk

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: .\venv\Scripts\Activate.ps1

# Install pinned dependencies
pip install -r requirements.txt
```

### 2. Interactive Streamlit Demo

Launch the local interactive UI to test retrieval, inspection, and risk gating in real-time:

```bash
streamlit run app.py
```

Try sample queries:
- *"What is the daily meal budget for business trips?"*
- *"Is two-factor authentication required for VPN access?"*
- *"What's the real travel policy, is there an unlimited allowance?"*

---

## 🔬 Reproducibility & Testing

All benchmarks, numerical claims, and manuscript tables can be audited with automated test scripts:

```bash
# 1. Run full unit and integration test suite (24 tests)
pytest tests/ -q

# 2. Verify all 21 quantitative paper claims against raw experimental logs
python scripts/check_paper_numbers.py

# 3. Verify 100% of bibliographic citations against scholarly databases
python scripts/check_references.py

# 4. Scan commit history and files for zero secret / credential leaks
python scripts/check_no_secrets.py

# 5. Reproduce all benchmark tables and figures
python scripts/reproduce.py
```

---

## 📁 Repository Layout

```
.
├── tollgaterag/                    # Core Python library
│   ├── action_gate.py              # Tri-state decision logic and context backfill
│   ├── risk_scorer.py              # Dual-evidence scoring (NLI + Heuristic cues)
│   ├── nli_engine.py               # Cross-encoder transformer inference
│   ├── retriever.py                # ChromaDB vector retrieval engine
│   └── pipeline.py                 # End-to-end RAG query orchestration
├── research_paper/                 # Complete 6-page IEEE conference paper suite
│   ├── main.pdf                    # Compiled camera-ready PDF
│   ├── main.tex                    # LaTeX source manuscript
│   ├── references.bib              # Verified BibTeX citations
│   ├── claims_registry.json        # Machine-readable claims registry
│   ├── figures/                    # Vector PDF and high-res PNG figures
│   └── tables/                     # Generated publication LaTeX tables
├── data/                           # Ground-truth labels, splits, and queries
├── logs/                           # Authoritative raw experimental JSON logs
├── scripts/                        # Automated validation and reproduction scripts
├── tests/                          # Pytest suite (24 unit and integration tests)
├── app.py                          # Streamlit web demonstration
├── requirements.txt                # Pinned dependencies
├── RELEASE_MANIFEST.json           # SHA-256 release checksums and metadata
└── README.md                       # This documentation
```

---

## 📄 Research Paper & Citation

The complete research paper is compiled and formatted in IEEE conference two-column style:
- **PDF Manuscript:** [`research_paper/main.pdf`](research_paper/main.pdf)
- **Source Files:** [`research_paper/main.tex`](research_paper/main.tex)

```bibtex
@inproceedings{tollgaterag2026,
  title     = {TollgateRAG: Retrieval-Time Epistemic Risk Gating Against Semantic Policy-Relaxation Attacks in Enterprise RAG Systems},
  author    = {Anonymous Authors},
  booktitle = {Proceedings of the IEEE International Conference on Information Reuse and Integration (IRI)},
  year      = {2026}
}
```

---

## ⚖️ License

Distributed under the MIT License. See `LICENSE` for details.
