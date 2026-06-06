## CRISPR-Cas9 Guide RNA Design System with RAG + LLM Ranking Pipeline

A biomedical AI system that ranks optimal CRISPR guide RNAs using
sequence scoring and retrieval-augmented generation, reducing manual
guide RNA selection from hours of database work to a structured
computational pipeline.

---

## TL;DR

A CRISPR-Cas9 guide RNA design system that:
- Generates candidate guide RNAs from target gene sequences using PAM scanning
- Scores and ranks candidates using biologically grounded rules (Doench 2016)
- Uses RAG + LLM to explain ranked outputs with literature backing
- Benchmarks standalone LLM vs RAG+LLM across 8 disease genes
- Covers HIV, Sickle Cell, Huntington's, Duchenne MD, Cystic Fibrosis,
  BRCA Cancer, Blindness (LCA), and Leukemia

---

## Problem Statement

Designing CRISPR guide RNAs is slow and manual. For any target gene,
scientists must evaluate hundreds of candidate 20-nucleotide sequences
against multiple biological criteria, cross-reference published
literature, and make a judgment call about which sequence to order from
a synthesis company. This process typically takes 8-10 hours per gene
before any wet lab work begins.

This project automates and ranks guide RNA candidates in minutes using
computational biology rules combined with LLM-based reasoning grounded
in retrieved literature.

---

## The 3 Problems in CRISPR Guide RNA Design

CRISPR-based gene editing involves three distinct computational
problems. This project addresses only Problem 3 in Phase 1, with
Problems 1 and 2 scoped for future phases.

| Problem | Input | Output | Status |
|---------|-------|--------|--------|
| 1. Mutation Detection | Patient DNA sequence | Which mutation exists | Phase 2 |
| 2. Target Selection | Disease knowledge + literature | Which gene to edit | Phase 3 |
| 3. Guide RNA Design | Target gene (already chosen) | Ranked sgRNA sequences | Phase 1 (this project) |

The distinction matters: a wet lab scientist already knows they want
to knock out CCR5 for HIV resistance. Their actual problem is "which
of the 258 possible cut sites in CCR5 should I use?" This project
solves that question.

---

## Biological Background

CRISPR-Cas9 is a gene editing system that cuts DNA at a precise
location. The guide RNA (sgRNA) is a 20-nucleotide sequence that
directs the Cas9 enzyme to the exact target site. Without the guide
RNA, Cas9 has no idea where to cut.

**PAM requirement:** Cas9 can only cut where an NGG sequence appears
immediately after the target. Every NGG in the genome is a potential
cut location. A human gene typically contains 200-500 NGG sites,
each generating a candidate guide RNA.

**Why ranking matters:** Not all candidates are equally effective.
A guide RNA with poor GC content will fall off the target. One with
a poly-T stretch will terminate transcription before the guide is
made. One that matches multiple genome locations causes unintended
edits. Ranking separates the top 5 usable candidates from the 250
that would fail in the lab.

---

## System Architecture

```
User selects disease or gene name
            |
            v
Disease-gene map resolves target
(e.g., HIV -> CCR5, Sickle Cell -> HBB)
            |
            v
Biopython fetches reference mRNA sequence from NCBI
(e.g., CCR5 = 10,247 bp via Entrez API)
            |
            v
PAM scanner finds all NGG sites in sequence
(e.g., CCR5 yields 258 candidate positions)
            |
            v
Each candidate evaluated by 5-criterion scoring engine
GC content + Poly-T + Seed region + Complexity + Position
(Doench 2016 rules, 100 point scale)
            |
            v
Candidates filtered (score >= 40) and ranked
Top 10 per gene saved to ranked_guides.json
            |
      +-----+-----+
      |           |
      v           v
  Cell 3        Cell 4
  LLM alone     RAG + LLM
  (no context)  PubMed papers + ranked guides
      |           |
      +-----+-----+
            |
            v
        Cell 5
        Side-by-side evaluation
        5 metrics across 8 genes
        Plots + final summary table
```

---

## Diseases and Target Genes

