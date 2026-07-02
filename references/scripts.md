# Runnable pipeline templates (proven in production)

Copy each block into `/tmp/PROJECT/<name>.py` (pick your own project directory), edit the CONFIG/screenplay, run with `FAL_KEY` in the environment.
Order: **build → clips (if any video scenes) → overlays → compose → audio**.
These are the exact scripts that produced a real delivered video, with the project path and screenplay parameterized. The screenplay rows below are placeholders — write your own; **add ~15 rows per minute of video**.

## `build.py` — screenplay assets + single-pass TTS + exact alignment

```python
import os, json, time, urllib.request
from concurrent.futures import ThreadPoolExecutor
import fal_client
W="/tmp/PROJECT"
LORA="https://v3b.fal.media/files/b/0aa0a171/vAJPaNucNT98AKJIyd9HM_krea2_lora_step_1000.safetensors"
IMG="fal-ai/krea-2/turbo/lora"; RB="fal-ai/ideogram/remove-background"
VOICE="YOUR_ELEVENLABS_VOICE_ID"  # any ElevenLabs voice id, or a voice cloned via fal-ai/elevenlabs/clone-voice
ST="softfacetstyle, "
ANCH=(", softly rounded geometric facet construction: round-cornered polygon heads, simple line eyes and "
 "brows, arms and legs clearly connected to the body, flat cel shading, clean bold dark outlines, cool "
 "neutral palette (slate blue, muted teal, terracotta, muted burgundy, sage green, charcoal, off-white), "
 "cool light gray background tones, NO yellow, NO beige, NO cream, NO amber wash")

# scene = (narration_sentence, treatment, visual_prompt, extras)
# treatment: static | composite:<key> | video:<key>
# extras: "" | "text:<card>" | "sfx:<key>" | "text:<card>|sfx:<key>"
# EXAMPLE ROWS ONLY — replace with your screenplay; add ~15 rows per minute of video.
SC=[
 ("The lighthouse: for two hundred years it guided every ship home.", "static",
  "a tall stone lighthouse on a rocky headland at dawn, LARGE EMPTY clear sky filling the left half", "text:t1"),
 ("One autumn night, the fishing fleet raced a storm back to the harbor.", "composite:boats",
  "a wide open sea with rolling waves under a darkening evening sky, NO boats, empty horizon", "sfx:waves"),
 ("By morning the storm had passed, and smoke curled from every chimney in the village.", "video:smoke",
  "a small seaside village at sunrise, thin smoke rising from stone chimneys, calm sea beyond", "sfx:morning"),
]

# cutout sources (subject on plain light background for clean removal)
CUT={
 "boat1":"a single small wooden fishing boat with one sail, side view facing right, on a plain solid light gray background, whole boat visible, centered",
 "boat2":"a smaller wooden fishing boat with a furled sail, side view facing right, on a plain solid light gray background, whole boat visible, centered",
}

os.makedirs(f"{W}/assets", exist_ok=True); os.makedirs(f"{W}/cut", exist_ok=True)
def gen_img(key, prompt, w=1920, h=1080, seed=0):
    for a in range(3):
        try:
            r=fal_client.subscribe(IMG, arguments={"prompt":ST+prompt+ANCH,
                "loras":[{"path":LORA,"scale":1.0}],"image_size":{"width":w,"height":h},
                "num_images":1,"output_format":"jpeg","seed":seed})
            return (key, r["images"][0]["url"])
        except Exception as e:
            print("retry",key,str(e)[:80],flush=True); time.sleep(5*(a+1))
    return (key, None)
jobs=[("bg%02d"%i, s[2], 1920,1080, 4000+i) for i,s in enumerate(SC)]
jobs+=[(k, p, 1024,1024, 6100+n) for n,(k,p) in enumerate(CUT.items())]
print(f"images: {len(jobs)}",flush=True)
urls={}
with ThreadPoolExecutor(max_workers=6) as ex:
    for k,u in ex.map(lambda j: gen_img(*j), jobs):
        urls[k]=u; print(k,"ok" if u else "FAIL",flush=True)
cuts={}
for k in CUT:
    for a in range(3):
        try:
            r=fal_client.subscribe(RB, arguments={"image_url":urls[k]})
            cuts[k]=r["image"]["url"]; print("cut",k,"ok",flush=True); break
        except Exception: time.sleep(5*(a+1))

# narration: single-pass TTS of the WHOLE script (never per-sentence)
text=" ".join(s[0] for s in SC)
print("tts...",flush=True)
r=fal_client.subscribe("fal-ai/elevenlabs/tts/eleven-v3", arguments={"text":text,"voice":VOICE,
    "stability":0.5,"apply_text_normalization":"auto","language_code":"en","timestamps":True})
aurl=r["audio"]["url"]; urllib.request.urlretrieve(aurl, f"{W}/assets/narration.mp3")
print("stt...",flush=True)
stt=fal_client.subscribe("fal-ai/elevenlabs/speech-to-text/scribe-v2", arguments={"audio_url":aurl,
    "language_code":"eng","diarize":False,"tag_audio_events":False})
words=[w for w in stt.get("words",[]) if w.get("type")=="word"]

# EXACT sentence alignment (proportional mapping is banned — it drifts and glues pauses to the next scene)
import unicodedata
def norm(t):
    t=unicodedata.normalize("NFKD",t.lower()); return "".join(c for c in t if c.isalnum())
WN=[norm(w["text"]) for w in words]; N=len(WN)
p=0; bounds=[]
for s in SC:
    toks=[norm(x) for x in s[0].split() if norm(x)]
    exp=p+len(toks); last=toks[-1]; best=None
    for j in range(max(p+1,exp-4), min(N,exp+5)+1):   # fuzzy search near the expected end
        wj=WN[j-1]
        if wj==last or wj.startswith(last) or last.startswith(wj):
            if best is None or abs(j-exp)<abs(best-exp): best=j
    idx=best if best is not None else min(exp,N)
    bounds.append(idx); p=idx
bounds[-1]=N
starts=[]; prev_idx=0; prev_end_t=0.0
for i,idx in enumerate(bounds):
    ft=words[prev_idx]["start"]
    st=0.0 if i==0 else max(prev_end_t, round(ft-0.25,2))   # cut lands 0.25s before the first word
    starts.append(st); prev_end_t=words[idx-1]["end"]; prev_idx=idx
audio_end=words[-1]["end"]
scenes=[]
for i,s in enumerate(SC):
    st=starts[i]; en=starts[i+1] if i<len(SC)-1 else audio_end
    scenes.append({"i":i,"tr":s[0],"treat":s[1],"extra":s[3],"bg":urls.get("bg%02d"%i),
                   "start":round(st,2),"end":round(en,2),"dur":round(en-st,2),
                   "w0":bounds[i-1] if i>0 else 0,"w1":bounds[i]})
json.dump({"scenes":scenes,"cuts":cuts,"words":words}, open(f"{W}/plan.json","w"), ensure_ascii=False, indent=1)
def dl(args):
    k,u,dst=args
    if u:
        try: urllib.request.urlretrieve(u,dst)
        except Exception: print("dl fail",k,flush=True)
dls=[("bg%02d"%i, urls.get("bg%02d"%i), f"{W}/assets/bg{i:02d}.jpg") for i in range(len(SC))]
dls+=[(k, cuts.get(k), f"{W}/cut/{k}.png") for k in CUT]
with ThreadPoolExecutor(max_workers=8) as ex: list(ex.map(dl, dls))
print("audio_end",round(audio_end,1),flush=True)
# validation table: every gap must be <= 0.25s
print(" #  visual  speech   gap  treat            sentence",flush=True)
for i,s in enumerate(scenes):
    ft=words[s["w0"]]["start"]
    print(f"{i+1:2d} {s['start']:7.2f} {ft:7.2f} {ft-s['start']:5.2f}  {s['treat']:16s} {s['tr'][:36]}",flush=True)
print("DONE",flush=True)
```

