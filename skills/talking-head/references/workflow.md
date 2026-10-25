# Local workflow

## Voice

Use a clean recording of yourself, or a speaker who has authorized the clone. A clean 30–60 second sample is a useful starting point; quality matters more than padding it out. A short sample may work less reliably. Cloning needs a suitable ElevenLabs plan.

```bash
node <skill>/scripts/talking-head.mjs clone-voice \
  --name 'My voice' --sample inputs/voice.wav --out voice.json --consent
```

`voice.json` contains the voice ID and whether provider verification is required. Complete verification in ElevenLabs when requested. Reusing this command with the same sample and output returns the saved voice; it does not create another clone. An existing authorized `voice_id` can be put straight into the plan instead.

## Plan

Image paths are relative to the plan file. `heading` is the on-screen heading; `text` is spoken. `points` is optional artwork content, not additional speech. All strings are plain text.

```json
{
  "language": "English",
  "voice_id": "YOUR_ELEVENLABS_VOICE_ID",
  "images": ["inputs/portrait.jpg"],
  "aspect_ratio": "16:9",
  "resolution": "768P",
  "scenes": [
    {
      "id": "intro",
      "kind": "talking",
      "text": "Your next video can start with a sentence. Here is how it works.",
      "heading": "Start with a sentence"
    },
    {
      "id": "steps",
      "kind": "motion",
      "text": "Write your message, choose your on-camera lines, and explain the rest with motion.",
      "heading": "From words to a film",
      "points": ["Write", "Speak", "Explain"]
    }
  ]
}
```

- Aspect ratio: `16:9`, `9:16` or `1:1`. Local output: 1280×720, 720×1280 or 960×960, at 30 fps. fal resolution describes the source shot, not the assembled output.
- H3 resolution: `480P` or `768P`, default supplied explicitly in the plan. No automatic model or resolution upgrades.
- H3 input: 1–9 JPG/PNG/WebP images plus one synthesized audio clip. First image sets the room and clothing.
- Each reference audio clip must fit the provider's 2–15 second range. Speech shorter than two seconds is padded with trailing silence locally, without another TTS call. Speech longer than 12.7 seconds is rejected to leave delivery headroom in a 5–15 second video; it is never silently cut off.
- TTS: `eleven_multilingual_v2`; transcript: `scribe_v2`; video: `minimax/h3/reference-to-video` on fal. OpenRouter is not an implemented provider in this release.

## Generate and resume

```bash
node <skill>/scripts/talking-head.mjs generate --plan plan.json --out runs/intro --dry-run
node <skill>/scripts/talking-head.mjs generate --plan plan.json --out runs/intro --max-video-seconds 30
```

Dry-run reads local inputs only. The second command is paid. Both keys must be set. `--max-video-seconds` caps the sum of H3 output seconds; it is **not a dollar budget** and does not cap TTS/transcription charges. TTS is generated before the measured H3 duration check. Read current rates in your provider accounts before a run; no price is hard-coded here.

`FAL_API_KEY` is accepted as an alias for `FAL_KEY`.

`--wait-seconds` defaults to 1200. A timeout stops local waiting; it does not cancel a provider job. Resume with the identical command and folder. A changed plan or photo is rejected so you cannot accidentally mix old takes and a new script. Do not edit `state.json` while a run is active.

If H3 submission timed out before its ID was saved, find that request in the fal dashboard, then:

```bash
node <skill>/scripts/talking-head.mjs recover --out runs/intro --scene intro --request-id REQUEST_ID
node <skill>/scripts/talking-head.mjs generate --plan plan.json --out runs/intro
```

For a TTS/transcription response lost in transit, check provider history first. `--retry-tts intro` or `--retry-transcription intro` permits repeating only that unfinished operation and may incur another charge. It does not overwrite a completed result. A confirmed failed H3 request stays saved; inspect its error in fal rather than blindly regenerating.

The lock prevents two processes from buying the same work. After a forced process termination, check that the old process is stopped, remove only `.lock` (or `.render.lock` for rendering), then resume. A clone with a lost response must be located in ElevenLabs; use its existing ID instead of creating another clone.

## Files

`film.json` contains source paths, durations and measured word times. `verification.json` compares the requested and transcribed text; formatting-only differences are ignored, other differences are marked for review. It does not certify voice similarity or lip sync. `speech.vtt` uses final, measured speech timings and is adjusted to frame-rounded joins on render.

`visuals.mjs` is editable artwork. `film.mp4` is the final output; `review/` has three composited frames per scene; `render-report.json` records media checks and outstanding transcript differences. The first render may use the starter design; replace the artwork to fit the actual brief.

Private outputs include the input text, synthesized audio, downloaded footage, voice IDs and provider asset URLs. Keep `runs/`, `inputs/`, `.env` and `voice.json` out of git. Deleting local files does not delete a provider-side voice or upload; manage those in the relevant provider account.

## Provider references

- [fal H3 reference-to-video schema](https://fal.ai/models/minimax/h3/reference-to-video/api)
- [fal machine-readable OpenAPI](https://fal.ai/api/openapi/queue/openapi.json?endpoint_id=minimax/h3/reference-to-video)
- [ElevenLabs instant voice cloning](https://elevenlabs.io/docs/api-reference/voices/ivc/create)
- [ElevenLabs speech with timing](https://elevenlabs.io/docs/api-reference/text-to-speech/convert-with-timestamps)
- [ElevenLabs transcription](https://elevenlabs.io/docs/api-reference/speech-to-text/convert)

Model IDs and input limits checked 2026-10-25. Provider availability and prices can change.