| Disease | Gene | Rationale |
|---------|------|-----------|
| HIV | CCR5 | CCR5 is the co-receptor HIV uses to enter T-cells |
| Sickle Cell | HBB | FDA-approved CRISPR therapy Casgevy targets HBB (2023) |
| Huntington's | HTT | CAG repeat expansion causes toxic protein accumulation |
| Duchenne MD | DMD | Exon skipping via CRISPR restores dystrophin reading frame |
| Cystic Fibrosis | CFTR | F508del mutation disrupts chloride channels in lungs |
| BRCA Cancer | BRCA1 | Mutations impair DNA repair, studied for cancer research |
| Blindness (LCA) | CEP290 | First in-vivo CRISPR trial injected directly into the eye |
| Leukemia | BCR | BCR-ABL fusion drives chronic myeloid leukemia |

---

## Guide RNA Scoring System

The scoring engine is based on published criteria from Doench et al.
2016 (referenced below). Five criteria combine into a 100-point score.
Each criterion maps to a defined numeric range with biologically
motivated thresholds rather than arbitrary weights.

| Criterion | Weight | What it measures |
|-----------|--------|-----------------|
| GC content | 25 pts | Binding stability. Optimal 40-70%, peak score at 55% |
| Poly-T penalty | 15 pts | Four or more consecutive T's terminate transcription |
| Seed region | 25 pts | Last 12 nucleotides before PAM control specificity |
| Sequence complexity | 20 pts | Low complexity sequences hit more genome locations |
| Genomic position | 15 pts | Coding region guides (10-70% of gene) are most effective |

Each function returns a continuous score within its weight range rather
than a binary pass/fail. For example, GC content at 55% returns the
full 25 points, GC at 45% returns partial credit, and GC below 30%
returns zero. This produces a spread of scores across candidates
(typically 65-92) that makes ranking meaningful rather than clustering
all candidates at the same value.

Off-target scoring in this project uses sequence complexity as a
heuristic proxy, measuring dinucleotide repeats, homopolymer runs,
and terminal GC clamping as indirect indicators of genome-wide
repetitiveness. Real genome-wide off-target analysis requires Bowtie
alignment against the full 3GB human genome reference, which is beyond
current hardware scope and is planned for Phase 2.

**Reference:** Doench et al. (2016). Optimized sgRNA design to maximize
activity and minimize off-target effects of CRISPR-Cas9.
https://pmc.ncbi.nlm.nih.gov/articles/PMC4744125/

---

## Tech Stack

| Component | Tool |
|-----------|------|
| Sequence retrieval | Biopython + NCBI Entrez API |
| Literature retrieval | PubMed via Entrez esearch + efetch |
| Embeddings | sentence-transformers/all-MiniLM-L6-v2 |
| Vector store | ChromaDB with MMR retrieval |
| RAG orchestration | LangChain |
| LLM | Qwen/Qwen2.5-7B-Instruct (4-bit quantization via bitsandbytes) |
| Evaluation | Custom metrics + Matplotlib |
| Hardware | NVIDIA RTX 3060 12GB VRAM |

---

## Dataset

| Source | Content | How used |
|--------|---------|----------|
| NCBI Nucleotide | Reference gene sequences (mRNA) | PAM scanning + guide design |
| PubMed via Entrez | 671 CRISPR research paper abstracts | RAG document corpus |
| Cell 2 output | Ranked guide RNA tables | Injected into RAG as structured documents |

Gene sequences are frozen to disk on first fetch to avoid re-hitting
NCBI rate limits on subsequent runs. Papers are fetched using 10 search
queries targeting different CRISPR subtopics for broad coverage.

---

## Cell Structure

| Cell | Purpose |
|------|---------|
| Cell 1 | Setup, NCBI gene sequence fetch, PubMed paper fetch, data freezing |
| Cell 2 | PAM scanning, guide RNA generation, 5-criterion scoring, ranking |
| Cell 3 | LLM standalone baseline — 8 CRISPR questions with no context |
| Cell 4 | RAG + LLM — retrieves ranked guides + papers, generates grounded answers |
| Cell 5 | Side-by-side evaluation and visualisation across all 5 metrics |

---

## Evaluation

