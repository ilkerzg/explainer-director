# Motion: Ken Burns, cutout composites, selective video

All scenes 1920×1080 @30fps, `crf 19`, direct cuts.

## Static scenes — gentle Ken Burns

```
[0:v]scale=2600:-1,zoompan=z='min(zoom+0.0006,1.10)':d=<dur*30>:
x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s=1920x1080:fps=30,setsar=1[base]
```

Also the fallback for any failed clip (looped still + zoompan).

## Cutout production

1. Generate the subject with the LoRA on a **plain solid light gray background**, side view facing left/right, whole subject visible, 1024².
2. `fal-ai/ideogram/remove-background` (`image_url` → transparent PNG). Edges are excellent.
3. **VALIDATE FACING (hard rule)**: view every cutout on a checker/gray board — subjects often face OPPOSITE the prompt. A mover entering from the left and travelling right must FACE right; set `hflip` per cutout accordingly. After composing, extract a mid-scene frame and confirm subjects face their direction of travel. "Walking backwards" is a hard fail.

## Composite recipes (ffmpeg overlay expressions, `-loop 1` inputs, `-t dur`)

Ships gliding with bob (two depths, different speeds/phases):
```
[bg][s2]overlay=x='-360+(t/D)*1250':y='430+7*sin(2*PI*(t+1.3)/4.2)'[a];
[a][s1]overlay=x='-560+(t/D)*1750':y='520+9*sin(2*PI*t/3.6)'[base]
```

Two groups converging and STOPPING (clamps; meet ≈ 55-80% of scene):
```
[bg][g]overlay=x='min(180, -640+(t/D)*1150)':y=520[a];
[a][t2]overlay=x='max(1100, 1920-(t/D)*1450)':y=520[base]
```

Heavy object rolling with wobble:
```
[bg][h]overlay=x='-560+(t/D)*1560':y='470+3*sin(2*PI*t/1.5)'[base]
```

Notes: scale cutouts 360-640px wide by depth; y = ground line of the plate; movers may keep moving through the whole scene (fleet) or clamp at a meeting point (duel — pair the stop with a clash SFX at `start + 0.55*dur`).

## Selective video (rare)

Veo 3.1 Lite primary → grok-imagine v1.5 fallback. Motion prompt = the scene's real modest action + boilerplate: "Only the characters and scene elements themselves move, with a small modest amount of real movement. ABSOLUTELY NO pen, pencil, marker, hand or fingers anywhere in the frame." Never say "hand-drawn / flipbook / frame-by-frame / sketch coming to life" (summons a literal pencil). Trim clips to the exact scene duration at assembly; request duration ≥ scene duration. Scan each clip's END frame for artifacts.