## `clips.py` — selective video scenes (Veo primary, grok fallback)

```python
import json, time, urllib.request, math
import fal_client
d=json.load(open("/tmp/PROJECT/plan.json"))
VEO="fal-ai/veo3.1/lite/image-to-video"; GROK="xai/grok-imagine-video/v1.5/image-to-video"
# scene_idx -> motion prompt: the scene's real modest action + the mandatory no-pen boilerplate.
# 1-3 clips per video MAXIMUM. Never say "hand-drawn / flipbook / sketch coming to life".
JOBS={2:("Thin smoke curls slowly upward from the chimneys and the water glints gently. "
        "Only the smoke and the water move, with a small modest amount of motion. No people appear. "
        "Static camera. ABSOLUTELY NO pen, pencil, marker, hand or fingers anywhere in the frame.")}
for i,prompt in JOBS.items():
    s=d["scenes"][i]; out=f"/tmp/PROJECT/vid{i:02d}.mp4"
    ok=False
    for a in range(2):
        try:
            r=fal_client.subscribe(VEO, arguments={"image_url":s["bg"],"prompt":prompt,
                "duration":"4s" if s["dur"]<=4 else ("6s" if s["dur"]<=6 else "8s"),
                "aspect_ratio":"16:9","resolution":"720p","generate_audio":False,"auto_fix":True})
            urllib.request.urlretrieve((r.get("video") or {}).get("url"),out); ok=True; print(i,"veo ok",flush=True); break
        except Exception as e: print(i,"veo retry",str(e)[:80],flush=True); time.sleep(6)
    if not ok:
        for a in range(3):
            try:
                r=fal_client.subscribe(GROK, arguments={"image_url":s["bg"],"prompt":prompt,
                    "duration":int(min(15,max(2,math.ceil(s["dur"])))),"resolution":"720p"})
                urllib.request.urlretrieve((r.get("video") or {}).get("url"),out); print(i,"grok ok",flush=True); break
            except Exception: time.sleep(6)
print("CLIPSDONE",flush=True)
```

