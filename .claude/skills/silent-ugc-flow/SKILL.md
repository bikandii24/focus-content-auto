---
name: silent-ugc-flow
description: Clone viral TikTok UGC app-ad hooks with your own AI actor and app demo, generating on your own Google AI Pro subscription via Google Flow instead of a paid per-credit service. Use when the user wants to recreate a viral TikTok ad, build a TikTok account around one consistent AI creator, or mass-produce UGC ad variations, and wants it to run on Google Flow (Veo, Nano Banana) rather than Arcads. Runs on the google-flow MCP (browser automation over your own logged-in Google account).
---

# Silent UGC on Google Flow: TikTok hook cloning and AI actor accounts

Take a viral TikTok app ad and turn it into as many variations as you want. Keep the proven hook, put your own AI actor and app demo in it, and rotate the on-screen text. Generation runs on **Google Flow** (labs.google/flow) through the `google-flow` MCP server, driving your own logged-in Chrome and your Google AI Pro credit pool — not a per-credit third-party service.

The format you are recreating: a person reacting to something off screen, a line of text over their face, then a hard cut to a screen recording of the app. The app demo is one real recording reused across all variations.

**This is a different pipeline from an Arcads-based version of this skill, not just a swap of tool names.** Google Flow has no reference-video motion transfer (no Seedance-equivalent) and no text-overlay/stitch tools, and video generation always stops one click short of spending credits. Read "What's different from a credit-service pipeline" below before promising frame-perfect motion cloning.

## Two modes, ask which one at the start

**Clone mode** recreates one video. The user gives you a TikTok URL. Run the interview, then the pipeline once.

**Account mode** builds a TikTok account of 10 to 20 videos around one AI actor. The user gives you a TikTok video, an actor image, the app, and how many videos they want. Every scene reuses the same Flow **Character**, in a new room, outfit, and light, so a scroller reads the whole account as one person. Get the user to approve the full scene and hook list before you generate anything — video is the only step that spends real credits.

## What's different from a credit-service pipeline (read this first)

