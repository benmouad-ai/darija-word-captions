---
name: darija-word-captions
description: Word-by-word captions that stay in sync with speech that switches between Moroccan Darija, Arabic, French and English. Transcribes with a speech-to-text model, times every word with local forced alignment (WhisperX + wav2vec2), then renders the captions with HyperFrames or Remotion. Use when someone asks for word-by-word, karaoke-style or synced captions on a Darija or mixed-language talking video.
---

# Darija word-by-word captions

Accurate captions need two models:
1. **Speech-to-text** writes the words.
2. **Forced alignment** gives every word the time it starts and ends.

Then an editing tool styles and renders the captions on the video.

Rules that keep the result accurate:
- Darija and Arabic stay in **Arabic script**. English and French stay in **Latin script**, exactly as spoken. Never translate, never transliterate.
- Timing comes **only** from acoustic alignment. Never spread words evenly across a sentence and never guess times. A word that cannot be aligned is reported, not invented.
- The displayed words never change to make alignment easier. Use pronunciation aliases instead (step 3).

---

## Prerequisites

Install once:

| Need | Install |
|---|---|
| Python 3.10+ | python.org or `brew install python` |
| FFmpeg | `brew install ffmpeg` (macOS) · `sudo apt install ffmpeg` (Linux) |
| WhisperX (runs the alignment models) | `pip install whisperx==3.8.6` |
| Node.js 18+ (for the editing tool) | nodejs.org or `brew install node` |

Alignment models (downloaded automatically on first run, then cached):

