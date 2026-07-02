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

## Symbol & word beats (expanded grammar)

Beyond quiet cards, use these ATTENTION beats sparingly (1 per 20-30s max), synced to their word:
- **Full-frame word slam**: a giant single word/year filling the frame over a flat PIL backdrop;
  appears with a fast wipe or a 2-frame scale-settle (1.06 → 1.0) for a "slam" feel.
- **Stamp**: a round seal/emblem PNG that lands with a quick scale-down (1.4 → 1.0 over ~0.18s)
  plus a thud SFX — ffmpeg: `scale=iw*max(1,1.4-2.2*t):-1` on the overlay input, or pre-render
  3 sizes and switch with enable windows.
- **X-out**: a bold red X drawn over an object insert — two thick strokes revealed sequentially
  with the geq wipe (stroke 1 for 0.15s, then stroke 2), plus a squeak/thud SFX.
- **Icon beats**: lightbulb = idea, clock = time, arrows = trends, speech-bubble-with-pictogram =
  quotes. Generate icons in-style on plain background, bg-remove, overlay like cutouts.
These carry the story's feel: sometimes the word IS the shot; sometimes a mark comments on the shot.
