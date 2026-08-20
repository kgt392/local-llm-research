# Local Speech-to-Text on Apple Silicon: Whisper Accuracy & Failure Modes

**Week 2, Day 4 — Speech** · Krishnagopal Tyagi · 18–19 Aug 2026
Hardware: MacBook Air M3, 16 GB · Runtime: **mlx-whisper** (Apple MLX), model **whisper-large-v3-turbo** (~1.6 GB)
Setup notes: required `brew install ffmpeg` (audio decoder) and a Python venv (`python3 -m venv ~/ml-env`) — macOS blocks system-wide pip installs (PEP 668).

> **TL;DR:** On clean speech, local Whisper (large-v3-turbo) is ~99% accurate — including on my Indian-English accent, which scored 5/5 on hard GRE vocabulary. Its errors are not mishearings but **hallucinations at audio boundaries**: clipped openings, invented filler words, and a repetition loop on trailing silence. Same law as Weeks 1–2: models fail *fluently*, at the edges, without warning.

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
2. **The accent tax on prepared speech is ~zero** (5/5 GRE words). The realistic risks left are spontaneous/fast speech, noise, and code-switching — tested next (Day 5).
3. **Whisper's failure mode is hallucination at boundaries, not mishearing in the middle:** clipped starts, invented fillers ("Sure."), and repetition loops on trailing silence. Practical rule: **trim silence from clip edges, and post-filter repeated lines** before trusting a transcript.
4. **Cross-study pattern (Weeks 1–2):** quantized LLMs (q2) lose reasoning fluently; vision models garble or fabricate text fluently; Whisper hallucinates boundaries fluently. **Every local model so far fails *confidently* — output quality must be measured, never assumed.**

## Method notes
- All runs via `mlx_whisper <file> --model mlx-community/whisper-large-v3-turbo`.
- Default model is whisper-tiny (74 MB) — deliberately upgraded to turbo for accuracy; tiny-vs-turbo ladder left as future comparison.
- Transcription speed felt near-real-time or faster; not systematically timed (gap to fix in Day 5 runs).

## Pending (Day 5)
Noisy-background recording · Hindi-English code-switching (Hinglish) · timing measurements → then results merge into `week2-vision-study.md` and push to GitHub.
