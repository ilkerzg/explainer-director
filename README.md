# explainer-director

A Claude skill that turns the agent into a full film director for narrated explainer videos, produced end-to-end on [fal.ai](https://fal.ai).

The agent writes the screenplay, assigns every scene a treatment (static Ken Burns, animated cutout composites, or a rare generated video clip), narrates the whole script in a single ElevenLabs TTS pass, aligns scene cuts to exact word timestamps, generates all imagery with a bundled Krea 2 style LoRA, renders word-synced serif text cards with a wipe reveal, mixes a music bed with entrance sound effects, and QA-checks its own frames before delivery. Most motion comes from ffmpeg compositing, not video models — it is cheaper, more controllable, and more charming.

## Install

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/explainer-director.git ~/Documents/GitHub/explainer-director
mkdir -p ~/.claude/skills
ln -s ~/Documents/GitHub/explainer-director ~/.claude/skills/explainer-director
```

Then ask Claude for an explainer video ("make a 2-minute video about the fall of Rome") and the skill takes over.

## Requirements

- A fal.ai account with `FAL_KEY` exported in the environment (or sourced from your own env file)
- Python 3 with `fal_client` and `Pillow` (`pip install fal-client Pillow`)
- `ffmpeg` and `ffprobe` on PATH
- Text cards default to the macOS Baskerville font (`/System/Library/Fonts/Supplemental/Baskerville.ttc`); substitute any serif font file on other platforms
- An ElevenLabs voice id for narration — use any voice id you like, or clone one via `fal-ai/elevenlabs/clone-voice`, and set it once in the build template

## Layout

```
SKILL.md                          the skill entry point (pipeline + hard rules)
references/
  style.md                        bundled softfacetstyle LoRA, palette, prompt templates
  directing.md                    treatment decision rules + measured editing grammar
  timing-text.md                  exact word-timestamp alignment + text card system
  motion-composites.md            Ken Burns, cutout composites, selective video
  audio.md                        music/SFX generation + the exact mix filtergraph
  scripts.md                      runnable Python templates (as fenced code blocks)
```

Everything is pure Markdown — the runnable pipeline templates live in [references/scripts.md](references/scripts.md) as Python code blocks you copy into your project directory.

## Bundled style

The `softfacetstyle` Krea 2 LoRA is provided ready-to-use, hosted on fal's CDN — no training needed. It renders characters with softly rounded geometric facet construction, flat cel shading, and a cool neutral palette. Trigger word, hosted weight URL, the full style-anchor prompt, and the dataset recipe (if you want to train your own variant) are in [references/style.md](references/style.md).

## License

MIT