## `overlays.py` — text cards (Baskerville caps, PIL)

```python
# Title cards: plain single-size CAPS, Baskerville SemiBold, crisp ink, no fake small-caps.
# Pass strings PRE-uppercased (Turkish: i -> İ by hand). Non-macOS: substitute any serif font file.
import os
from PIL import Image, ImageDraw, ImageFont
W="/tmp/PROJECT"; os.makedirs(f"{W}/txt", exist_ok=True)
FONT="/System/Library/Fonts/Supplemental/Baskerville.ttc"; INK=(30,26,20,255); TRACK=3
def render(name, lines, size, leading=1.22):
    f=ImageFont.truetype(FONT, size, index=1)  # index 1 = SemiBold
    dummy=ImageDraw.Draw(Image.new("RGBA",(8,8)))
    def linew(s): return sum(dummy.textlength(c,font=f)+TRACK for c in s)-TRACK
    wmax=int(max(linew(l) for l in lines))+12
    lh=int(size*leading); H=lh*len(lines)
    im=Image.new("RGBA",(wmax+12,H+16),(0,0,0,0)); d=ImageDraw.Draw(im)
    for li,l in enumerate(lines):
        x=(wmax+12-linew(l))//2; y=li*lh+8
        for c in l:
            d.text((x,y),c,font=f,fill=INK); x+=dummy.textlength(c,font=f)+TRACK
    im.save(f"{W}/txt/{name}.png"); print(name, im.size)
render("t1", ["THE","LIGHTHOUSE"], 110)   # one render() call per text card in the screenplay
print("DONE")
```