Cells 3 and 4 are compared across 8 disease genes using 5 metrics.

| Metric | What it measures | Why it matters |
|--------|-----------------|----------------|
| Guide RNA Citation Rate | Does response reference actual computed 20-mer sequences? | LLM alone scores 0 — it cannot cite sequences that did not exist in training data |
| Score Reference Rate | Does response cite numerical scores like 90/100? | Measures whether RAG grounding translates to specific evidence use |
| Specificity | Does response use gene-specific disease terminology? | Distinguishes focused clinical answers from generic CRISPR advice |
| Safety Disclaimer Rate | Does response recommend experimental validation? | Measures responsible AI behavior in a medical context |
| Response Relevance | Is response on-topic for the queried gene and disease? | Guards against hallucinated or off-topic generation |

Guide RNA citation is the most inarguable metric: RAG retrieves the
Cell 2 output and the LLM references specific sequences. The standalone
LLM cannot produce these because the sequences were computed by this
project, not present in any training data.

---

## Key Findings

RAG significantly improves factual grounding. The standalone LLM 
produces correct general CRISPR guidance but cannot reference specific 
computed guide RNA sequences or ranking scores. In contrast, RAG+LLM 
consistently retrieves and cites exact 20-mer sequences with associated 
scores (e.g., CCTCACTGCAAGCACTGCAT, 90/100), grounded in both computed 
outputs and PubMed literature.

Simple keyword based safety metrics are brittle. The disclaimer metric 
relies on string matching, which fails when the same concept is expressed
 using alternative phrasing. This highlights the limitation of rule-based 
 evaluation for semantic safety properties and motivates the need for 
 LLM-as-judge or embedding-based evaluation in Phase 2.

7B-scale reasoning is grounded but limited in depth. While the model 
correctly incorporates retrieved sequences and scores, it tends to summarize
 rather than deeply compare biological tradeoffs between candidate guides. 
 This reflects the constraints of a 7B parameter model under 4-bit quantization
  on consumer hardware, rather than a flaw in the retrieval or scoring pipeline.
---

## Challenges

### 1. CRISPR is a three-problem domain — scoped to Problem 3 only

Mutation detection (Problem 1) requires clinical patient data and
variant calling infrastructure. Target gene selection (Problem 2)
requires deep biological expertise and literature synthesis. Guide
RNA design (Problem 3) is fully computational given a chosen target
gene. The project UI presents disease-to-gene context for all three
problems, but only Problem 3 runs under the hood. This is how real
bioinformatics tools work — Cas-OFFinder, one of the most cited CRISPR
tools in the world, also solves only Problem 3.

### 2. Guide RNA ranking without ML or genome-scale tooling

Initial guide RNA candidates had no principled ranking. Two
approaches exist in the literature: machine learning models trained
on large experimental datasets, and genome-wide alignment tools like
Bowtie for off-target analysis. Both require either substantial
compute, external validated datasets, or full genome indexing at
around 3GB, making them outside current project scope.

The solution was a rule-based heuristic scoring system derived from
Doench et al. 2016. Each of the five scoring functions defines
explicit numeric ranges with biologically motivated thresholds. GC
content peaks at 55% with continuous scoring that falls off toward
the 40% and 70% boundaries rather than a binary cutoff. Seed region
quality penalises homopolymers and dinucleotide repeats that reduce
specificity. Position scoring rewards guides in the coding region
(10-70% of the sequence) over UTR regions. This approach is not
equivalent to genome-wide alignment but produces biologically
reasonable rankings consistent with published CRISPR design principles,
and is transparent enough to explain criterion-by-criterion. Bowtie
integration against GRCh38 is the primary Phase 2 upgrade.

### 3. Literature retrieval returning only 3 papers

The original approach filtered general medical QA datasets (PubMedQA,
MedAlpaca) for CRISPR keywords, which returned 3 papers total. The
root cause was that CRISPR papers represent under 0.1% of general
medical datasets. The fix was switching to direct PubMed search via
the Entrez API with 10 targeted queries, fetching up to 80 papers per
query and deduplicating by title. This produced 671 unique CRISPR
papers, a 224x increase from the original approach.

