# Local LLM Research — Quantization, Vision & Speech on Apple Silicon

**Empirical studies of LLM quantization, vision models, and speech models on a 16 GB MacBook Air (M3)** — August 2026.

> **Headline findings:** LLM speed on consumer hardware is decided by whether the model **fits in RAM** — a 12B model at q8 (13 GB) ran **65× slower** than the same model at q4 (0.17 vs ~11 tok/s) because of swap thrashing, despite being only 1.6× larger. When everything fits, **q4 is the free lunch**: it matched q8's quality (5.0 vs 5.0 across 5 tasks) at 1.6× the speed, while **q2 lost multi-step reasoning entirely without gaining any speed** (18.3 vs 18.1 tok/s — decode overhead cancels the bandwidth win). Across text, vision, and speech, models fail in two modes — **omission** (honest) or **fabrication** (dangerous, because it looks like success).

## Studies

| Folder | Contents |
|---|---|
| [`quantization/`](quantization/) | **Quantization study**: baseline benchmarks (8B/12B/20B incl. a Mixture-of-Experts finding), the RAM-cliff experiment, the q2/q4/q8 quality ladder with full transcripts, graphs |
| [`vision-speech/`](vision-speech/) | **Vision & speech studies**: multimodal chart-reading and OCR (Gemma 3 12B), Gemma-vs-LLaVA comparison with two distinct failure modes, Whisper speech-to-text accuracy and stress tests, plus a commands-and-methods guide |
| [`lab-notes.md`](lab-notes.md) | Dated lab notes — raw measurements and observations behind the studies |

## Setup

- **Hardware:** MacBook Air, Apple M3, 16 GB unified memory
- **Tools:** [Ollama](https://ollama.com) (llama.cpp backend) with `--verbose` timing; Activity Monitor for memory/swap; [mlx-whisper](https://pypi.org/project/mlx-whisper/) for speech
- **Models tested:** llama3.1:8b (q2_K / q4_K_M / q8_0) · gemma3:12b (q4_K_M / q8_0, incl. multimodal vision) · gpt-oss:20b (MoE) · llava:7b · whisper tiny & large-v3-turbo
- **Method:** fixed question sets asked identically to every variant; correctness-only scoring against ground truth; memory recorded mid-generation; fresh session per experiment

## Key results at a glance

**The cliff (fit vs doesn't fit):**
| gemma3:12b | Size | tok/s | Swap | Pressure |
|---|---|---|---|---|
| q4_K_M | ~8.1 GB | ~11 | 3.6 GB | yellow |
| q8_0 | ~13 GB | **0.17** | 3.4 GB + heavy compression | **RED (thrashing)** |

**The quality ladder (llama3.1:8b — all variants fit in RAM):**
| Quant | Size | tok/s | Quality (avg/5, 5 tasks) |
|---|---|---|---|
| q2_K | 3.2 GB | 18.3 | **3.0** (multi-step math: 1/5 — 859-token collapse) |
| q4_K_M | 4.9 GB | 18.1 | 5.0 |
| q8_0 | 8.5 GB | 11.2 | 5.0 |

**MoE:** gpt-oss:20b (20B total, ~3.6B active/token) ran **faster** than dense gemma3:12b (~16 vs ~11 tok/s) — speed follows *active* parameters, memory follows *total* parameters.

**Vision (gemma3:12b):** ~7.2 s flat encoding cost per image; generation unchanged (~11 tok/s); reads charts with printed values perfectly; OCR ~98% on clean print (but injects page headers into content), unreliable on dense/low-res text. **LLaVA-7B:** 2× faster, 7× lighter, but *fabricates* text content it cannot read.

**Speech (whisper-large-v3-turbo):** ~99% on clean read English (incl. accented speech); noise → omission; Hinglish code-switching → fabrication + hallucination loops.

## Author
Krishnagopal Tyagi ([kgt392](https://github.com/kgt392)) — final-year CS student · ML/AI-infrastructure track
