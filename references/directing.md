# Directing: treatment decisions + editing grammar

## Screenplay format

```python
SC = [  # (narration_sentence, treatment, visual_prompt, extras)
  ("...", "static",            "<plate prompt>",                    "text:t1"),
  ("...", "composite:<key>",   "<empty plate prompt, NO subjects>", "sfx:<key>"),
  ("...", "video:<key>",       "<plate prompt>",                    "sfx:<key>"),
]
```

One sentence per scene, ~28-32 scenes ≈ 2 minutes. Every visual must depict ITS sentence's event — never generic filler.

## Treatment decision rules (the director's core judgment)

- **static + Ken Burns** (default, ~70%): narrative beats, portraits, establishing shots, reactions.
- **composite** (~4-5 per video, the signature move): whenever the sentence describes MOVEMENT of discrete subjects — ships sailing, armies converging, a duel, something being pulled/carried. Empty plate + cutout subjects animated with ffmpeg overlay expressions. This is cheaper, more controllable, and more charming than generated video.
- **video** (1-3 per video MAXIMUM): only when the motion is intrinsically ungeneratable by compositing — fire/smoke, water surfaces, complex crowd choreography, something emerging/transforming. Veo 3.1 Lite primary (`duration` "4s"/"6s"/"8s" ≥ scene, `generate_audio:false`, `auto_fix:true`), grok-imagine v1.5 fallback (int 2-15s).
- **text card** (3-5 per video): title, place names, dates/durations, numbers. Only where the plate has a reserved empty area.
- **insert shot** (from the editing grammar below): a 1-1.5s cutaway inside a longer sentence.

## Editing grammar (measured from strong explainer channels; median shots 2.0-4.6s)

1. **2-3 shots per sentence** where the sentence carries multiple ideas: main scene + a quick symbolic insert landing exactly on its word (we have word timestamps). Keep median shot length 2-5s.
2. **Insert vocabulary**: single prop on plain background (crown, key, coins, map, hourglass); concept icons (lightbulb = idea, red X over object = negation, clock = time); giant full-frame word/year/number in the card font ("1892"-style).
3. **Stat beats**: render numbers visually — big percentage over a simple map/graphic — instead of only narrating them.
4. **Recurring-character identity**: fixed costume color + chest badge with initials, restated in every prompt (also fixes LoRA consistency).
5. **Speech-bubble-with-icon**: a character "says" a pictogram instead of words for quotes/claims.
6. **Anchor location**: one recurring set the story returns to (war camp, workshop, council hall) for rhythm.
7. Full-frame transition device (ink splash / smash to white): at most 1-2 per video.
8. Footnote-style small serif line at the bottom for asides; real-photo insert only when citing a real expert.

## Language

Default narration + on-screen text: **English**. Voice: `YOUR_ELEVENLABS_VOICE_ID` — any ElevenLabs voice id, or a voice cloned via `fal-ai/elevenlabs/clone-voice`; set it once in the build template. Turkish on request (`language_code:"tr"` / scribe `"tur"`; Turkish İ handled in card rendering).


## Character sheets (consistency across scenes)

Before writing scenes, define a CHARACTERS dict with a FULL fixed descriptor per named character
and reuse it VERBATIM in every prompt that shows them:

```python
CHARACTERS = {
  "alaric": "a tall broad king with a short dark beard, a simple iron crown, a fur-collared "
            "slate-blue cloak over a muted burgundy tunic, a round chest badge with an A mark",
}
# prompt: f"... {CHARACTERS['alaric']} standing before the gates ..."
```

Never paraphrase the descriptor — paraphrases drift the design between scenes.

## Period accuracy (hard rule)

Every prompt with people or objects must name PERIOD-ACCURATE dress and props explicitly
("roman tunic and sandals", "segmented legionary armor", "medieval wool cloak") — without this
the model dresses ancient scenes in modern caps, belts and uniforms. The QA pass audits for
anachronisms (see audio.md) and offending shots are regenerated, cropped or re-treated.

## Object-beat grammar (MEASURED from reference explainers — not optional garnish)

Frame-sampled study of two strong explainer channels (2s sampling, mid-video segments):

| device | restrained style | prop-heavy style |
|---|---|---|
| plain scene | 96% | 70% |
| full-frame insert | 4% | 16% |
| map/diagram | ~0% | 13% |
| **pop-in overlaid ON a scene** | **10% of frames** | **39% of frames** |

The prop-heavy grammar is the house default: in a narrated minute that means **a word-synced
object beat roughly every 6-8 seconds**, not 1-2 per video. Devices observed and how to use them:

1. **Noun literalization (the core rule)**: when the narration names a concrete noun (gold,
   tribes, crown, plague), SHOW it — a transparent prop PNG pops in over the scene exactly on
   the word (we have word timestamps). Props float in empty sky/wall areas or just above heads.
2. **Pop-settle animation**: 3 pre-scaled sizes switched with enable windows
   (1.35x for 0.07s → 1.12x for 0.07s → 1.0x until scene end) + a short pop SFX (~0.3 gain)
   on the beat. Reads as a snappy cartoon pop without per-frame scale expressions.
3. **X-out on a prop**: crown pops on "deposed", then a bold red X lands over it in two
   sequential strokes (stroke 2 at +0.12-0.14s) + thud SFX. Negation/death/ending beats.
4. **Word slam over the scene**: a giant card-font word/year stamped OVER the live scene
   (not only on flat backdrops).
5. **Label + tiny arrow**: small caps label pointing at a thing in-scene for asides.
6. **Overlay marks on faces/objects**: a single tear, sweat drop, ! or ? — one mark, brief.
7. **Repeating micro-beats**: the same small mark popping 2-3 times in a row synced to a word
   sequence ("one bad decision at a time" → three X's popping left to right).

Production: generate props square on a plain solid light-gray background, `remove-background`,
PIL-trim transparent margins (accurate centering), overlay on the CONCATENATED video at
absolute word times as a separate pass — no need to touch per-scene segments. Pop SFX beats
join the final mix via adelay (pop 0.30, thud 0.38 gain).

## Symbol & word beats (attention devices, use sparingly on top of object beats)

- **Full-frame word slam**: giant single word/year filling the frame; fast wipe or 2-frame
  scale-settle (1.06 → 1.0). 1 per 20-30s max.
- **Stamp**: round seal/emblem PNG landing with a scale-down (1.4 → 1.0 over ~0.18s) + thud.
- **Icon beats**: lightbulb = idea, clock = time, arrows = trends, speech-bubble-with-pictogram
  = quotes. Generated in-style, bg-removed, overlaid like cutouts.
Sometimes the word IS the shot; sometimes a mark comments on the shot.
