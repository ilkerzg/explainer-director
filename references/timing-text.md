# Timing alignment + text card system

## Exact sentence alignment (mandatory — proportional mapping is banned)

Proportional word-count mapping drifts ±1 word and glues inter-sentence pauses to the NEXT scene, so visuals appear ~1s before their sentence. Correct method:

```python
# words = scribe-v2 word entries; WN = normalized word texts
p = 0; bounds = []
for scene in scenes:
    toks = [norm(x) for x in scene_text.split() if norm(x)]
    exp = p + len(toks); last = toks[-1]
    best = None
    for j in range(max(p+1, exp-4), min(N, exp+5)+1):     # fuzzy search near expected end
        wj = WN[j-1]
        if wj == last or wj.startswith(last) or last.startswith(wj):
            if best is None or abs(j-exp) < abs(best-exp): best = j
    idx = best if best is not None else min(exp, N)
    bounds.append(idx); p = idx
# starts: the cut lands 0.25s before the sentence's first word (splits the pause)
start_i = 0 if i == 0 else max(prev_last_word_end, first_word_start - 0.25)
# scenes are contiguous; last scene ends at audio_end
```

`norm(t)`: NFKD-normalize, lowercase, keep alnum only. **Validate**: print a table of (visual_start, first_word_start, gap) — every gap must be ≤ 0.25s — and extract before-cut / at-cut / word-onset frames for one spot scene.

If a sync word falls at a sentence's END (titles like "...: the Trojan War."), REWRITE the sentence so the word comes early ("The Trojan War: ...") instead of accepting a truncated card.

## Text cards

**Rendering (PIL)**: Baskerville SemiBold (`/System/Library/Fonts/Supplemental/Baskerville.ttc`, index 1; substitute any serif font file on non-macOS systems), plain single-size CAPS (NO fake small-caps — mixed sizes top-align into a superscript mess), sizes 78-120px, +3px tracking, ink (30,26,20), no shadow, transparent PNG. Pass strings PRE-uppercased (Turkish: i→İ by hand). Stacked lines for titles ("THE / TROJAN / WAR").

**Placement**: reserve the empty area in the plate prompt ("LARGE EMPTY clear sky filling the left half"), then center the card in that area: `x = cx - png_w/2, y = cy - png_h/2`. Never over characters. Verify with a frame at sync+1s.

**Appearance = wipe, never fade**: left-to-right soft wipe in, reverse wipe out. ffmpeg `crop` cannot animate `w` (init-only); use `geq` on alpha with the text PNG fed as a `-loop 1` input (single-frame `movie=` streams break time-based filters):

```
[1:v]format=rgba,fps=30,
geq=r='r(X,Y)':g='g(X,Y)':b='b(X,Y)':
a='alpha(X,Y)*clip(((W+90)*clip(min((T-TS)/0.55,(TE-T)/0.45),0,1)-X)/60,0,1)'[txt];
[base][txt]overlay=x=X0:y=Y0:enable='between(t,TS,TE)'
```

- `TS` = sync word start (scene-local); `TE = max(TS+1.6, scene_dur-0.25)`.
- The `W+90` margin is REQUIRED — without it the trailing ~60px stays ghost-transparent at full reveal.
- Word sync: find the first word in the scene window whose normalized text starts with the target prefix.


## TTS length limit (hard)

`eleven-v3` rejects texts over **5000 characters**. For long scripts (~5+ minutes), split the
screenplay text at a ROW boundary into <4600-char chunks, TTS each chunk separately, decode
to wav, concat with ffmpeg, re-encode one narration.mp3, then upload it and run scribe-v2 ONCE
on the full file — global word timestamps keep the exact-alignment code unchanged. The join
lands on a natural sentence gap (a cut point), so pacing is unaffected.


## Text readability: auto-backing box

Cards must stay readable even when a plate's reserved area turns out busy. Before compositing,
measure the region under the card and add a white backing box when needed:

```python
from PIL import Image, ImageStat
def needs_backing(plate_path, x, y, w, h, pad=28):
    reg = Image.open(plate_path).convert("L").crop((max(0,x-pad), max(0,y-pad), x+w+pad, y+h+pad))
    st = ImageStat.Stat(reg)
    variance, mean = st.var[0], st.mean[0]
    contrast = abs(mean - 26)          # 26 = ink luminance
    return variance > 900 or contrast < 90
```

If true, re-render the card PNG on a rounded white box (padding ~28px, fill (250,249,246,235),
thin charcoal border optional) so the text owns its area — like a printed label. Never shrink
the text to fit noise; back it instead.