- **No motion transfer from a reference video.** Nothing in this MCP takes "here's a clip, copy its exact motion" as input. Performance cloning is done by watching the source hook frame-by-frame and writing what you saw into the video prompt as text (gaze, hand position, head movement, intensity, camera drift, timing). This gets close on a single frozen gesture with light micro-acting (the format's own sweet spot) but will drift more than a true motion-transfer model on anything with multiple beats.
- **Actor consistency is native, not a face-swap hack.** Create the actor once with `flow_create_character` (name, description, reference photo). Every later image or video references it by name via `ingredients` / `use_character` (Flow's "@" system), and Flow keeps the face consistent — no two-image face-swap step needed.
- **No text overlay or stitch tools.** Burn in on-screen text and join hook+demo with local `ffmpeg`.
- **Video never auto-generates.** `flow_generate_video` always stops at `ready_for_confirmation` — it fills the prompt and inserts the character/image references but does not click Generate (it's a paid step). Tell the user to click Generate in the visible Chrome window, or click it yourself if you have browser control, then poll with `flow_status` / `flow_queue_status` and pull the result with `flow_download_latest`.
- **Costs come out of a monthly Google AI Pro credit pool**, not a per-video invoice. Images are effectively free against that pool; video is not.
- **This is unofficial browser automation**, not a Google API. It can violate Google's Terms of Service and puts the user's own Google account at risk. Say this plainly before the first generation, don't bury it.

## Phase 1: interview

**The app.** What is it? A store link is enough. Fetch it and read the category, features, pricing, audience, and reviews. This is what drives the hook angles. Ask what the goal is: installs, trials, or awareness.

**The two assets you need.** Search the working folder first, and only ask if they are missing.
1. App demo video. A screen recording of the app in use (look for mp4 or mov files named demo, screen, app, or recording). Strip the audio. If it has a hook attached, split it and keep only the demo.
2. Actor image. The face for every video (jpg or png named model, actor, face, or reference). Frontal, well lit, at least 700px. If it is low resolution or a stock photo the user may not have rights to, say so. A generated face avoids both problems.

**The source TikTok.** The viral video whose hook you are cloning.

## Phase 2: verify the google-flow MCP and Chrome

Call `flow_connect`. Three outcomes:
- `status: connected` → note the account, continue.
- `status: oauth_required` → the user signs in to Flow once in the dedicated Chrome window, then retry `flow_connect`.
- Connection error / `BROWSER_NOT_CONNECTED` → Chrome with CDP debugging isn't running yet.

If the `mcp__google-flow__*` tools aren't available at all, walk the user through setup:

> To connect the Google Flow MCP:
> 1. Clone https://github.com/shaig-mahmudov/google-flow-mcp and run `npm install`.
> 2. Copy `config/flow.config.example.json` to `config/flow.config.json`, set `expectedAccount` to your Google email and `chromeProfile` to a Chrome profile already signed in to Flow (a dedicated profile is safest — Chrome locks the profile it's using).
> 3. Run `npm run start-browser` — this launches Chrome with remote debugging and must stay open for the whole session.
> 4. Register the server in Claude Desktop's `claude_desktop_config.json`:
>    `{"mcpServers": {"Google-Flow": {"command": "node", "args": ["/absolute/path/to/google-flow-mcp/src/index.js"]}}}`
> 5. Restart Claude Desktop, then run `npm run start-browser` again before each new session (the Chrome window has to be open before `flow_connect` works).
> Tell me when it's set up and I'll check again.

This skill handles the source video on its own. `yt-dlp` downloads it, `ffmpeg` pulls the frames, and the Read tool renders those frames directly — no separate video-watching tool needed.

## Phase 3: pick the hook and suggest the text

Propose 10 to 20 on-screen texts, grouped by angle, strongest first:

1. Price against the human alternative. "my trainer charges $80 for what this app does free"
2. Confession. "i wasted 2 years before finding this"
3. Gatekeeping. "trainers are mad this app exists"
4. Mistake callout. "you're doing X wrong and don't even know it"
5. Beginner anxiety, matched to the app's audience.
6. Lazy. "i stopped planning X, the app does it now"
7. Transformation. "day 1 vs day 60, i'm shook"

Keep each under 12 words, casual lowercase, with a curiosity gap that pairs with a reaction shot. Re-texting a video you already generated costs nothing (it's a local ffmpeg pass), so that is the cheapest way to scale a rotation.

## Phase 4: confirm before you run

Show the plan and get an explicit yes:
- How many videos, which hook goes on which scene, which source each scene is built from.
- The actor (the Flow Character you'll create or reuse).
- That video generation spends real Google Flow credits and needs a manual click to confirm each one (see cost model).
- The ToS/account-risk note above.

Do not start generating until the user confirms.

## The pipeline, per video

### 1. Download and split
TikTok's JS challenge fails now and then, so retry instead of stopping on the first error:

```bash
for i in 1 2 3 4 5; do yt-dlp -o "video.%(ext)s" "<TIKTOK_URL>" && break; sleep 3; done
```

Given a profile URL instead of a single video, list it first and pick the ad, or ask:
`yt-dlp --flat-playlist --print "%(id)s | %(view_count)s views | %(title).70s" --playlist-end 20 "<PROFILE_URL>"`

Find the boundary between the hook and the demo:
`ffmpeg -i v.mp4 -vf "select='gt(scene,0.3)',metadata=print" -an -f null -`
Take the first big scene score, then confirm it by eye. Cut the hook, mute it, and re-encode it for frame accuracy:
`ffmpeg -i v.mp4 -t <cut> -an -c:v libx264 -crf 18 -preset fast hook.mp4`
Keep the hook. The user's own demo replaces the source's demo.

### 2. Create the actor as a Flow Character (once)
`flow_create_character`: `name` (e.g. "Mia"), `description` (face, hair, general vibe — kept identical across every scene), `reference_images: [actor photo]`. This creates a reusable identity in the project; every later generation references it by name instead of redoing a face swap.

### 3. Build the scene image with Nano Banana
For a new scene, call `flow_generate_image` with `model: "Nano Banana 2"` (or `"Nano Banana Pro"` for higher fidelity), `auto_confirm: true`, `ratio: "9:16"`, `ingredients: ["<character name>"]`, and a prompt describing the room, outfit, light, and held gesture. Keep the character reference on every call — that's what keeps the face and hair identical scene to scene.

To match a specific source frame closely instead of inventing a new scene:
- Recreate the source frame with its text stripped (`flow_generate_image`, plain prompt, no character reference yet):
> Recreate this image exactly: same vertical selfie framing, same camera angle, same pose and hand position, same clothing, same lighting, same background, same amateur iPhone front-camera quality with mild sensor noise and soft compression. Remove all text overlays. Keep real skin texture, uneven lighting, no beautification.
- Pass that clean frame back in as `reference_images` on a second `flow_generate_image` call, this time with `ingredients: ["<character name>"]`, asking Flow to put the character into that exact pose/framing/background.

Images are cheap against the credit pool — open each result with vision, compare it element by element to the request, and regenerate before moving on if the face, pose, or framing drifted.

### 4. Analyze the source performance
Extract frames from `hook.mp4` (`ffmpeg -i hook.mp4 -vf fps=6 frame_%03d.png`) and read them with the Read tool. Write a short **performance profile**: where the hand starts, whether the gesture holds or transitions, peak intensity on a 1-10 scale, gaze choreography, head amplitude, camera drift, what the person never does, and the total duration. Winning hooks are usually one frozen gesture plus eye micro-acting at low intensity, not a multi-beat arc — describing an arc is what makes the output look AI-generated.

### 5. Generate the video
`flow_generate_video`: `prompt` = the performance profile above turned into an imperative description (subject, held gesture, eye/head movement, handheld camera drift, raw amateur TikTok look, no on-screen text), `model` (start with the cheap tier to test framing, e.g. `"Veo 3.1 - Lite"`, then `"Veo 3.1 - Fast"` or `"Quality"` for the final), `ratio: "9:16"`, `duration` matched to the hook's length, `ingredients: ["<character name>"]` or `use_scene` if you generated a scene image first.

This call always stops at `ready_for_confirmation` without spending credits — it does not click Generate. Tell the user to approve it in the visible Chrome window (or click it yourself if you have browser/computer-use access), then poll `flow_queue_status` and call `flow_download_latest` once it completes. Veo renders take a few minutes; do not loop tightly, wait and re-check.

### 6. Trim, text, stitch (all local ffmpeg — Flow has no tools for this)
- Trim the result to the hook's length: `ffmpeg -i flow_video.mp4 -t <hook_duration> -c copy hook_final.mp4`
- Burn in the on-screen text (classic TikTok white-with-outline look):
  `ffmpeg -i hook_final.mp4 -vf "drawtext=text='your text here':fontfile=/path/to/bold.ttf:fontsize=68:fontcolor=white:bordercolor=black:borderw=4:x=(w-text_w)/2:y=h*0.15" -c:a copy hook_texted.mp4`
  For a stacked two-line format, chain two `drawtext` filters. Re-texting later is free — keep `hook_final.mp4` (no text) so you can rotate hooks over one generation at no extra cost.
- Stitch hook + demo: `ffmpeg -i hook_texted.mp4 -i demo.mp4 -filter_complex "[0:v][1:v]concat=n=2:v=1:a=0[outv]" -map "[outv]" final.mp4`
- Deliver the mp4 and keep the intermediates in `assets/`.

## Scaling an account

One source hook gives you the performance profile; the variety comes from the scene, not frame-accurate motion cloning. Reuse the same Flow Character across every scene (step 2 above happens once). For each scene: generate a new room/outfit/light image (step 3), then generate a video against the same performance profile text (step 5) — expect more take-to-take variation than a true motion-transfer pipeline, so budget an extra regeneration or two per scene.

**Gesture library** (one per scene): hand over mouth; finger to lips; palm-forward talking; eyes closed then open; hand near mouth held; adjusting hair while glancing away; lean-in; eye-roll into a smirk.

**Scene and light library** (same face, rotate): warm dim lamp bedroom in the evening; dark bedroom with a purple-blue LED at night; bright daylight bedroom in an oversized hoodie; sunny bedroom in a colourful top; outdoor cafe or patio; kitchen counter in the morning; parked car; bathroom mirror; couch under a blanket; walking outside in winter clothes.

**Wardrobe and props:** everyday variety per scene — hoodies, tanks, casual tops, different nail colours, rings, a watch. The face and hair stay identical across every scene via the shared Character.

## Style

Every scene is an extreme close-up front-camera selfie. The face fills most of the frame, shot at arm's length or closer, with constant handheld drift and amateur framing. Keep it unpolished: real skin texture, uneven light, sensor noise, compression, no grading, no beautification.

For the on-screen text, use the white TikTok font with a black outline, two or three lines, centred mid-frame, casual lowercase. The hook runs two to four seconds, then a hard cut to the demo.

## Working with Flow media

- Reference images and character photos are passed as local file paths (`reference_images: ["/abs/path/to/photo.jpg"]`) — no presigned-upload dance, but Chrome needs read access to that path.
- Generated files download straight to disk through the authenticated Flow session; `flow_download_latest` grabs the most recent one if a call's own `files[]` result gets lost.
- The dedicated Chrome window (`npm run start-browser`) must stay open for the whole session — closing it drops the CDP connection and every `flow_*` call fails until it's relaunched.
- If Flow's UI has moved and a tool starts failing to find prompt inputs or buttons, run `flow_discover_ui` on the relevant page to refresh the selector map before retrying.
- `flow_status` and `flow_queue_status` are free — poll them instead of guessing whether a generation finished.

## Cost model (Google AI Pro monthly credit pool, not per-video billing)

| Step | Cost |
|---|---|
| `flow_create_character` | small one-time cost per actor |
| `flow_generate_image` (Nano Banana 2/Pro, Imagen 4) | effectively free against the pool |
| Veo 3.1 Lite video | ~10 credits |
| Veo 3.1 Fast video | ~20 credits |
| Veo 3.1 Quality video | ~100 credits |
| Omni Flash video | ~15-30 credits |
| Text overlay, trim, stitch (local ffmpeg) | free |
| New text on an existing video | free |

A Google AI Pro plan is usually around 1,000 credits/month. A 10-video account on Veo Fast runs roughly 200 credits — cheap relative to a per-credit service, but it's a shared monthly pool, not something you top up per video, so budget across everything else the user generates on that account too.