## `compose.py` — per-scene ffmpeg (Ken Burns / composites / video / pop-in text) + concat + narration

```python
# Director's composition: per-scene ffmpeg builds (kenburns / animated composites / video),
# word-synced text cards with a FAST pop-in (never fade/wipe), concat, narration mux.
import json, os, subprocess, unicodedata
W="/tmp/PROJECT"
d=json.load(open(f"{W}/plan.json"))
scenes=d["scenes"]; words=d["words"]
os.makedirs(f"{W}/segs", exist_ok=True)
FPS=30; WD,HT=1920,1080

def norm(t):
    t=unicodedata.normalize("NFKD", t.lower())
    return "".join(c for c in t if c.isalnum())

def word_time(scene, target_prefix):
    """scene-local start time of the first word in the scene window matching the prefix."""
    for wo in words[scene["w0"]:scene["w1"]]:
        if norm(wo["text"]).startswith(norm(target_prefix)):
            return max(0.0, wo["start"]-scene["start"])
    return 0.3

# (scene_idx, png, sync_word, cx, cy) — cards centered in EMPTY areas reserved in the plate prompts
TEXTS=[(0,"t1","lighthouse", 470, 300)]
TXT={i:(p,s,x,y) for i,p,s,x,y in TEXTS}

def kenburns(dur):
    fr=max(1,round(dur*FPS))
    return (f"[0:v]scale=2600:-1,zoompan=z='min(zoom+0.0006,1.10)':d={fr}:"
            f"x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s={WD}x{HT}:fps={FPS},setsar=1[base]")

def run(cmd): subprocess.run(cmd, check=True)

from PIL import Image as _PImage
def txt_chain(idx, s, in_label, txt_input=1):
    """FAST POP-IN (the house rule — never a fade, never a slow wipe): 3 pre-scaled
    card PNGs switched with enable windows, 1.18x -> 1.06x -> 1.0x; instant out."""
    if idx not in TXT: return ("", in_label, [])
    png,sync,cx,cy=TXT[idx]
    base=_PImage.open(f"{W}/txt/{png}.png")
    ts=word_time(s, sync)
    te=max(ts+1.6, s["dur"]-0.25)          # visible until just before the cut
    f=""; ol=in_label; files=[]
    for j,(sc_,ta,tb) in enumerate([(1.18,ts,ts+0.06),(1.06,ts+0.06,ts+0.12),(1.0,ts+0.12,te)]):
        w=max(2,int(base.width*sc_)); h=max(2,int(base.height*sc_))
        p=f"{W}/txt/{png}_{j}.png"; base.resize((w,h)).save(p); files.append(p)
        f+=(f";[{txt_input+j}:v]format=rgba[tx{idx}{j}];"
            f"[{ol}][tx{idx}{j}]overlay=x={cx-w//2}:y={cy-h//2}:enable='between(t,{ta:.2f},{tb:.2f})'[to{idx}{j}]")
        ol=f"to{idx}{j}"
    return (f, ol, files)

for s in scenes:
    i=s["i"]; dur=s["dur"]; out=f"{W}/segs/seg{i:02d}.mp4"
    bg=f"{W}/assets/bg{i:02d}.jpg"; treat=s["treat"]
    if treat.startswith("video:") and os.path.exists(f"{W}/vid{i:02d}.mp4"):
        fc=f"[0:v]scale={WD}:{HT}:force_original_aspect_ratio=increase,crop={WD}:{HT},fps={FPS},setsar=1[base]"
        run(["ffmpeg","-y","-v","error","-i",f"{W}/vid{i:02d}.mp4","-t",f"{dur}",
             "-filter_complex",fc,"-map","[base]","-an","-r","30","-c:v","libx264","-preset","medium","-crf","19",out])
    elif treat=="composite:boats":
        # Two depths, different speeds and phases; far boat smaller and slower.
        # VALIDATE FACING: movers entering from the left travelling right must FACE right
        # (view each cutout, add hflip to its scale chain if it faces the wrong way).
        fc=(f"[0:v]scale={WD}:{HT},setsar=1,fps={FPS}[bg];"
            f"[1:v]scale=520:-1[s1];[2:v]scale=360:-1[s2];"
            f"[bg][s2]overlay=x='-360+(t/{dur})*1250':y='430+7*sin(2*PI*(t+1.3)/4.2)'[a];"
            f"[a][s1]overlay=x='-560+(t/{dur})*1750':y='520+9*sin(2*PI*t/3.6)'[pre]")
        tfc,ol,extra=txt_chain(i,s,"pre",txt_input=3)   # 3 card PNGs start at input 3: bg(0) + two cutouts(1,2)
        cmd=["ffmpeg","-y","-v","error","-loop","1","-i",bg,"-loop","1","-i",f"{W}/cut/boat1.png",
             "-loop","1","-i",f"{W}/cut/boat2.png"]
        for e in extra: cmd+=["-loop","1","-i",e]
        run(cmd+["-t",f"{dur}","-filter_complex",fc+tfc,"-map",f"[{ol if tfc else 'pre'}]","-an","-r","30",
                 "-c:v","libx264","-preset","medium","-crf","19",out])
    else:  # static kenburns (+ optional text card)
        fc=kenburns(dur)
        tfc,ol,extra=txt_chain(i,s,"base")
        cmd=["ffmpeg","-y","-v","error","-loop","1","-i",bg]
        for e in extra: cmd+=["-loop","1","-i",e]
        run(cmd+["-t",f"{dur}","-filter_complex",fc+tfc,"-map",f"[{ol}]","-an","-r","30",
                 "-c:v","libx264","-preset","medium","-crf","19",out])
    print(f"seg {i:02d} {treat}", flush=True)

open(f"{W}/concat.txt","w").write("".join(f"file '{W}/segs/seg{s['i']:02d}.mp4'\n" for s in scenes))
run(["ffmpeg","-y","-v","error","-f","concat","-safe","0","-i",f"{W}/concat.txt",
     "-c:v","libx264","-preset","medium","-crf","19","-r","30","-pix_fmt","yuv420p",f"{W}/video_only.mp4"])
run(["ffmpeg","-y","-v","error","-i",f"{W}/video_only.mp4","-i",f"{W}/assets/narration.mp3",
     "-c:v","copy","-c:a","aac","-b:a","192k","-shortest",f"{W}/draft.mp4"])
print("COMPOSED", flush=True)
```

