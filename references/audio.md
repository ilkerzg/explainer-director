# Audio: music bed, entrance SFX, mix, QA

## Music

`fal-ai/elevenlabs/music`: instrumental underscore fitting the topic, `force_instrumental: true`, `music_length_ms = ceil(video_seconds)*1000`. Regenerate whenever narration length changes by more than ~3s. Prompt shape: "gentle <era/mood> underscore, <2-3 instruments>, restrained, cinematic, builds subtly, no vocals."

## SFX (entrances only, 5-8 per video)

`fal-ai/elevenlabs/sound-effects/v2` (`text`, `duration_seconds` 1.2-3.5, `prompt_influence` 0.4). One per composite/video entrance moment: waves+hull creaks (ships), distant drums (armies), a single sword clash (duel meet: `start + 0.55*dur`), wooden wheel creaks (rolling object), fire crackle. Place each at its scene's START timestamp (from the alignment), except meet-time effects.

## Mix (one filtergraph; watch the input index offset)

Video is input 0 → narration `[1:a]`, music `[2:a]`, SFX from `[3:a]`. Getting this wrong = exit 234.

```
[1:a]volume=1.0[n];
[2:a]volume=0.11,afade=t=in:st=0:d=2,afade=t=out:st=<dur-3>:d=3[m];
[3:a]adelay=<ms>|<ms>,volume=0.4[s1]; ...
[n][m][s1]...amix=inputs=<N>:normalize=0:duration=first[mixed];
[mixed]alimiter=limit=0.95[aout]
```

`-map 0:v -map [aout] -c:v copy -c:a aac -b:a 192k -shortest`. Levels: narration 1.0 dominant, music 0.11 (≈ −23 dB under narration), SFX 0.4.

## QA checklist (every delivery)

1. Contact sheet of all plates → style consistency, no yellow wash, reserved empty areas intact.
2. Brightness scan (grayscale mean < 38 → regenerate brighter).
3. Cutout facing board + composed mid-frames (facing = direction of travel).
4. Text frames at sync+1s → in empty areas, fully crisp (no ghost edge), correct language.
5. Scene-gap table: every visual-start ≤ 0.25s before its first word.
6. End-frame scan of video clips (no pen/hand artifacts).
7. `ffmpeg -hide_banner -i out.mp4 -af volumedetect -f null -`: mix mean ≈ narration-alone mean; max ≈ −0.1 dB (no clipping). (`-v error` suppresses volumedetect output — don't use it here.)
8. Duration = narration length; filmstrip of ~8 spread timestamps matches the screenplay order.


## Plate QA at scale (mandatory for 60+ scene builds)

1. **Persist every plate prompt in plan.json** (`scene["plate"]`). Audits must compare the image against the PLATE PROMPT (visual intent), never against the narration sentence — metaphorical narration makes a narration-based auditor flag good plates, and mass-regenerating from narration destroys authored prompts (reserved text areas, composition notes).
2. **Style LoRAs trained on character-heavy datasets have a crowd bias**: they insert rows of foreground people into "empty" scenes. Prose clauses ("no people") are stochastic. The reliable fix is a **retry-until-clean loop**: generate → focused vision check ({"people":bool,"top_clear":bool}) → new seed on failure; after 2 failures drop LoRA scale (1.0 → 0.75 → 0.6). Converges in 1-3 tries.
3. **Pure-backdrop plates (flat color cards) should be drawn with PIL**, not generated — guaranteed clean.
4. Never write the word "EMPTY" in a plate prompt where text could render — it can leak as literal rendered text. Use "a large clear open sky" phrasing.
