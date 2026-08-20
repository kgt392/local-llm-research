# Local Speech-to-Text on Apple Silicon: Whisper Accuracy & Failure Modes

Krishnagopal Tyagi · 18–19 Aug 2026
Hardware: MacBook Air M3, 16 GB · Runtime: **mlx-whisper** (Apple MLX), model **whisper-large-v3-turbo** (~1.6 GB)
Setup notes: required `brew install ffmpeg` (audio decoder) and a Python venv (`python3 -m venv ~/ml-env`) — macOS blocks system-wide pip installs (PEP 668).

> **TL;DR:** On clean speech, local Whisper (large-v3-turbo) is ~99% accurate — including on my Indian-English accent, which scored 5/5 on hard GRE vocabulary. Its errors are not mishearings but **hallucinations at audio boundaries**: clipped openings, invented filler words, and a repetition loop on trailing silence. Same law as the quantization and vision studies: models fail *fluently*, at the edges, without warning.

---

## Tests & Results

### Test 1 — Clean read English (book.ogg, slow clear passage)
- **Result: ~99% word-for-word**, punctuation correct.
- Only error: the opening two words ("I read…") missing — audio-start clipping, not a recognition error.

### Test 2 — My voice, GRE vocabulary (gre.ogg) — the "accent tax" test
Spoken: *"His pragmatic approach helped him ameliorate the problem despite the arduous circumstances. His perspicacious analysis exposed the fallacious assumptions behind the argument."*
- **Result: 5/5 hard words recognized and correctly spelled** — pragmatic, ameliorate, arduous, perspicacious, fallacious.
- **Accent tax on prepared Indian-English speech ≈ 0.**
- Artifact: a spurious **"Sure."** inserted at 0:00 — a known Whisper behavior of hallucinating filler words from noise at clip boundaries.
- Side use discovered: this doubles as **pronunciation feedback** — any word Whisper misses is a word worth practicing aloud (useful for GRE/TOEFL/interviews).

### Test 3 — Native narration, archaic formal prose (nat.mp3, LibriVox ~2.5 min)
- **Result: essentially perfect** on hard content — 19th-century prose, proper nouns ("Alexandra Feodorovna, Empress of all the Russias"), long formal sentences. Minor: one spelling wobble on the author's name (rainesford/rainsford); output lowercase without punctuation for this file.
- **⭐ Key specimen — the repetition loop:** after the speech ends (trailing silence/outro), the model emitted **"end of preface" ~15 times in a row** with collapsing timestamps. This is Whisper's documented repetition-loop hallucination on non-speech audio: it echoes its last phrase instead of stopping.

---

## Findings

1. **Local speech-to-text is a solved problem for clean speech.** ~99% accuracy on a laptop, offline, free — including accented English and archaic prose.
2. **The accent tax on prepared speech is ~zero** (5/5 GRE words). The realistic risks left are spontaneous/fast speech, noise, and code-switching — covered in the stress tests below.
3. **Whisper's failure mode is hallucination at boundaries, not mishearing in the middle:** clipped starts, invented fillers ("Sure."), and repetition loops on trailing silence. Practical rule: **trim silence from clip edges, and post-filter repeated lines** before trusting a transcript.
4. **Cross-study pattern:** quantized LLMs (q2) lose reasoning fluently; vision models garble or fabricate text fluently; Whisper hallucinates boundaries fluently. **Every local model so far fails *confidently* — output quality must be measured, never assumed.**

## Method notes
- All runs via `mlx_whisper <file> --model mlx-community/whisper-large-v3-turbo`.
- Default model is whisper-tiny (74 MB) — deliberately upgraded to turbo for accuracy; tiny-vs-turbo ladder left as future comparison.
- Transcription speed felt near-real-time or faster; not systematically timed (acknowledged gap).

## Stress tests (19 Aug)

### Test 4 — Multi-speaker background chatter
- Locked onto the **dominant voice** and transcribed it coherently ("…Thanksgiving, and then I went up to dinner the day after…") while **omitting** the overlapping crosstalk entirely, skipping stretches of overlap.
- Failure mode: **omission** — honest degradation; it drops what it can't parse.

### Test 5 — Hindi-English code-switching (Hinglish WhatsApp voice note, low-bitrate opus)
- Language auto-detect chose "English" for mixed speech.
- Output was **fabricated fluent nonsense** ("Your new monitor is going to be Sir the Live." / "Please pick an English Driller."), a repetition loop ("I'm a little bit more." ×2), and ~18 s silently skipped.
- Failure mode: **fabrication** — dishonest degradation; it invents rather than admitting failure. First mitigation to try: force the language (`--language hi`).
- Note: WhatsApp's aggressive audio compression is itself a variable here.

### Stress-test conclusion
Whisper's two failure modes mirror the vision study exactly: **noise → omission (honest), code-switching/unclear audio → fabrication (dangerous)**. Fabrication is always the riskier mode because the output *looks* like a successful transcript. Full context in `vision-speech-study.md`, Experiment 4.
