# Local Multimodal Vision Models on Apple Silicon: Chart Reading, OCR, and Failure Modes

Krishnagopal Tyagi · August 2026
Hardware: MacBook Air M3, 16 GB unified memory · Runtime: Ollama (`--verbose`)
Models: **gemma3:12b** (multimodal, ~8.1 GB q4) · **llava:7b** (~4.7 GB)

> **TL;DR:** Gemma3-12B reads charts and printed text impressively (~98% on a clean book page) but garbles dense/small text while staying fluent. LLaVA-7B is 2× faster and 7× lighter but **cannot read text — it fabricates plausible content instead**, a strictly worse failure mode. Vision costs a flat ~7.2 s/image to encode on this hardware; generation speed is unchanged by vision. Verdict for EduRAG: use the PDF's real text layer (PyMuPDF) as primary ingestion; vision-OCR only as a verified fallback for scans.

---

## Background: how one model gained sight

gemma3:12b is multimodal: the downloaded file contains a **vision encoder (SigLIP)** bolted onto the language model. The encoder converts an image into a few hundred *visual tokens* — the same kind of embedding vectors text becomes — which are inserted into the prompt. The transformer then attends over them like any other tokens. Consequences observed in the data below: (a) each image adds a fixed encoding cost at prompt-eval time; (b) generation speed is identical to text-only use. llava:7b is the same idea on an older/smaller base (LLaVA = Large Language and Vision Assistant).

---

## Experiment 1 — Chart reading & scene understanding (gemma3:12b)

### Metrics
| # | Test | prompt tokens | prompt eval | eval rate |
|---|---|---|---|---|
| 1 | Cliff chart — describe | 286 | **7.68 s** | 11.18 |
| 2 | Chart — exact values (follow-up) | 830 | 6.57 s | 10.97 |
| 3 | Chart — conclusion (follow-up) | 1030 | 2.96 s | 10.59 |
| 4 | Tiger photo — describe | 1645 | **7.50 s** | 10.71 |
| 5 | Tiger — interpretation (follow-up) | 2047 | 0.80 s | 10.71 |
| 6 | VS Code screenshot — describe | 2604 | **7.52 s** | 10.17 |
| 7 | follow-up (late session) | 3135 | **45.2 s** ⚠️ | 10.57 |
| 8 | Quote photo — describe | 3580 | **7.70 s** | 10.31 |
| 9 | follow-up (late session) | 3972 | **57.8 s** ⚠️ | 10.70 |
| 10 | Wolf art — describe (fresh session) | 289 | 7.42 s | 10.66 |
| 11 | Wolf — identify animal | 801 | 13.2 s | 9.78 |

