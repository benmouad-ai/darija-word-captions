# Darija word-by-word captions

An agent skill for **word-by-word captions that stay in sync**, even when the speaker switches between Moroccan Darija, Arabic, French and English in the same sentence.

Darija stays in Arabic script. English and French stay in Latin letters, exactly as spoken. Every word gets its real start and end time from the audio.

## How it works

Accurate captions need two models:

1. **Speech-to-text** writes the words (Gemini Flash, Gemini 3.5 Transcribe, Microsoft MAI-Transcribe-2, or the free local model Moulsot).
2. **Forced alignment** gives every word the time it starts and ends. It runs locally and free with [WhisperX](https://github.com/m-bain/whisperX) and wav2vec2 models from Hugging Face: one Arabic model for Darija, plus an English and a French model for the code-switched words.

Then [HyperFrames](https://github.com/heygen-com/hyperframes) or [Remotion](https://www.remotion.dev) renders the styled captions on the video.

## Install

```bash
npx skills add benmouad-ai/darija-word-captions
```

Or copy `skills/darija-word-captions/` into `~/.claude/skills/` (Claude Code) or your Codex skills folder.

## Requirements

- Python 3.10+, FFmpeg, Node.js 18+
- `pip install whisperx==3.8.6`
- One speech-to-text option (an API key, or a GPU / Apple Silicon Mac for the local model)
- HyperFrames (`npx skills add heygen-com/hyperframes`) or Remotion (`npx skills add remotion-dev/skills`)

The full list, the workflow and a copy-paste setup prompt for Claude Code or Codex are in [`skills/darija-word-captions/SKILL.md`](skills/darija-word-captions/SKILL.md).

## Quick start

Paste this to Claude Code (or Codex) after installing:

> Use the darija-word-captions skill. Check the prerequisites, ask me which speech-to-text model and which editing tool I want, then run the full workflow on `test.mp4` and show me `issues.json` and a preview before a real video.

## What's inside

| File | What it is |
|---|---|
| `skills/darija-word-captions/SKILL.md` | The skill: prerequisites, workflow, setup prompt |
| `skills/darija-word-captions/scripts/align_words.py` | The alignment script (WhisperX, Arabic + English + French wav2vec2) |

## Alignment models

| Language | Model |
|---|---|
| Darija / Arabic | [`jonatasgrosman/wav2vec2-large-xlsr-53-arabic`](https://huggingface.co/jonatasgrosman/wav2vec2-large-xlsr-53-arabic) |
| French | [`jonatasgrosman/wav2vec2-large-xlsr-53-french`](https://huggingface.co/jonatasgrosman/wav2vec2-large-xlsr-53-french) |
| English | `WAV2VEC2_ASR_BASE_960H` (torchaudio) |

## License

MIT. Use it, change it, share it.
