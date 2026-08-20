# LLM Quantization on Consumer Hardware: Speed, Memory, and Quality Trade-offs

**An empirical study on a 16 GB MacBook Air (M3)**
Krishnagopal Tyagi · August 2026

> **TL;DR:** On a 16 GB machine, quantization level decides three things: whether a model *fits* (and missing RAM costs a 65× slowdown, not a proportional one), how *fast* it runs (q2 buys nothing over q4), and whether it can still *reason* (q2 loses multi-step arithmetic while remaining fluent). **q4 is the sweet spot: q8-level quality at 1.6× the speed.**

---

## 1. Setup

| | |
|---|---|
| Hardware | MacBook Air, Apple M3, 16 GB unified memory, macOS |
| Runtime | Ollama (llama.cpp backend), `--verbose` for timing stats |
| Measurement | `eval rate` (tokens/sec), Activity Monitor (Memory Used, Wired, Swap, Compressed, Memory Pressure) |
| Models | gemma3:12b (q4_K_M, q8_0) · llama3.1:8b-instruct (q2_K, q4_K_M, q8_0) · gpt-oss:20b (MoE) · llama3.1:8b baseline |
| Quality test | 5 fixed questions asked identically to each variant, scored 1–5 on correctness: factual recall, multi-step arithmetic, Python coding, constraint-following, summarization |

**Terminology:** `q<bits>_<scheme>` — the number is bits per weight (original weights are 16-bit); `_0` is the simple per-block-scale scheme; `_K` is the newer K-quant scheme (super-blocks with smarter bit allocation). Quantization maps each block of float weights onto b-bit integers via a per-block scale (`q = round(w/scale)`, reconstructed as `w' = q·scale`), so error per weight is bounded by `scale/2` — fewer bits ⇒ fewer representable levels ⇒ larger rounding error, accumulated across billions of weights and 30+ layers.

---

## 2. Baseline: model size vs speed on 16 GB

| Model | Type | Size (q4) | eval rate | Memory Used | Swap | Note |
|---|---|---|---|---|---|---|
| llama3.1:8b | dense | ~4.9 GB | ~18 tok/s | 7.7 GB | 1.1 GB | comfortable |
| gemma3:12b | dense | ~8.1 GB | ~11 tok/s | 15.1 GB | 3.6 GB | pressured |
| gpt-oss:20b | **MoE** | ~13 GB | ~16 tok/s | 15.3 GB | ~3 GB | at the edge — but *faster than the 12B* (see §6) |

Speed scales with how many bytes must be read per token, and degrades as swap grows.

---

## 3. Part 1 — The Cliff (when the model does not fit)

**Experiment:** gemma3:12b at q8_0 (13 GB weights) on 16 GB RAM.

| Variant | Size | eval rate | Memory Used | Wired | Swap | Compressed | Pressure |
|---|---|---|---|---|---|---|---|
| q4_K_M | ~8.1 GB | **~11 tok/s** | 15.1 GB | 10.3 GB | 3.6 GB | – | yellow |
| q8_0 | ~13 GB | **0.17 tok/s** | 15.4–15.6 GB | 10.3–10.5 GB | 3.25–3.4 GB | 3.0–4.6 GB | **RED** |

One answer (1,067 tokens) took **~1h44m**. During generation, `llama-server` CPU sat at ~33% — the machine was *waiting on the SSD*, not computing. The 13 GB working set exceeds physical RAM, so for every token several GB of weights must be paged in from swap; the system thrashes.

> **The Cliff Law: speed is not proportional to model size. A model 1.6× larger ran ~65× slower. Fits in RAM → fast; doesn't fit → cliff. The marginal GB beyond RAM costs ~100×, because SSD is ~100× slower than RAM and every weight is read for every token.**

Implication: below the RAM ceiling, quantization is an *optimization*; at the ceiling, it is the difference between usable and unusable. (The run was stopped after one unambiguous answer.)

---

## 4. Part 2 — The Quality Ladder (when everything fits)

To isolate quality effects from fit effects, the ladder uses llama3.1:8b-instruct, whose q2/q4/q8 variants **all fit in RAM** (swap stayed ≤ ~0.4 GB, memory pressure green throughout).