Memory mid-session: llama-server ~10.3 GB · Memory Used 15.2/16 GB · Wired ~11 GB · Swap ~3.9 GB · pressure yellow. (Docker Desktop's VM held 6.9 GB concurrently — inflated the swap; quit it in later sessions.)

### Results
- **Chart (my own quantization "cliff" graph):** read the table AND the bars, reported both exact values (11 and 0.17 tok/s), and drew the correct conclusion — even inferred unprompted that q4/q8 are quantization levels. Vision models are strong when values are *printed* in the image. (Miss: interpreted the title "The Cliff" as a project name.)
- **Dense small text (VS Code screenshot):** structure and gist correct, exact strings corrupted — "Gemma 3, LLaVA" became **"Gamma 8, LLaid"**; "PyMuPDF" → "PyPDF"; invented date "2024/08/14". Confident, fluent, wrong in the details.
- **Scenes/art (tiger photo, stylized wolf print):** excellent — composition, lighting, even the halftone print technique; identified the wolf with reasoned features.
- **Text overlay (quote photo):** large clean lettering read perfectly.

### Findings
1. **"Looking" costs a flat ~7.4–7.7 s per image** (rows 1, 4, 6, 8, 10) regardless of image content — the vision-encoder cost.
2. **Generation is unchanged by vision: ~10–11 tok/s**, identical to text-only 12B — once encoded, image tokens are just tokens.
3. **⚠️ rows 7 & 9 (45–58 s prompt evals):** not image cost — the conversation had grown past ~3,000 tokens with RAM nearly full; the KV cache spilled into swap. **Lesson: fresh session per experiment on memory-tight hardware** (row 10 confirms: fresh session → back to 7.4 s).
4. Failure signature: **structure survives, precision breaks** — the vision counterpart of the q2 quantization finding.

---

## Experiment 2 — OCR test (gemma3:12b): "Extract all the text exactly"

### Metrics (fresh session per image)
| Test | prompt eval | eval rate |
|---|---|---|
| Book PDF page (Hands-On LLM, Ch 3) | 7.16 s | 11.4 |
| Novel page 1 (low-res JPEG) | 7.19 s | 11.1 |
| Novel page 2 (low-res JPEG) | 7.22 s | 11.4 |
| Handwritten quote (large lettering) | 7.32 s | 11.0 |
| Handwritten math notes | 7.29 s | 11.1 |

### Results
| Source | Accuracy | Notes |
|---|---|---|
| **Printed book page (clean screenshot)** | **~98%** | Near-perfect words — but a **structural error**: the running page-header "Hands-On Large Language Models" was injected into the middle of the code block, splitting the import line `from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline` in half. Extracted code would not run. |
| **Dense novel pages (low-res)** | Poor | Fluent garbling: merged/dropped/invented fragments that read like sentences ("I stop and lean against the wall of the staircase nothing."). Never signals uncertainty. |
| **Large stylized handwriting (quote)** | ~100% | Flawless, including attribution. |
| **Handwritten math notes** | ~90% | Got the two-column proof structure, theorem numbering, and symbols (□, ≅, △, ∠, ASA, CPCTC). Minor slips on the reflexive-property line. Better than expected. |

### Findings
1. **Clean printed text: high word accuracy, but layout elements (headers/footers) contaminate content** — dangerous for code extraction and chunking.
2. **Resolution/density is the killer variable:** low-res dense paragraphs → confident hallucination, with no warning.
3. **EduRAG design decision (from own data):** for digital PDFs use **PyMuPDF's embedded text layer** (exact, free, no model); use vision-OCR **only as a fallback for scanned pages, never trusted for exact citations without verification.**

---

## Experiment 3 — Showdown: gemma3:12b vs llava:7b (same images, same prompts)

### Comparison
| Test | Gemma3-12B | LLaVA-7B | Winner |
|---|---|---|---|
| Cliff chart | Exact values, correct conclusion | **Total fabrication:** "The CGI Model 12", grid rows "Villain/Room/Kitchen/Bathroom/Dining", "Curtain/Blinds" options, an "Average" block — none of it exists | Gemma |
| Scene (tiger) | Excellent | Genuinely good | Tie |
| Screen with text | Content read, names garbled | Punted ("blurry… difficult to discern") | Gemma |
| Book page OCR | ~98% word-for-word | Invented a split-screen webpage + article "Characterizing the transformer model", "encoders and **decodators**" | Gemma |
| Quote image | Perfect | Quote roughly right — plus the same hallucinated article again | Gemma |
| Novel page | Real text, some garbling | Pure invention ("A Beginner's Guide to Language Models") | Gemma |
| **Speed** | ~11 tok/s · 7.2 s/image encode | **~20 tok/s · 4.0–4.5 s/image** | LLaVA |
| **Memory** | ~10.3 GB, swap 3.9 GB, pressure yellow | **~1.5 GB process, swap <1 GB, pressure GREEN** | LLaVA |

(Note: LLaVA's encoder produces ~590 prompt tokens/image vs Gemma's ~290 — more tokens, yet faster to encode. Different encoder designs.)

### Findings
1. **Two distinct failure modes.** Gemma **degrades** (right structure, corrupted details); LLaVA **fabricates** (invents plausible content that fits the image's vibe). For OCR/RAG use, fabrication is strictly more dangerous — it produces confident, well-formed, false text.
2. **Context contamination (accidental discovery).** The last three LLaVA tests ran in one session (prompt tokens 590 → 1377 → 2145). Hallucinated content from the book-page turn ("Characterizing the Transformer model") **reappeared inside the answers for the quote and novel images.** Prior images bleed into later answers. Methodology rule confirmed the hard way: **fresh session per image.**
3. **Efficiency below the quality floor is worthless.** LLaVA is 2× faster and 7× lighter — and unusable for any text-in-image task. Same law as q2 in the quantization study. (Honest caveat: for pure scene description, LLaVA-7B at 20 tok/s is a legitimate budget option.)
4. **Encoder generation matters more than parameter count** for reading text: a newer 12B beats an older 7B not by degree but by category.

---

## Cross-study conclusions (together with the quantization study)

1. **A recurring law:** models fail *fluently*. Quantized q2 lost arithmetic while sounding confident; Gemma's OCR garbled names in grammatical sentences; LLaVA invented entire documents. **Quality loss is invisible unless measured against ground truth** — which is why evaluation, not vibes, must decide model choice.
2. **Fixed costs of multimodality on 16 GB:** ~7.2 s/image encode (Gemma), unchanged generation speed, and vision+12B pushes the machine to its memory edge (quit heavy apps first).
3. **Task-fit beats size and speed:** 12B beats 7B at reading; 7B wins scenes-per-second; neither replaces a real OCR engine for exact text.

## Experiment 4 — Speech-to-text: Whisper (mlx-whisper, large-v3-turbo)

Setup: Python venv + `mlx-whisper` + ffmpeg. Default model is tiny (74 MB); tests used **large-v3-turbo** (1.6 GB), which runs fast on the M3.

### Results
| Test | Result |
|---|---|
| Clean read passage (.ogg) | **~99% accurate** — only the first 2 words clipped (recording start, not model error); punctuation correct |
| Group chatter (multi-speaker noise) | Locked onto the dominant voice, transcribed it coherently, **omitted** the crosstalk entirely; also skipped stretches of overlap |
| Hinglish WhatsApp voice note (low-bitrate opus) | Detected "English"; produced **fabricated fluent nonsense** ("Please pick an English Driller"), a repetition loop ("I'm a little bit more." ×2 — Whisper's classic hallucination signature), and silently skipped ~18 s |

### Findings
1. **Clean English speech-to-text is solved on-device** — near-perfect, offline, fast on a laptop.
2. **Whisper has two failure modes, matching the vision experiments' taxonomy:**
   - **Noise → omission:** drops what it can't parse (honest degradation).
   - **Code-switched / unclear audio → fabrication:** invents fluent sentences and loops (dishonest degradation — looks like success).
3. **Cross-modality meta-finding (Weeks 1–2):** every model family shows an *omission* mode and a *fabrication* mode — q2 quantization (reasoning collapse), LLaVA (invented documents), Whisper (invented sentences). **Fabrication is always the dangerous mode**, because output fluency hides the failure. Evaluation against ground truth is the only defense.
4. Practical notes: ffmpeg is the audio decoder underneath; filenames with spaces must be quoted; WhatsApp's heavy audio compression is itself an accuracy variable.