| Language | Model |
|---|---|
| Darija / Arabic | `jonatasgrosman/wav2vec2-large-xlsr-53-arabic` (Hugging Face) |
| French | `jonatasgrosman/wav2vec2-large-xlsr-53-french` (Hugging Face) |
| English | `WAV2VEC2_ASR_BASE_960H` (torchaudio, WhisperX's English default) |

Speech-to-text, pick one:

| Option | Model | Access |
|---|---|---|
| Gemini Flash | `gemini-3.1-flash-lite` | Gemini API key (cheap, fast) |
| Gemini 3.5 Transcribe | `gemini-3.5-transcribe` | Gemini API key |
| Microsoft | `MAI-Transcribe-2` | Azure Speech key + region |
| Local, free | `atlasia/moulsot.v0.3` (made for Darija) | Hugging Face; needs a GPU or an Apple Silicon Mac |

Editing tool, pick one (both are agent skills):

| Tool | Install the agent skill |
|---|---|
| HyperFrames (HTML → video) | `npx skills add heygen-com/hyperframes` |
| Remotion (React → video) | `npx skills add remotion-dev/skills` |

API keys go in environment variables (for example `GEMINI_API_KEY`). Never paste a key into a file you share.

---

## Workflow

### 0. Set up the script
The script is `scripts/align_words.py` next to this SKILL.md. If you only received SKILL.md, save the copy at the end of this file as `scripts/align_words.py`, exactly as written.

### 1. Prepare the audio
```bash
ffmpeg -i input.mp4 -vn -ac 1 -ar 16000 audio.wav
```
Optional: trim long pauses first (`silencedetect`), and apply any speed change to the audio **before** transcribing, so every timestamp is on the final clock.

Cut the audio into chunks of 10–30 seconds at pauses (`ffmpeg -af silencedetect=noise=-35dB:d=0.3`). Never cut inside a word. Record each chunk's start and end.

### 2. Transcribe each chunk
Ask the model for a **verbatim** transcript. A cleaned transcript drops fillers and repetitions, then no longer matches the audio, and alignment fails.

Prompt that works with Gemini:
> Transcribe ONLY this audio chunk exactly as spoken. The speaker mixes Moroccan Darija with English and occasional French. Write Darija in Arabic script and English or French in Latin letters. Do NOT transliterate English into Arabic. Keep every audible word, filler and repetition. Return JSON: segments [{start, end, text}] with short 2–5 second clauses, plus uncertain [{start, end, heard, reason}].

Re-transcribe any unclear window on its own, and fix names and brand spellings by hand.

### 3. Write transcript.json
```json
{
  "audio": "audio.wav",
  "provider": "gemini-3.1-flash-lite",
  "latin_default": "en",
  "aliases": {
    "24": "ربعة وعشرين",
    "OpenAI": "أوبن إيآي",
    "captions": "كابشنز"
  },
  "segments": [
    {"start": 0.0, "end": 4.4, "text": "هاد الـ captions كتبع كل كلمة"},
    {"start": 4.4, "end": 9.1, "spans": [
      {"lang": "ar", "text": "وقالو ليك"},
      {"lang": "fr", "text": "c'est pas encore disponible"}
    ]}
  ]
}
```
- `segments`: the chunk windows from step 1 with their text.
- `spans`: use them when one segment mixes French and English, so each run gets the right model.
- `aliases`: how a word **sounds**, for alignment only. The caption still shows the original word. Give every number an alias. In Darija speech, also give English and French words an Arabic-script alias: Darija speakers pronounce them with a Darija accent, and the Arabic model then places them well. The script cross-checks them with the native English or French model and keeps the stronger match.

### 4. Align
```bash
python scripts/align_words.py transcript.json --out captions/
```
Outputs:
- `captions/words.json`: every word with `start`, `end`, `score`, `lang`, `method`
- `captions/issues.json`: words to check (unaligned, overlaps, low confidence)
- `captions/evidence.json`: raw aligner output

If the script asks for aliases, add them and run it again. Fix every entry in `issues.json` before rendering. Wrong transcript text is the usual cause.

### 5. Render the captions
Give `words.json` to HyperFrames or Remotion and build one caption element per word (or per short group). Each one shows from its `start` and hides at the next word's `start`.

Display tips:
- **Mixed-direction text:** put the line in a right-to-left container, and keep consecutive English words in one left-to-right group (`<bdi dir="ltr">even in English</bdi>`). If every word is a separate inline block, the English comes out in reverse order.
- Standalone articles such as `الـ` can merge into the next word's caption, so no caption is only "الـ".
- Keep captions and video on the same clock. If you cut or speed up the video, apply the same change to the timestamps.

### 6. Check before you publish
Watch the opening, the middle, the end and every language switch. Say clearly whether a human has listened to the result. Automated checks do not prove that every word is right.

---

## Setup prompt

Paste this to your agent together with this file:

> I'm giving you a skill file, SKILL.md, for word-by-word captions on videos where I speak Moroccan Darija mixed with English and French.
> 1. Install it: `npx skills add benmouad-ai/darija-word-captions`, or put the folder in `~/.claude/skills/darija-word-captions/` (Claude Code) or your Codex skills folder.
> 2. Make sure `scripts/align_words.py` sits next to SKILL.md. If I only gave you SKILL.md, save the script at the end of it there, exactly as written.
> 3. Check and install the prerequisites: Python, FFmpeg, `pip install whisperx==3.8.6`, Node.js. Tell me what you installed.
> 4. Ask me which speech-to-text option I want (Gemini Flash, Gemini 3.5 Transcribe, Microsoft MAI-Transcribe-2, or local Moulsot) and how to provide the API key safely, as an environment variable.
> 5. Ask me which editing tool I want (HyperFrames or Remotion) and install its agent skill.
> 6. Run the full workflow on a short test clip (`test.mp4`) and show me `issues.json` and a rendered preview before we do a real video.

---

## scripts/align_words.py

```python
#!/usr/bin/env python3
"""Word-level caption timing for code-switched speech.

Built for Moroccan Darija mixed with Modern Standard Arabic, French and English,
but works for any Arabic-script + Latin-script mix.

Input:  transcript.json (format in SKILL.md). Text comes from any speech-to-text
        provider. This script never changes the words, only times them.
Output: words.json   every word with start, end, score, method and language
        issues.json  everything a human should check
        evidence.json raw aligner output for audit

All timing comes from acoustic CTC forced alignment (WhisperX). Nothing is
interpolated, estimated from sentences, or spread evenly. A word the aligner
cannot place stays null and is listed in issues.json.

Usage:
  python align_words.py transcript.json --out captions/
"""
import argparse
import json
import re
import sys
from pathlib import Path

import torch
from whisperx.alignment import align, load_align_model
from whisperx.audio import SAMPLE_RATE, load_audio

# Alignment models. Hugging Face ids are downloaded once and cached.
ALIGN_MODELS = {
    "ar": "jonatasgrosman/wav2vec2-large-xlsr-53-arabic",  # Darija + MSA in Arabic script
    "fr": "jonatasgrosman/wav2vec2-large-xlsr-53-french",
    "en": "WAV2VEC2_ASR_BASE_960H",  # torchaudio bundle, WhisperX English default
}
ARABIC = re.compile(r"[؀-ۿݐ-ݿ]")
LATIN = re.compile(r"[A-Za-zÀ-ÖØ-öø-ÿ]")
TATWEEL = "ـ"

PAD = 0.45           # search margin around a segment window, seconds
LATIN_PAD = 0.18     # margin around a Latin run, bounded by neighbours
MIN_WINDOW = 0.12    # smaller windows cannot hold a word
MAX_BATCH = 30.0     # merge segments into windows up to this length
LOW_SCORE = 0.08     # below this a match is flagged for review
OVERLAP_TOL = 0.03


def script_lang(text, latin_lang):
    if ARABIC.search(text):
        return "ar"
    if LATIN.search(text):
        return latin_lang
    return None


def load_transcript(path):
    data = json.loads(path.read_text(encoding="utf-8"))
    audio = (path.parent / data["audio"]).resolve()
    latin_default = data.get("latin_default", "en")
    aliases = data.get("aliases", {})
    words, missing = [], []
    for s, seg in enumerate(data["segments"]):
        spans = seg.get("spans") or [{"text": seg["text"], "lang": None}]
        for span in spans:
            for token in span["text"].split():
                alias = aliases.get(token)
                # The alias decides which model hears the word: an Arabic-script
                # alias for "OpenAI" sends it through the Arabic pass.
                latin_lang = span.get("lang") if span.get("lang") in ("fr", "en") else latin_default
                lang = script_lang(alias if alias else token, latin_lang)
                if lang is None:
                    missing.append(token)
                    continue
                words.append({
                    "i": len(words), "word": token, "lang": lang, "segment": s,
                    "latin_lang": latin_lang if LATIN.search(token + " " + (alias or "")) else None,
                    "alignment_text": (alias or token).replace(" ", "").replace(TATWEEL, ""),
                    "alias": alias, "start": None, "end": None, "score": None, "method": None,
                })
    if missing:
        sys.exit("These tokens have no letters to align (numbers, symbols). Add a spoken-form "
                 "alias for each in transcript.json -> aliases:\n  " + " ".join(sorted(set(missing))))
    return data, audio, words


def batches(segments):
    """Merge consecutive segments into windows of at most MAX_BATCH seconds."""
    out, cur = [], []
    for s, seg in enumerate(segments):
        if cur and seg["end"] - segments[cur[0]]["start"] > MAX_BATCH:
            out.append(cur)
            cur = []
        cur.append(s)
    if cur:
        out.append(cur)
    return out


def run_align(text_tokens, start, end, model, meta, audio, device):
    seg = {"start": start, "end": end, "text": " ".join(text_tokens)}
    result = align([seg], model, meta, audio, device, return_char_alignments=False)
    return result, result["word_segments"]


def timed(w):
    return w["start"] is not None and w["end"] is not None


def main():
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("transcript", type=Path)
    ap.add_argument("--out", type=Path, default=Path("captions"))
    ap.add_argument("--device", default="cuda" if torch.cuda.is_available() else "cpu")
    args = ap.parse_args()

    data, audio_path, words = load_transcript(args.transcript)
    segments = data["segments"]
    audio = load_audio(str(audio_path))
    duration = len(audio) / SAMPLE_RATE
    args.out.mkdir(parents=True, exist_ok=True)
    evidence, issues = [], []
    models = {}

    def model_for(lang):
        if lang not in models:
            models[lang] = load_align_model(lang, args.device, model_name=ALIGN_MODELS[lang])
        return models[lang]

    # Pass 1: Arabic-script words (and Latin words with an Arabic-script alias).
    arabic = [w for w in words if w["lang"] == "ar"]
    if arabic:
        model, meta = model_for("ar")
        for group in batches(segments):
            ids = [w for w in arabic if w["segment"] in group]
            if not ids:
                continue
            start = max(0.0, segments[group[0]]["start"] - PAD)
            end = min(duration, segments[group[-1]]["end"] + PAD)
            result, got = run_align([w["alignment_text"] for w in ids], start, end, model, meta, audio, args.device)
            if len(got) != len(ids):
                issues.append({"type": "token_mismatch", "pass": "ar", "segments": group,
                               "expected": len(ids), "got": len(got)})
                continue
            for w, g in zip(ids, got):
                if "start" in g and "end" in g:
                    w.update(start=g["start"], end=g["end"], score=g.get("score"),
                             method="ar-ctc-alias" if w["alias"] else "ar-ctc")
            evidence.append({"pass": "ar", "segments": group, "window": [start, end], "result": result})
        print(f"arabic pass: {sum(timed(w) for w in arabic)}/{len(arabic)} words timed", flush=True)

        # Re-align around overlaps with surrounding context; keep only the centre.
        timed_ar = [w for w in arabic if timed(w)]
        for k in range(1, len(timed_ar)):
            if timed_ar[k]["start"] >= timed_ar[k - 1]["end"]:
                continue
            lo, hi = max(0, k - 8), min(len(timed_ar), k + 9)
            local = timed_ar[lo:hi]
            ws = max(0.0, min(w["start"] for w in local) - 0.15)
            we = min(duration, max(w["end"] for w in local) + 0.15)
            result, got = run_align([w["alignment_text"] for w in local], ws, we, model, meta, audio, args.device)
            if len(got) != len(local):
                continue
            for pos, (w, g) in enumerate(zip(local, got), start=lo):
                if k - 3 <= pos <= k + 3 and "start" in g and "end" in g:
                    w.update(start=g["start"], end=g["end"], score=g.get("score"), boundary_realign=True)
            evidence.append({"pass": "ar-boundary", "index": timed_ar[k]["i"], "window": [ws, we], "result": result})

    # Pass 2: each Latin run with its own language model, boxed in by Arabic neighbours.
    runs = []
    for w in words:
        if w["latin_lang"]:
            run_lang = w["latin_lang"]
            prev = runs[-1] if runs else None
            if prev and prev["lang"] == run_lang and prev["ids"][-1] == w["i"] - 1 and words[w["i"] - 1]["segment"] == w["segment"]:
                prev["ids"].append(w["i"])
            else:
                runs.append({"lang": run_lang, "ids": [w["i"]]})

    for run in runs:
        first, last = run["ids"][0], run["ids"][-1]
        before = next((words[j] for j in range(first - 1, -1, -1) if timed(words[j]) and words[j]["lang"] == "ar"), None)
        after = next((words[j] for j in range(last + 1, len(words)) if timed(words[j]) and words[j]["lang"] == "ar"), None)
        seg_first, seg_last = segments[words[first]["segment"]], segments[words[last]["segment"]]
        start = before["end"] if before else max(0.0, seg_first["start"] - PAD)
        end = after["start"] if after else min(duration, seg_last["end"] + PAD)
        if timed(words[first]):  # had an Arabic-alias timing: search tightly around it
            start = max(start, words[first]["start"] - LATIN_PAD)
        if timed(words[last]):
            end = min(end, words[last]["end"] + LATIN_PAD)
        if end - start < MIN_WINDOW:
            issues.append({"type": "no_room_for_latin_run", "ids": run["ids"], "window": [start, end]})
            continue
        model, meta = model_for(run["lang"])
        text = [words[i]["word"].replace("’", "'") for i in run["ids"]]
        if any(words[i]["alias"] and not ARABIC.search(words[i]["alias"]) for i in run["ids"]):
            text = [words[i]["alignment_text"] if words[i]["alias"] and not ARABIC.search(words[i]["alias"])
                    else t for i, t in zip(run["ids"], text)]
        result, got = run_align(text, start, end, model, meta, audio, args.device)
        ok = len(got) == len(run["ids"]) and all("start" in g and "end" in g and g["end"] > g["start"] for g in got)
        evidence.append({"pass": run["lang"], "ids": run["ids"], "window": [start, end], "accepted": ok, "result": result})
        if not ok:
            issues.append({"type": "latin_run_failed", "lang": run["lang"], "ids": run["ids"], "text": " ".join(text)})
            continue
        for i, g in zip(run["ids"], got):
            w = words[i]
            # A strong Arabic-pronunciation match beats a weak native one.
            if timed(w) and (w["score"] or 0) > 0.4 and g.get("score", 0) < LOW_SCORE:
                w["latin_check"] = {"lang": run["lang"], "start": g["start"], "end": g["end"], "score": g.get("score"),
                                    "status": "kept stronger Arabic-alias timing"}
                continue
            if timed(w):
                w["arabic_alias_timing"] = {"start": w["start"], "end": w["end"], "score": w["score"]}
            w.update(start=g["start"], end=g["end"], score=g.get("score"), lang=run["lang"],
                     method=f"{run['lang']}-ctc-between-arabic-neighbours")
        print(f"{start:7.2f}-{end:7.2f} [{run['lang']}] {' '.join(text)}: {'ok' if ok else 'REVIEW'}", flush=True)

    # Checks. Flags are for a human; nothing is silently fixed.
    prev = None
    for w in words:
        if not timed(w):
            issues.append({"type": "unaligned", "i": w["i"], "word": w["word"]})
            continue
        if w["end"] <= w["start"]:
            issues.append({"type": "nonpositive_duration", "i": w["i"], "word": w["word"]})
        if (w["score"] is None or w["score"] < LOW_SCORE):
            issues.append({"type": "low_confidence", "i": w["i"], "word": w["word"], "score": w["score"]})
        if prev and w["start"] < prev["end"] - OVERLAP_TOL:
            issues.append({"type": "overlap", "i": w["i"], "word": w["word"], "previous": prev["word"]})
        prev = w

    out = {
        "audio": str(audio_path),
        "transcript": str(args.transcript),
        "stt_provider": data.get("provider"),
        "align_models": {k: ALIGN_MODELS[k] for k in models},
        "device": args.device,
        "review_status": "automated acoustic alignment; human listening not yet done",
        "words": words,
    }
    (args.out / "words.json").write_text(json.dumps(out, ensure_ascii=False, indent=2), encoding="utf-8")
    (args.out / "issues.json").write_text(json.dumps(issues, ensure_ascii=False, indent=2), encoding="utf-8")
    (args.out / "evidence.json").write_text(json.dumps(evidence, ensure_ascii=False, indent=2), encoding="utf-8")
    n = sum(timed(w) for w in words)
    print(f"{n}/{len(words)} words timed, {len(runs)} Latin runs, {len(issues)} review flags -> {args.out}/")


if __name__ == "__main__":
    main()
```
