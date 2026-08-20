# Lab Notes — dated measurements and observations

Hardware for all entries: MacBook Air M3, 16 GB unified memory · Ollama `--verbose`.

---

## Quantization experiments (Aug 9–14)

**Problem:** run a big model at usable speed on 16 GB — keep the model out of swap.

### Baseline (3 models, 5 fixed questions each)
| Model | Type | Total / Active params | tok/s (eval rate) | Memory Used | Wired | Swap |
|---|---|---|---|---|---|---|
| llama3.1:8b (q4) | dense | 8B / 8B | **~18** (17.93 / 17.97 / 18.32) | 7.73 GB | 1.73 GB | 1.14 GB |
| gemma3:12b (q4) | dense | 12B / 12B | **~11** (10.61 / 10.89 / 11.00) | 15.09 GB | 10.29 GB | 3.61 GB |
| gpt-oss:20b | **MoE** | 20B / ~3.6B | **~16** (15.56 / 16.46 / 15.96) | 15.3 GB | 11.3 GB | ~3 GB, CPU 58–87% |

- Memory anatomy learned: **Memory Used** (everything active), **Wired** (locked in RAM — grows under model pressure), **Cached Files** (where mmap'd model weights live), **Compressed**, **Swap** (disk pretending to be RAM, ~100× slower — the speed killer).
- **MoE finding:** the "bigger" 20B beat the 12B because only ~3.6B params fire per token. Speed ∝ active params; memory ∝ total params.

### The RAM-cliff experiment (Aug 10)
gemma3:12b-**q8_0** (13 GB) on 16 GB: **0.17 tok/s** — 1,067 tokens took ~1h44m. Swap 3.4 GB, Memory Used 15.56 GB, Wired 10.52 GB, compressed ~3–4.6 GB, memory-pressure RED, CPU only ~33% (waiting on SSD). One question was enough; run stopped.
→ **Cliff law: fits in RAM = fast; doesn't fit = ~100× penalty per marginal GB.** q8 is 1.6× bigger than q4 but ran ~65× slower.

### The quality-ladder experiment (Aug 12–13) — llama3.1:8b, all fit in RAM (swap ≤0.4 GB, pressure green)
| Q (task) | q2_K | q4_K_M | q8_0 |
|---|---|---|---|
| Seasons (fact) | 3 | 5 | 5 |
| Pens profit (math) | **1** | 5 | 5 |
| 2nd-largest (code) | 2 | 5 | 5 |
| 3 bullets (constraints) | 4.5 | 5 | 5 |
| Summary | 4.5 | 5 | 5 |
| **Avg** | **3.0** | **5.0** | **5.0** |
| tok/s | 18.3 | 18.1 | 11.2 |

- q2's math answer: computed CP=480 and revenue=600 correctly, then spiraled into invented "mark-up" logic for **859 tokens and never produced the answer** (₹120). Fluent but reasoning collapsed.
- **Findings:** (1) reasoning collapses first, fluency survives — quantization makes models *wrong while sounding right*; (2) q4 ≈ q8 in quality → q4 is the free lunch; (3) q2 is NOT faster than q4 — aggressive packing costs extra decode compute — so below q4 there's no speed reward, only quality loss.

---

## Vision & speech experiments (Aug 15–19)

### Gemma 3 12B vision — chart reading & scenes (Aug 15) (multimodal: SigLIP vision encoder + LM in one file)
- Image → ~290 visual tokens → transformer treats them like text tokens.
- **Every image costs a flat ~7.4–7.7 s prompt-eval; generation unchanged at ~10–11 tok/s.**
- Read my own cliff chart perfectly (11 and 0.17 values + correct conclusion). Dense small screen text got confidently garbled ("Gemma 3, LLaVA" → "Gamma 8, LLaid"). Scenes/art excellent.
- Late-session follow-ups hit 45–58 s prompt-evals: KV cache + full RAM → swap. **Rule: fresh session per experiment.** (llama-server 10.3 GB, Memory Used 15.2/16, swap 3.9 GB — Docker VM was eating 6.9 GB concurrently.)

### OCR test (Aug 16) ("extract all the text exactly")
| Source | prompt eval | eval rate | Accuracy |
|---|---|---|---|
| Book PDF page | 7.16 s | 11.4 | ~98% — but injected page header mid-code-block (broke the import line) |
| Novel page 1 (low-res) | 7.19 s | 11.1 | poor — fluent garbling |
| Novel page 2 (low-res) | 7.22 s | 11.4 | poor — merged/invented fragments |
| Handwritten quote | 7.32 s | 11.0 | ~100% |
| Handwritten math notes | 7.29 s | 11.1 | ~90% incl. symbols (□ ≅ △ ∠ CPCTC) |

→ **EduRAG decision:** PyMuPDF text layer first; vision-OCR only as verified fallback; never trust it for exact citations.

### gemma3:12b vs llava:7b — same 6 images (Aug 16)
- LLaVA: **2× faster (~20 tok/s, 4–4.5 s/image), 7× lighter (1.5 GB, swap <1 GB)**, adequate on scenes — but on any text-in-image it **fabricates confidently** (my chart → invented smart-home display "The CGI Model 12" with Villain/Room/Kitchen rows; book page → invented split-screen webpage, "encoders and decodators").
- Also found **context contamination**: 3 images in one session (prompt tokens 590→1377→2145) — earlier images' hallucinated content bled into later answers.
- **Findings:** Gemma *degrades*, LLaVA *fabricates*; vision model choice is task-dependent; text tasks need the newer/bigger encoder; efficiency below the quality floor is worthless.

### Whisper accuracy (mlx-whisper, venv + ffmpeg) (Aug 18)
- Default model = tiny (74 MB); real tests on **large-v3-turbo** (1.6 GB — large-v3's encoder with the decoder pruned 32→4 layers: ~6–8× faster, slight accuracy cost).
- Clean read passage: **~99%** (only first 2 words clipped — recording start).
- GRE-vocabulary sentences in my own (Indian-English) voice: transcribed correctly — pragmatic, ameliorate, arduous, perspicacious, fallacious all recognized.
- Setup lessons: pip needs a venv on macOS; ffmpeg is the decoder underneath; `~` already means /Users/me; quote paths with spaces.

### Whisper stress tests (Aug 19)
- Group chatter: locked onto dominant speaker, **omitted** crosstalk.
- Hinglish WhatsApp note (low-bitrate): detected "English", **fabricated** fluent nonsense ("Please pick an English Driller"), repetition loop ("I'm a little bit more." ×2), skipped ~18 s.
- **Cross-modality law (the strongest overall lesson):** every model family fails by **omission** (honest) or **fabrication** (dangerous — looks like success). Seen in q2 reasoning, LLaVA text, Whisper code-switching. The only defense is evaluation against ground truth.