**Results (5 questions, scored 1–5 on correctness):**

| Quant | Size | tok/s | Q1 fact | Q2 math | Q3 code | Q4 constraints | Q5 summary | **Avg** |
|---|---|---|---|---|---|---|---|---|
| q2_K | ~3.2 GB | 18.3 | 3 | **1** | 2 | 4.5 | 4.5 | **3.0** |
| q4_K_M | ~4.9 GB | 18.1 | 5 | 5 | 5 | 5 | 5 | **5.0** |
| q8_0 | ~8.5 GB | 11.2 | 5 | 5 | 5 | 5 | 5 | **5.0** |

**The q2 math specimen (key evidence).** Asked a 2-step profit problem (answer: ₹120), q2_K correctly computed cost = 480 and revenue = 600 — then, one subtraction from the answer, spiraled into invented "mark-up" logic, contradicted itself, and looped for **859 tokens without ever producing an answer**. The same model at q4 and q8 solved it cleanly. q2's code answer likewise *looked* correct but contained a wrong worked example (claims `[5,10,15]` → 15; correct is 10) and a broken third variant.

**Failure ordering.** Quantization damage is not uniform across abilities:
- **Breaks first:** multi-step reasoning and code correctness (errors compound across steps).
- **Survives longest:** fluency, summarization, and (surprisingly) instruction constraints — q2 passed "exactly 3 bullets, <12 words, no banned word."

The dangerous consequence: **a heavily quantized model doesn't sound dumb — it sounds confident while being wrong.** Quality loss is invisible unless measured.

---

## 5. Findings

1. **The Cliff Law.** Fit in RAM is binary: 1.6× more bytes → 65× less speed the moment the working set exceeds physical memory (§3).
2. **q2 is the worst of both worlds.** It ran *no faster* than q4 (18.3 vs 18.1 tok/s) — the aggressive packing costs extra decode compute per weight, cancelling the bandwidth savings — while losing 2 of 5 quality points. Below q4 there is no speed reward, only quality loss.
3. **q4 is the free lunch.** q8 matched q4's quality on every task (5.0 vs 5.0) while running 38% slower (11.2 vs 18.1 tok/s) at 1.7× the size. Speed above q4 scales with bytes/token, as expected for a memory-bandwidth-bound workload (~5 GB → 18 tok/s, ~8.5 GB → 11 tok/s).
4. **Practical rule for 16 GB-class machines:** default to q4_K_M; use q8 only if RAM is abundant *and* the task is precision-critical; never q2 for anything requiring reasoning; and check the RAM ceiling before anything else, because fit dominates every other consideration.

*Caveat: quality was scored on 5 tasks by a single rater — sufficient to expose the q2 collapse and the q4≈q8 equivalence on everyday tasks, but standard benchmarks (MMLU, GSM8K) report small residual gaps between q4 and q8 that this sample cannot resolve.*

---

## 6. Bonus finding — MoE: total vs active parameters

gpt-oss:20b (a Mixture-of-Experts model: ~20B total parameters, ~3.6B active per token) ran at **~16 tok/s — faster than the dense 12B (~11 tok/s)** despite being larger on disk and in RAM.

> **Speed follows *active* parameters (compute per token); memory follows *total* parameters (all experts must be loaded).** MoE deliberately separates the two — which also means quantization matters *more* for MoE models, since their memory footprint is set by the total count while their speed advantage survives only as long as they fit.

---

## 7. Method notes (glitches, disclosed)

- The q2_K arithmetic prompt accidentally included an extra hint line ("Watch for: arithmetic slips…"), so prompts were not 100% identical for that one cell; the collapse occurred regardless.
- One mid-session entry (a shell command typed into the chat) was excluded from scoring.
- The 12B q8 cliff run was stopped after one question; the result was unambiguous and further runs risked system instability (an earlier attempt with gpt-oss:20b forced a shutdown).

## Repository contents

```
quantization/
├── quantization-study.md      <- this report
├── graphs/                    <- tok/s vs quant · quality vs quant · the cliff
└── answers/                   <- full transcripts + scoring notes per variant
```