## `audio.py` — music + entrance SFX mix

```python
import json, math, os, subprocess, urllib.request
from concurrent.futures import ThreadPoolExecutor
import fal_client
W="/tmp/PROJECT"
d=json.load(open(f"{W}/plan.json")); sc=d["scenes"]
VID=f"{W}/video_only.mp4"; NAR=f"{W}/assets/narration.mp3"
dur=float(subprocess.run(["ffprobe","-v","error","-show_entries","format=duration","-of","default=nk=1:nw=1",VID],capture_output=True,text=True).stdout.strip())
os.makedirs(f"{W}/audio",exist_ok=True)
def music():
    r=fal_client.subscribe("fal-ai/elevenlabs/music", arguments={
        "prompt":("Gentle coastal folk underscore: acoustic guitar, soft strings, light hand percussion, "
                  "restrained, cinematic instrumental, builds subtly, no vocals."),
        "force_instrumental":True,"music_length_ms":int(math.ceil(dur)*1000)})
    return r["audio"]["url"]
# (key, scene_idx, time_frac_within_scene, description, seconds)
SFX=[("waves",1,0.0,"ocean waves with wooden hull creaks and rope sounds",3.0),
     ("morning",2,0.0,"distant seagulls and a soft calm morning harbor ambience",2.5)]
def sfx(k,t,s):
    r=fal_client.subscribe("fal-ai/elevenlabs/sound-effects/v2", arguments={"text":t,"duration_seconds":s,"prompt_influence":0.4})
    return (k, r["audio"]["url"])
print("gen music+sfx...",flush=True)
with ThreadPoolExecutor(max_workers=4) as ex:
    fm=ex.submit(music); futs=[ex.submit(sfx,k,t,s) for k,_,_,t,s in SFX]
    urllib.request.urlretrieve(fm.result(), f"{W}/audio/music.mp3")
    for f in futs:
        k,u=f.result(); urllib.request.urlretrieve(u, f"{W}/audio/{k}.mp3")
# Mix: video is input 0, so narration is [1:a], music [2:a], SFX start at [3:a].
# Getting this input-index offset wrong = ffmpeg exit 234.
inputs=["-i",NAR,"-i",f"{W}/audio/music.mp3"]
fc=["[1:a]volume=1.0[n]", f"[2:a]volume=0.11,afade=t=in:st=0:d=2,afade=t=out:st={dur-3:.2f}:d=3[m]"]
labels=["[n]","[m]"]; k=3
for key,si,frac,_,_ in SFX:
    s=sc[si]; ms=int(round((s["start"]+frac*s["dur"])*1000))
    inputs+=["-i",f"{W}/audio/{key}.mp3"]
    fc.append(f"[{k}:a]adelay={ms}|{ms},volume=0.4[s{k}]"); labels.append(f"[s{k}]"); k+=1
fc.append("".join(labels)+f"amix=inputs={len(labels)}:normalize=0:duration=first[mixed]")
fc.append("[mixed]alimiter=limit=0.95[aout]")
out=f"{W}/final.mp4"
subprocess.run(["ffmpeg","-y","-v","error","-i",VID]+inputs+["-filter_complex",";".join(fc),
    "-map","0:v","-map","[aout]","-c:v","copy","-c:a","aac","-b:a","192k","-shortest",out],check=True)
print("FINAL:",out,flush=True)
```