### 4. VRAM management during Cell 4 (resolved from prior experience)

Loading the LLM after building the ChromaDB vector store caused CUDA
out-of-memory errors on the RTX 3060. The fix was identical to the
pattern developed in the NeuroQA project: clear stale Chroma cache
with shutil.rmtree before rebuilding, use 4-bit quantization with
bitsandbytes, and set device_map to auto with low_cpu_mem_usage. Prior
experience meant this was diagnosed and resolved without extended
debugging.

### 5. RAG retrieving only PubMed documents

Initial retrieval returned all-PubMed sources for every query, with
zero Cell2-Scorer documents appearing. The cause was that guide RNA
documents were being chunked by the text splitter, separating the gene
name from the actual sequences in different chunks. The retriever found
the header chunk containing the gene name but not the sequence chunk,
so the LLM reported that no sequences were available in context.

The fix was applying chunking only to paper documents and appending
guide RNA documents whole. MMR retrieval was also enabled to penalise
redundant results, preventing the retriever from returning six similar
PubMed chunks rather than mixing paper and guide sources.

### 6. LLM reasoning quality ceiling

After fixing retrieval, the LLM correctly referenced specific guide
RNA sequences and scores but paraphrased rather than reasoning deeply
about biological tradeoffs between candidates. This is the ceiling of
a 7B model in 4-bit quantization on consumer hardware. The responses
are correct and grounded, which is the core contribution. Fine-tuning
on CRISPR-specific Q&A data and upgrading to a larger model via API
are scoped for Phase 2.

### 7. Disclaimer metric inconsistency (partially resolved)

The safety disclaimer metric uses keyword matching which fails when
the model expresses validation requirements in varied phrasing. This
caused DMD, CFTR, and BRCA1 to score zero on the disclaimer metric
despite producing responsible outputs that recommended laboratory
validation in non-standard phrasing. A hybrid prompt improved
consistency across genes but did not fully resolve the keyword
brittleness. Semantic evaluation using an LLM-as-judge approach is
planned for Phase 2.

---

## Limitations

- Off-target scoring is a sequence complexity heuristic, not genome-wide alignment
- RAG quality depends on the coverage and recency of fetched PubMed papers
- System solves Problem 3 only — mutation detection and target selection not implemented
- No wet lab validation integration
- Disclaimer metric uses keyword matching, not semantic detection
- Reasoning depth is limited by 7B model size on consumer hardware

---

## Reproducibility

Gene sequences and papers are frozen to disk on first run. To
reproduce:

```bash
pip install biopython transformers peft bitsandbytes datasets \
            langchain langchain-community langchain-huggingface \
            chromadb sentence-transformers pandas matplotlib
```

Run cells in order: 1, 2, 3, 4, 5.
Restart kernel before Cell 4.
All data fetched from public sources — no restricted access required.

---

## Future Work

**Phase 2:**
- Bowtie-based genome-wide off-target scoring against GRCh38
- Semantic disclaimer evaluation using LLM-as-judge
- Fine-tuning on CRISPR-specific Q&A data
- Reranker layer for improved retrieval quality
- Extend to Problem 1 — mutation detection in patient sequences

**Phase 3:**
- Problem 2 — literature-based target gene selection
- Multi-gene therapy design support
- End-to-end pipeline connecting all three problems
- Clinical validation layer simulation

---

## Impact

This system reduces CRISPR guide RNA selection from 8–10 hours of manual
literature review and sequence analysis to a 5–10 minute computational
pipeline.

It combines:
- deterministic biological scoring (PAM + Doench-inspired rules)
- large-scale literature retrieval (PubMed RAG)
- LLM-based interpretation grounded in computed results

The output is not just ranked sequences, but ranked sequences with
biological rationale and literature-backed explanation.

---

## Project Context

Built as part of a self-directed AI/ML specialization alongside a
Biotechnology undergrad degree. Project 8 in a structured roadmap
combining ML engineering with biomedical domain knowledge. The Bio + AI/ML combination is intentional
 this project sits at the intersection of ML systems and computational biology, 
 requiring both model-driven design and domain biological reasoning.
