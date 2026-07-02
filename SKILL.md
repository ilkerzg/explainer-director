---
name: explainer-director
description: Produce narrated explainer videos end-to-end on fal.ai where the agent acts as the FULL DIRECTOR making every creative decision per scene — screenplay with per-scene treatments (static Ken Burns / animated cutout composites / rare generated video clips), single-pass ElevenLabs TTS narration aligned to exact word timestamps, scene images from the bundled softfacetstyle Krea 2 LoRA, word-synced serif text cards with a wipe reveal, background music and entrance sound effects, and a mandatory frame-by-frame QA pass. Use when the user asks for an explainer, story, or history video ("make a video about X", "explainer üret") without specifying every asset themselves. Requires a fal.ai account with FAL_KEY set in the environment, plus python fal_client, Pillow, and ffmpeg/ffprobe.
---

# Explainer Director

The agent is the director: it writes the screenplay, decides every scene's treatment, produces every asset, composes with ffmpeg, and validates its own work frame-by-frame. Video generation models are NOT the default — most motion comes from compositing. Everything is used ONLY WHEN NEEDED: text sparingly, video rarely, inserts purposefully.

Work in `/tmp/<project>/`. `FAL_KEY` must be in the environment — export it, or source your env file. Long jobs: `nohup ... > run.log 2>&1 &`, poll for DONE, one writer per file.

## Pipeline (fixed order)

1. **Screenplay** — ~28-32 short English sentences, one per scene, each scene a tuple `(narration, treatment, visual_prompt, extras)`. The director assigns treatments deliberately: see [references/directing.md](references/directing.md) for the decision rules and editing grammar (inserts, stat beats, badges, anchor location).
2. **Voice** — single-pass TTS of the WHOLE script: `fal-ai/elevenlabs/tts/eleven-v3`, voice `YOUR_ELEVENLABS_VOICE_ID` (any ElevenLabs voice id, or a voice cloned via `fal-ai/elevenlabs/clone-voice`), `language_code:"en"`, stability 0.5. Never per-sentence.
3. **Timing** — `scribe-v2` word timestamps → **exact sentence alignment** (NOT proportional): each scene's last word is located near its expected position; cuts land `max(prev_end, first_word_start − 0.25s)`. Validate with a per-scene gap table (all gaps ≤ 0.25s). See [references/timing-text.md](references/timing-text.md).
4. **Assets** — scene plates at 1920×1080 + cutout subjects (plain-bg krea image → `fal-ai/ideogram/remove-background` → transparent PNG), all with the bundled softfacetstyle LoRA: trigger, palette and prompt templates in [references/style.md](references/style.md). **No yellow/beige wash, ever.**
5. **Clips** — ONLY for scenes that truly need generated motion (fire, complex crowd motion): Veo 3.1 Lite primary, grok-imagine fallback. 1-3 clips per video maximum.
6. **Compose** — per-scene ffmpeg: gentle Ken Burns for statics; animated overlay expressions for composites (slide + bob + clamp); word-synced text cards with left-to-right **wipe reveal** (never alpha fade); direct cuts; concat; mux narration with `-shortest`. Recipes in [references/motion-composites.md](references/motion-composites.md).
7. **Audio** — instrumental music bed at 0.11 gain with fades + entrance SFX (0.4 gain) at scene-start timestamps; `amix normalize=0` + limiter. See [references/audio.md](references/audio.md).
8. **QA (mandatory)** — contact sheet of plates; cutout facing check; composite mid-frames (subjects face their movement direction — "walking backwards" is a hard fail); text frames at sync+1s (in empty areas, fully crisp); scene-gap table; end-frame artifact scan; `volumedetect` (mix mean ≈ narration mean, max ≈ 0 dB).

## Hard rules

- **Director decides everything**; never ask the user which treatment to use per scene.
- Motion is **small but real** and purposeful — subjects move (ships bob and slide, armies converge), not just camera zoom; but never wild/smooth AI-slop motion.
- **Validate compositing direction**: cutouts often face opposite the prompt; view each cutout, set hflip so movers face their direction of travel, verify from a composed frame.
- **Text**: few moments per video, ALWAYS in empty areas reserved in the plate prompt, NEVER over characters; if the text matches narration words it appears word-synced; Baskerville SemiBold plain caps, large, centered in the empty region; wipe reveal in, wipe out.
- **No pen/hand/notebook artifacts**; exclusions live in positive prompts (krea has no negative_prompt).
- **Style fidelity**: every prompt restates the style anchor from style.md; cool neutral palette; no yellow.
- Total length = narration length; direct cuts only; ~2 minutes ≈ 30 sentences.

## Files

- [references/style.md](references/style.md) — softfacetstyle trigger, LoRA URL, palette, prompt templates
- [references/directing.md](references/directing.md) — treatment decision rules + measured editing grammar
- [references/timing-text.md](references/timing-text.md) — alignment algorithm + text card system (font, wipe, placement)
- [references/motion-composites.md](references/motion-composites.md) — Ken Burns, composite overlay math, cutout production, selective video
- [references/audio.md](references/audio.md) — music/SFX generation and the exact mix filtergraph
- [references/scripts.md](references/scripts.md) — runnable Python templates (build → clips → overlays → compose → audio)