## `audit_all.py` — mandatory post-build plate audit (anachronism / text-leak / blank-face / crowd)

```python
# MANDATORY post-build audit: every plate checked for anachronism / text-leak / unwanted people.
# Flagged plates get their prompt rewritten with period-accurate wording (LLM) and are
# regenerated retry-until-clean. This runs as a pipeline STEP, not a manual spot-check.
import json, re, time, urllib.request
from concurrent.futures import ThreadPoolExecutor
import fal_client
W="/tmp/PROJECT"
LORA="https://v3b.fal.media/files/b/0aa0a6ac/lxyuPGSABiVvQI83vZROe_krea2_lora_step_1000.safetensors"
ERA="late antique Rome (5th century AD); the final scenes may show Byzantium and ruins"
VLM,MODEL="openrouter/router/vision","anthropic/claude-sonnet-4.6"
ANCH=(", softly rounded geometric facet construction: round-cornered polygon heads, simple line eyes and "
 "brows, a small simple mouth on each face, connected limbs, flat cel shading, clean bold dark outlines, "
 "bold saturated sunny palette (vivid grass green #6FBF44, bright sky blue #4FB3E8, punchy orange #F59B2D, "
 "warm red #E2603F, deep cobalt #2E6FA3, warm brown #8A5A33, charcoal #2E2E2E outlines) PLUS warm skin "
 "tones for all faces and hands (light skin #F2C9A0, tan skin #D99A6C, deep skin #A96B48) - faces are "
 "NEVER white or blank. HIGH saturation, sunny, NOT pastel, NOT dark")
d=json.load(open(f"{W}/plan.json"))
SYS=("You audit a cartoon plate for an explainer video set in: "+ERA+". "
 "Compare the image to its INTENDED SCENE. Output ONE-LINE JSON: "
 '{"anachronism":true|false,"what":"<=10 words","text_leak":true|false,"blank_faces":true|false}. '
 "anachronism=true if ANY clothing or object is modern or wrong-era (business suits, ties, epaulette "
 "uniforms, peaked caps, modern furniture). blank_faces=true if faces are white featureless masks.")
def audit(s):
    for a in range(3):
        try:
            r=fal_client.subscribe(VLM, arguments={"model":MODEL,"system_prompt":SYS,
                "prompt":"Intended scene: "+s.get("plate", s["tr"]),"image_urls":[s["bg"]],
                "temperature":0.0,"max_tokens":90})
            m=re.search(r"\{.*\}",(r.get("output") or r.get("response") or ""),re.S)
            return (s["i"], json.loads(m.group(0)) if m else {})
        except Exception: time.sleep(4*(a+1))
    return (s["i"],{})
def rewrite(plate):
    for a in range(3):
        try:
            r=fal_client.subscribe("openrouter/router", arguments={"model":"anthropic/claude-sonnet-4.6",
                "system_prompt":("Rewrite the scene prompt so every person/role wears explicitly PERIOD-ACCURATE "
                  "dress and every object is period-accurate for "+ERA+". Replace era-neutral role words "
                  "(official, guard, soldier) with specific period roles + dress. Keep composition notes "
                  "(reserved empty areas) untouched. Output ONLY the rewritten prompt line."),
                "prompt":plate,"temperature":0.3,"max_tokens":200})
            t=(r.get("output") or r.get("response") or "").strip().splitlines()[0]
            if len(t)>20: return t
        except Exception: time.sleep(4)
    return plate
print("auditing ALL",len(d["scenes"]),"plates...",flush=True)
res={}
with ThreadPoolExecutor(max_workers=8) as ex:
    for i,a in ex.map(audit, d["scenes"]): res[i]=a
flag=[i for i,a in res.items() if a.get("anachronism") or a.get("text_leak") or a.get("blank_faces")]
print("FLAGGED:",sorted(flag),flush=True)
for i in sorted(flag): print(" ",i,res[i],flush=True)
for i in sorted(flag):
    s=d["scenes"][i]
    newp=rewrite(s.get("plate", s["tr"]))
    print(i,"rewritten:",newp[:100],flush=True)
    okurl=None
    for k in range(4):
        try:
            r=fal_client.subscribe("fal-ai/krea-2/turbo/lora", arguments={
                "prompt":"softfacetstyle, "+newp+ANCH,"loras":[{"path":LORA,"scale":1.0}],
                "image_size":{"width":1920,"height":1080},"num_images":1,"output_format":"jpeg","seed":33000+i*7+k})
            u=r["images"][0]["url"]
            _,c=audit({"i":i,"plate":newp,"bg":u,"tr":s["tr"]})
            print(i,"try",k,c,flush=True)
            if not (c.get("anachronism") or c.get("text_leak") or c.get("blank_faces")):
                okurl=u; break
        except Exception as e: print(i,"err",str(e)[:60],flush=True); time.sleep(4)
    if okurl:
        s["bg"]=okurl; s["plate"]=newp
        urllib.request.urlretrieve(okurl, f"{W}/assets/bg{i:02d}.jpg")
        print(i,"FIXED",flush=True)
    else: print(i,"UNRESOLVED (consider crop/re-treat)",flush=True)
json.dump(d, open(f"{W}/plan.json","w"), ensure_ascii=False, indent=1)
print("AUDITALLDONE",flush=True)
```


