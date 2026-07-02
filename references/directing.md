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
