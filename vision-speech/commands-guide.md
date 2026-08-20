# Commands & Methods Guide — Vision & Speech Models on Apple Silicon

**Krishnagopal Tyagi · August 2026 · MacBook Air M3, 16 GB**
Companion to `vision-speech-study.md` (the formal study). This document = every command used, what each part means, what was downloaded, experiment logs, results, and findings — so everything is reproducible from scratch.

---

## 0. Scope

The quantization study measured **text** models. This study gave the same laptop **eyes** (vision: Gemma 3 multimodal, LLaVA) and **ears** (speech: Whisper), and measured where they succeed and how they fail.

**Models used:**
| Model | Size | Tool | Purpose |
|---|---|---|---|
| gemma3:12b | 8.1 GB (already had) | Ollama | multimodal vision tests |
| llava:7b | ~4.7 GB | Ollama | vision comparison |
| whisper-tiny | 74 MB | mlx-whisper (auto) | speech, default |
| whisper-large-v3-turbo | 1.6 GB | mlx-whisper | speech, real tests |

---

## 1. Setup commands, explained line by line

### 1.1 Python virtual environment (needed for mlx-whisper)

```bash
python3 -m venv ~/ml-env
```
- `python3 -m venv` → run Python's built-in **v**irtual **env**ironment maker.
- `~/ml-env` → create it in a folder called `ml-env` in your home directory.
- **Why:** macOS blocks `pip install` into the system Python ("externally-managed-environment" error). A venv is an isolated sandbox where pip works freely and can't break the OS. Every Python project gets its own venv — standard practice.

```bash
source ~/ml-env/bin/activate
```
- `source` → run a script **in the current shell** (so it can change your environment).
- `activate` → switches your shell to use the venv's Python and pip.
- **Success check:** the prompt shows `(ml-env)` at the start.
- ⚠️ Must be re-run in **every new terminal window** before using `mlx_whisper`.

```bash
pip install mlx-whisper
```
- Installs the **mlx-whisper** package (OpenAI's Whisper speech-to-text, rewritten on Apple's **MLX** framework so it runs fast on Apple Silicon GPUs) into the active venv.

```bash
brew install ffmpeg
```
- **ffmpeg** = the universal audio/video decoder. Whisper doesn't read .ogg/.mp3/.m4a itself — it calls ffmpeg to convert any audio into raw 16 kHz samples. Without it: `FileNotFoundError: 'ffmpeg'`.

### 1.2 Vision commands (Ollama)

```bash
ollama run gemma3:12b --verbose
>>> Describe this image in detail: /Users/me/Desktop/photo.jpg
>>> Extract all the text from this image exactly, word for word: /path/page.jpg
```
- Putting a **file path in the prompt** makes Ollama load that image, run it through the model's vision encoder, and insert it as visual tokens.
- `--verbose` → print timing stats after each answer (`prompt eval` = reading cost incl. image encoding; `eval rate` = generation tokens/sec).
- **Rule learned:** fresh session (`/bye`, rerun) per image — long sessions grow the KV cache, spill into swap, and slow prompt-eval 6×; with LLaVA, earlier images even contaminate later answers.

### 1.3 Speech commands (mlx-whisper) — full anatomy

```bash
mlx_whisper ~/Downloads/gre.ogg --model mlx-community/whisper-large-v3-turbo --language hi
```

| Part | Meaning |
|---|---|
| `mlx_whisper` | the transcription program (installed by pip into the venv) |
| `~/Downloads/gre.ogg` | the audio file. `~` = `/Users/<you>` — don't write both (`~//Users/...` = broken path). **Paths with spaces must be quoted:** `"…/WhatsApp Ptt 2026-08-19 at 11.20.58 PM.ogg"` (or drag the file from Finder into the terminal — it auto-escapes) |
| `--model mlx-community/whisper-large-v3-turbo` | which Whisper variant to use, named as a Hugging Face repo (`org/model-name`). Downloaded automatically on first use, cached after |
| `--language hi` | force the language (hi = Hindi) instead of auto-detect. Useful when auto-detect guesses wrong (our Hinglish test detected "English" and fabricated) |
| (omitted) | with no `--model`, it defaults to **whisper-tiny** |

**Run at tiny (default, fast/weak) vs turbo (accurate):**
```bash
mlx_whisper ~/Downloads/book.ogg                                          # tiny, 74 MB
mlx_whisper ~/Downloads/book.ogg --model mlx-community/whisper-large-v3-turbo   # turbo, 1.6 GB
```

**What does "turbo" mean?** Whisper comes in sizes: tiny (39M params) → base → small → medium → large-v3 (1.55B). **large-v3-turbo** is a *speed-optimized* variant of large-v3: same encoder ("ears"), but the decoder ("writer") is pruned from 32 layers to 4 — making it ~6–8× faster than large-v3 with only slightly lower accuracy. Same idea as quantization: trade a little quality for a lot of speed — except here it's done by removing layers (pruning/distillation) instead of shrinking numbers. Best accuracy-per-second choice on a laptop.