## 7) `props.py` — word-synced object-beat pass (run AFTER compose, BEFORE audio)

```python
# Noun literalization: prop PNGs pop in over the concatenated video at word timestamps.
import json, subprocess
from PIL import Image, ImageDraw
W="/tmp/PROJECT"
d=json.load(open(f"{W}/plan.json")); sc=d["scenes"]; words=d["words"]
def wt(si, prefix):
    import unicodedata
    n=lambda t:"".join(c for c in unicodedata.normalize("NFKD",t.lower()) if c.isalnum())
    for wo in words[sc[si]["w0"]:sc[si]["w1"]]:
        if n(wo["text"]).startswith(n(prefix)): return wo["start"]
    raise SystemExit(f"word {prefix} not in scene {si}")
import os
for f in os.listdir(f"{W}/assets"):   # trim transparent margins for accurate centering
    if f.startswith("prop_") and f.endswith(".png"):
        p=f"{W}/assets/{f}"; im=Image.open(p).convert("RGBA"); bb=im.getbbox()
        if bb and bb!=(0,0,im.width,im.height): im.crop(bb).save(p)
def stroke(p0,p1,path):                # red X as two strokes revealed sequentially
    im=Image.new("RGBA",(420,420),(0,0,0,0)); dr=ImageDraw.Draw(im)
    dr.line([p0,p1],fill=(226,54,40,255),width=64)
    for pt in (p0,p1): dr.ellipse([pt[0]-32,pt[1]-32,pt[0]+32,pt[1]+32],fill=(226,54,40,255))
    im.save(path)
stroke((60,60),(360,360),f"{W}/assets/xs1.png"); stroke((360,60),(60,360),f"{W}/assets/xs2.png")
# (png, scene, sync_word, cx, cy, size_px, extra_delay) — pick cx,cy by VIEWING each plate
B=[("prop_coin",3,"gold",960,140,210,0),
   ("prop_crown",11,"deposed",1560,215,260,0),
   ("xs1",11,"deposed",1560,215,300,0.30), ("xs2",11,"deposed",1560,215,300,0.44)]
cmd=["ffmpeg","-y","-v","error","-i",f"{W}/video_only.mp4"]
fc=[]; cur="0:v"; beats=[]
for bi,(png,si,wrd,cx,cy,sz,off) in enumerate(B):
    t0=wt(si,wrd)-0.05+off; te=sc[si]["end"]-0.08
    cmd+=["-loop","1","-i",f"{W}/assets/{png}.png"]
    iw,ih=Image.open(f"{W}/assets/{png}.png").size
    fc.append(f"[{bi+1}:v]format=rgba,split=3[p{bi}a][p{bi}b][p{bi}c]")
    ol=cur   # pop-settle: 1.35x -> 1.12x -> 1.0x via enable windows
    for j,(f_,ta,tb) in enumerate([(1.35,t0,t0+0.07),(1.12,t0+0.07,t0+0.14),(1.0,t0+0.14,te)]):
        w=int(sz*f_*iw/max(iw,ih)); h=int(sz*f_*ih/max(iw,ih)); lbl=f"b{bi}{'abc'[j]}"; nxt=f"v{bi}{j}"
        fc.append(f"[p{bi}{'abc'[j]}]scale={w}:{h}[{lbl}]")
        fc.append(f"[{ol}][{lbl}]overlay=x={cx-w//2}:y={cy-h//2}:enable='between(t,{ta:.2f},{tb:.2f})'[{nxt}]")
        ol=nxt
    cur=ol
    if off==0: beats.append({"t":round(t0+0.02,2),"kind":("thud" if "crown" in png else "pop")})
json.dump(beats, open(f"{W}/beats.json","w"))
subprocess.run(cmd+["-filter_complex",";".join(fc),"-map",f"[{cur}]","-an","-r","30",
    "-c:v","libx264","-preset","medium","-crf","19",f"{W}/video_props.mp4"],check=True)
print("PROPS PASS DONE")
```

In `audio.py`: use `video_props.mp4` when present, and mix `beats.json` pops via
`adelay` (`sfx_pop` 0.30 / `sfx_thud` 0.38 gain). Generate the two SFX once with
`fal-ai/elevenlabs/sound-effects/v2` ("short cartoon bubble pop", "short deep comedic stamp thud").
