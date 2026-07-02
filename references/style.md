# House style: `softfacetstyle` (bundled, ready to use)

Rounded-facet character construction on a cool neutral palette. Trained as a Krea 2 LoRA from a 50-image GPT Image 2 dataset anchored to a single style reference image. The weights are hosted on fal's CDN and are free to use as-is — no training required.

- **Trigger**: `softfacetstyle` (always first in the prompt)
- **LoRA URL**: `https://v3b.fal.media/files/b/0aa0a6ac/lxyuPGSABiVvQI83vZROe_krea2_lora_step_1000.safetensors` (1000 steps / res 768 / LR 1e-4)
- **Generate with**: `fal-ai/krea-2/turbo/lora`, `loras:[{path, scale:1.0}]`, plates at 1920×1080 (~6s), cutout sources at 1024².

## Style anchor (append to EVERY prompt)

```
, softly rounded geometric facet construction: round-cornered polygon heads, simple line eyes
and brows, arms and legs clearly connected to the body (no floating hands), flat cel shading,
clean bold dark outlines, bold saturated sunny palette (vivid grass green #6FBF44, bright sky
blue #4FB3E8, punchy orange #F59B2D, warm red #E2603F, deep cobalt #2E6FA3, warm brown #8A5A33,
charcoal #2E2E2E outlines, golden sand #EFC75E only as a small accent) PLUS warm skin tones for
all faces and hands (light skin #F2C9A0, tan skin #D99A6C, deep skin #A96B48) - faces are NEVER
white or blank, always a skin tone with simple line eyes, brows and a small mouth, HIGH
saturation and brightness, sunny daylight, NOT pastel, NOT muted, NOT dark, NO yellow wash
```

Rules learned the hard way:
- **The yellow wash is the #1 failure mode** — image models drift to warm cream backgrounds; the explicit NO-yellow clause is mandatory in dataset AND generation prompts.
- LoRA scale 1.0 for rich scenes (higher flattens composition); restate the anchor instead of raising scale.
- Drop realism words (mighty/heroic/cinematic lighting); they pull the base model through.
- Character identity across scenes: fixed costume color + a chest badge with initials, restated in every prompt.

## Prompt templates

**Scene plate** (1920×1080): `softfacetstyle, <event with full setting; reserve "LARGE EMPTY <area>" if a text card lands here><ANCHOR>`

**Cutout source** (1024², for compositing): `softfacetstyle, <single subject>, side view facing <left/right>, on a plain solid light gray background, whole subject visible, centered<ANCHOR>`

**Insert shot** (square, 1-1.5s beats): `softfacetstyle, a single <prop> on a plain background, centered, minimal<ANCHOR>`

## Dataset recipe (for training your own variant)

~108 images via `openai/gpt-image-2/edit` with one reference image in your target style as `image_urls`, prompt = "Use the reference image ONLY for its art style and copy that style EXACTLY: <anchor text>. Draw this scene in exactly that style: <subject>." Subjects span eras (ancient→future), characters AND objects/landscapes/crowds/close-ups, mixed aspects. Captions: `<your trigger>, <content only>` (no style/color words). Train: krea-2-trainer, 1000 steps, res 768, LR 1e-4.

## Palette v2 (user-approved, saturated & sunny)

Use EXACTLY these hexes in prompts: vivid grass green `#6FBF44`, bright sky blue `#4FB3E8`,
punchy orange `#F59B2D`, warm red `#E2603F`, deep cobalt `#2E6FA3`, leaf green `#4E9E3D`,
warm brown `#8A5A33`, charcoal `#2E2E2E` (outlines), golden sand `#EFC75E` ONLY as a small
accent (grounds prefer stone gray / green / blue). HIGH saturation + HIGH brightness, sunny;
NOT pastel, NOT muted, NOT dark, NO yellow wash.

**The full style anchor is MANDATORY in every generation prompt** — the v2 LoRA was trained on
a 108-image breadth dataset (empty scenes, objects, icons, maps, all eras) whose faces drifted
slightly toward generic cartoon; with the anchor ("round-cornered polygon heads, simple line
eyes...") the facet look locks back in. Bare prompts without the anchor give softer faces.


## Face & wardrobe failure modes (learned in production)

1. **A hex-restricted palette with NO skin tone produces white mask-faces** — "use EXACTLY these
   hexes" leaves the model nothing to paint skin with. The anchor above now includes three skin
   hexes and the "faces are NEVER white or blank" clause; keep them in every prompt.
2. **Era-neutral role words summon modern wardrobe**: "official" → business suit and tie,
   "guard" → epauletted uniform. Never use a bare role word — always attach period dress:
   "a roman quaestor in a white toga with a purple stripe", "legionaries in segmented armor
   with crested helmets". This is the same period-accuracy rule as directing.md, enforced at
   the word level.
3. **Clone faces**: when every character is hood+dash-eyes, vary hair/beard/age and give named
   characters their character-sheet descriptor; faces need eyes, brows AND a small mouth.