---

## 2. Experiment logs

### Experiment 1 — Vision: chart reading & scenes (gemma3:12b)
- **Ran:** 5 images (my own cliff chart, tiger photo, VS Code screenshot, quote photo, stylized wolf art) + follow-up questions.
- **Data:** every new image cost a flat **~7.4–7.7 s prompt-eval** (the vision encoder); generation stayed **~10–11 tok/s** (same as text-only). Memory: 15.2/16 GB, swap 3.9 GB (Docker's VM was hogging 6.9 GB — quit it next time). Two late-session prompt-evals blew up to 45–58 s → KV cache spilling into swap.
- **Results:** read my chart's exact values (11 / 0.17 tok/s) and drew the right conclusion; garbled dense small text ("Gemma 3, LLaVA" → "Gamma 8, LLaid"); scenes/art excellent.
- **Finding:** *"Looking" is a fixed cost; generation is unchanged; structure survives, precision breaks.*

### Experiment 2 — Vision: OCR ("extract text exactly")
- **Ran:** book PDF page, 2 low-res novel pages, handwritten quote, handwritten math notes — fresh session each.
- **Results:** printed book page **~98%** but injected the page header into the middle of a code block (extracted code wouldn't run); low-res dense pages → fluent garbling; big handwriting flawless; handwritten math notes ~90% incl. symbols (□ ≅ △ ∠ CPCTC).
- **Finding / EduRAG decision:** *use PyMuPDF's real text layer for digital PDFs; vision-OCR only as a verified fallback for scans — never trusted for exact citations.*

### Experiment 3 — Vision showdown: gemma3:12b vs llava:7b
- **Ran:** same 6 images through llava:7b.
- **Results:** LLaVA = 2× faster (~20 tok/s), 7× lighter (1.5 GB, pressure green), fine on scenes — but on ANY text it **fabricated**: my chart became "The CGI Model 12" home display with rooms and curtains; the book page became an invented webpage ("encoders and decodators").
- **Bonus discovery:** ran 3 images in one session (prompt tokens 590→1377→2145) → earlier images' hallucinated content **bled into later answers** (context contamination).
- **Finding:** *two failure modes — Gemma degrades, LLaVA fabricates; fabrication is the dangerous one. Encoder generation matters more than parameter count for reading text.*

### Experiment 4 — Speech: Whisper accuracy
- **Setup battles (all solved, all educational):** `pip` blocked → venv; `ffmpeg` missing → brew; `~//Users/...` path doubling; file was in Downloads not Desktop.
- **Ran:** clean read passage (book.ogg) through large-v3-turbo.
- **Result:** **~99% accurate** — only the first 2 words clipped (recording start), punctuation correct.
- **Finding:** *clean English speech-to-text is effectively solved, offline, on a laptop.*

### Experiment 5 — Speech: stress tests
- **Ran:** multi-speaker group chatter; a Hinglish WhatsApp voice note (low-bitrate opus).
- **Results:** chatter → locked onto the dominant voice, **omitted** crosstalk, skipped overlaps. Hinglish → auto-detected "English", **fabricated** fluent nonsense ("Please pick an English Driller"), looped a filler phrase twice (classic Whisper hallucination signature), silently skipped ~18 s.
- **Finding:** *Whisper's noise failure = omission (honest); its code-switching failure = fabrication (dangerous). `--language hi` forcing is the first mitigation to try.*

---

## 3. Headline findings

1. **Vision cost model (M3, 16 GB):** ~7.2 s per image to encode, generation speed unchanged, 12B-vision pushes RAM to the edge — close heavy apps first.
2. **OCR verdict:** ~98% on clean print (with layout-contamination hazards), breaks on dense/low-res, surprisingly good on structured handwriting → EduRAG uses the PDF text layer first.
3. **Model choice is task-fit, not size or speed:** a 2×-faster 7B that fabricates is worthless for text tasks (same law as q2 in Week 1: efficiency below the quality floor buys nothing).
4. **The cross-modality law (both studies):** every model family fails in one of two modes — **omission** (drops what it can't handle: Whisper in noise) or **fabrication** (invents fluent falsehoods: q2 reasoning, LLaVA on text, Whisper on Hinglish). **Fabrication is always the dangerous mode because it looks like success. The only defense is evaluation against ground truth.**
5. **Methodology rules earned the hard way:** fresh session per experiment; quote paths with spaces; venv per project; record prompt-eval and eval-rate separately — they measure different things.

---

## 4. Repo layout
```
llm-lab-notebook/
├── README.md
├── lab-notes.md
├── quantization/        (study + transcripts + graphs)
└── vision-speech/       (study + this guide + whisper findings)
```
