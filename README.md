# Ludo AI MCP Server

Generate game assets using AI through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/).

## Features

| Category | Capabilities |
|----------|-------------|
| **Images** | Sprites, icons, screenshots, backgrounds, UI assets, textures, background removal |
| **3D Models** | Convert 2D images to GLB models with PBR textures, auto-rig models (skeleton + skin weights, engine-ready joint naming), text-driven skeletal animation, retarget curated animation presets onto rigged models |
| **Animation** | Animated spritesheets from static sprites (4-64 frames), keyframe animation through up to three fixed frames, motion transfer from video or presets, spritesheet editing (re-prompt, outpaint, loop fixing), sound effects for a spritesheet |
| **Video** | Generate short videos from images or reference images (5-15 seconds, with a soundtrack), prompt-driven video editing, 2x upscaling |
| **Audio** | Sound effects, background music, character voices, TTS |
| **Jobs & History** | Async job queue: submit, poll or long-poll, list and cancel jobs (queued jobs refund their credits), plus paginated generation history across the API and the web app |
| **Documentation** | Search or read Ludo's own feature guidance before generating: animation modes, model choice, margins, known limitations |

## Quick Start

### 1. Get an API Key

Sign up at [ludo.ai](https://ludo.ai) and get your API key from [app.ludo.ai](https://app.ludo.ai).

### 2. Configure Your MCP Client

#### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "ludo": {
      "url": "https://mcp.ludo.ai/mcp",
      "headers": {
        "Authorization": "ApiKey YOUR_API_KEY"
      }
    }
  }
}
```

#### Cursor

Add to your MCP settings in Cursor preferences:

```json
{
  "mcpServers": {
    "ludo": {
      "url": "https://mcp.ludo.ai/mcp",
      "headers": {
        "Authorization": "ApiKey YOUR_API_KEY"
      }
    }
  }
}
```

## Available Tools

Generation tools run on a job queue and return a job id immediately; the **Returns** field of each tool below describes the `result` you get back from `getApiJob` once the job succeeds. See [How Generation Calls Work](#how-generation-calls-work).

### Models

Sprite animation, motion transfer, spritesheet editing and video generation take an optional `model`. Pick from the current models; the defaults are what the Ludo web app uses.

| Model | Used for | Notes |
|-------|----------|-------|
| `hydra` | Sprite animation (default), keyframes (default), motion transfer, spritesheet editing | Most capable model; its spritesheets come with a generated sound effect (`audio_b64`). 3 credits/s, shortest animation 3s (9 credits minimum) |
| `forge` | Motion transfer (default), spritesheet editing (default), sprite animation, keyframes | Cost-effective; tuned for presets and matching poses. 1.5 credits/s on animation, 2 credits/s on transfer and editing, 4-credit minimum charge |
| `forge-pixel` | Same actions as `forge` | Same pricing as `forge`; for low-res pixel-art sprites |
| `griffin` | Video generation (default), video from references (default) | Fast cinematic 480p video with a soundtrack. 1.5 credits/s (2.5 credits/s from references), 5s minimum |
| `griffin-hd` | Video editing (default), video generation, video from references | Same as Griffin in 720p. 2 credits/s (4 credits/s from references), 5s minimum. Its output is too large for `upscaleVideo` |

**Legacy models** - `blitz`, `eagle`, `eagle-audio`, `tango` (and the alias `standard`) - are still accepted so existing integrations keep working, but they are scheduled for removal. Do not use them for new work. The server marks them `LEGACY` in every tool description and lists them last.


### Search Documentation (`searchDocs`)

Ask Ludo's own documentation a plain-language question and get back only the few sections that answer it: how to choose a sprite animation mode, when to use Generate Before / Generate After, how margins behave, which model suits a job, what something costs, and each generator's known limitations. This is the fastest way to learn how a feature is meant to be used before generating with it, so start here rather than reading whole documents.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `query` | Yes | What you want to know, in plain language (up to 500 characters), e.g. "how do I keep a sprite animation's colors consistent" |
| `n` | No | Maximum number of sections to return (1-10, default: 3) |

**Returns:**
- `results`: Array of `{ doc, label, section, relevance, content }`, best first. `content` is the section's full markdown; `doc` and `section` can be passed straight to `getDocs` to read it again. `relevance` is a 0-1 match score, and only results scoring 0.6 or higher are returned
- `message`: Present only when `results` is empty, which means the documentation does not cover the question. Rephrase it, or browse with `getDocs`

If the search backend is briefly unavailable the tool returns `503`; fall back to `getDocs` rather than retrying in a loop.

**Credits:** Free

---

### Feature Documentation (`getDocs`)

Browse or read Ludo's documentation in full. This is the same documentation the Ludo web app shows its users, so it occasionally describes buttons rather than parameters; the substance applies here just the same. To answer a specific question, use `searchDocs` first.

Call it with no parameters to get a table of contents, then again with `doc` (and ideally `sections`) to read only what you need. The whole corpus is roughly 280,000 characters, so fetching a full document without naming sections can return tens of thousands of characters.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `doc` | No | Document to read. Omit for the table of contents. One of `assistant`, `game-ideator`, `image-generator`, `project`, `account`, `faq`, `3d-generator`, `video-generator`, `sprite-generator`, `audio-generator`, `api-mcp`, `game-asset-generation` |
| `sections` | No | Section titles to return from `doc`, matched ignoring case, spacing and punctuation. Only valid together with `doc` |

**Returns:**
- `docs`: Array of `{ id, label, sections }`, where each section is `{ title, content }`. `content` is markdown, and is omitted from the table of contents
- `unmatched_sections` / `available_sections`: Present only when some, but not all, requested titles matched: the titles that matched nothing, and every valid title of `doc`

An unknown `doc`, or a request where no section title matches, returns `400` listing the valid values, so a wrong guess costs one extra call rather than a dead end.

The server also announces both documentation tools to MCP clients via its `instructions` field during the initialize handshake, so models connected through clients that surface server instructions are told to search the documentation before generating.

**Credits:** Free

---

### Image Generation (`createImage`)

Generate sprites, icons, backgrounds, UI assets, and textures from a text prompt alone (no source image needed). To match the art style of an existing image use `generateWithStyle`, to modify an existing image use `editImage`, and to cut out a subject use `removeBackground`.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | Detailed description of the image |
| `image_type` | Yes | `sprite`, `icon`, `screenshot`, `art`, `asset`, `sprite-vfx`, `ui_asset`, `fixed_background`, `side_scrolling_background`, `texture`, `horizontal_tile`, `tile`, `item-icon`, `portrait`, `card-art`, `splash`, `3d`, `generic` |
| `art_style` | No | `8-Bit`, `16-Bit`, `32-Bit`, `Hi-Bit`, `Low Poly`, `Stylized 3D`, `Voxel Art`, `Flat Design`, `Anime/Manga`, `Western Cartoon`, `Hand-Painted`, `Photorealistic 3D`, `Cel-Shaded`, etc. |
| `perspective` | No | `Side-Scroll`, `Top-Down`, `Isometric`, `First-Person`, `Third-Person`, `2.5D` |
| `aspect_ratio` | No | `default`, `ar_1_1`, `ar_4_3`, `ar_16_9`, `ar_9_16` |
| `n` | No | Number of variations (1-8, default: 1) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 0.5 per image

---

### Edit Image (`editImage`)

Modify an existing image using text instructions (smart editing).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image to edit |
| `prompt` | Yes | Description of changes (e.g., "remove the background", "make it darker", "add clouds to the sky") |
| `reference_image` | No | URL or base64 reference image for style/content guidance |
| `n` | No | Number of variations (1-4, default: 1) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 0.5 per image

---

### Generate with Style (`generateWithStyle`)

Generate new content while maintaining the visual style of a reference image.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `style_image` | Yes | URL or base64-encoded reference image for style matching |
| `prompt` | Yes | Description of what to generate (e.g., "a warrior character", "a treasure chest") |
| `image_type` | Yes | `sprite`, `icon`, `screenshot`, `art`, `asset`, `sprite-vfx`, `ui_asset`, `fixed_background`, `texture`, `3d`, `generic` |
| `n` | No | Number of variations (1-4, default: 1) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 0.5 per image

---

### Generate Pose (`generatePose`)

Generate a new pose for an existing sprite. **Use this BEFORE `animateSprite`** to get the best animation results - the starting pose should match your intended animation.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded sprite image |
| `pose` | Yes | Target pose: `Idle (Front)`, `Idle (Back)`, `Idle (Left Facing)`, `Idle (Right Facing)`, `Walk (Left)`, `Walk (Right)`, `Run (Left)`, `Run (Right)`, `Crouching`, `Crawling`, `Sitting`, `Attack Ready`, `Jump Preparation`, `Defending / Blocking`, `Flying`, `Sleeping`, or `Other` (with a free-text `description`) |
| `description` | No | Additional instructions to guide pose generation |
| `n` | No | Number of variations (1-4, default: 1) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `pose`, `motion_prompt`

**Example workflow:**
1. Generate a "Walk (Left)" pose with `generatePose`
2. Use the returned `motion_prompt` directly in `animateSprite` for optimal animation results

**Credits:** 0.5 per image

---

### Rotate Sprite (`rotateSprite`)

Rotate the camera view of an existing sprite to a new angle, keeping the same character and pose. Useful for generating turnaround / directional views of a sprite.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded sprite image |
| `camera_rotation` | Yes | Camera azimuth in degrees: `0` (front), `45` (front-right), `90` (right side), `135` (back-right), `180` (back), `-135` (back-left), `-90` (left side), `-45` (front-left) |
| `camera_elevation` | No | Camera tilt in degrees: `0` (eye-level), `30` (elevated), `60` (high-angle). Omit to keep the sprite's current elevation. |
| `n` | No | Number of variations (1-4, default: 1) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `camera_rotation`, `camera_elevation`

**Credits:** 0.5 per image

---

### Remove Background (`removeBackground`)

Remove the background from a single image, returning the subject isolated on a transparent background. For broader edits (not just cutting out the subject), use `editImage` instead.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image |
| `crop` | No | Trim the result to the subject's bounding box (default: false) |
| `creative_edit` | No | Higher-quality output that may not match the input pixel-for-pixel (default: true). Set false when the subject must stay pixel-identical, e.g. an existing sprite you will animate |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** a single result with `url` (transparent PNG)

**Credits:** 0.5 per image

---

### 3D Model Generation (`create3DModel`)

Convert a 2D image to a 3D GLB model with textures.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image |
| `texture_type` | No | `pbr` (default), `simple`, `none` |
| `texture_size` | No | `1024`, `2048` (default) |
| `target_num_faces` | No | Triangle count 1,000-200,000 (default: 50,000) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `model_url` (GLB file) + 4 snapshot images from different angles

**Credits:** 3 per model
**Processing time:** 60-120 seconds

---

### Rig 3D Model (`rigModel`)

Generate a skeleton and skin weights for an existing 3D model so it can be animated. Non-destructive to the geometry: it returns a new rigged GLB. Rig a model **before** using `animate3DModel`.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | URL or base64-encoded GLB to rig |
| `rig_type` | No | Skeleton style prior: `general` (default, works for any asset), `humanoid` (anime-style characters, densest skeleton), `game` (classic game-character rig), or the pinned humanoid templates with named joints (**required for `animate3DModelPreset`**): `humanoid_template` (22 joints) and `humanoid_template_hands` (52 joints, five fingers per hand). The templates only suit two-armed, two-legged characters |
| `joint_naming` | No | Bone naming convention for the identified joints: `smpl` (default), `mixamo` (Unity's humanoid auto-mapper), `humanik` (unprefixed names for Maya/MotionBuilder/FBX), `unreal` (UE mannequin), `godot` (SkeletonProfileHumanoid), `rigify` (Blender) or `vroid` (VRM). Purely a relabel, the skeleton is identical |
| `request_id` | No | Optional client-provided identifier for this request |

**Returns:** `model_url` (rigged GLB, skeleton + skin weights baked in), `rigged` (`true`)

**Credits:** 1 per rig
**Processing time:** 60-120 seconds

---

### Animate 3D Model (`animate3DModel`)

Generate text-driven skeletal animations for an **already-rigged** 3D model (rig it first with `rigModel`). Animation quality is hit-or-miss, so several candidates are returned for you to choose from. Each candidate is a standalone animation-only GLB (skeleton + one clip, **no mesh**) plus an mp4 preview; pick the best one and fuse it onto your model in a game engine or three.js.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | URL or base64-encoded **rigged** GLB to animate (rig it first with `rigModel`) |
| `prompt` | Yes | Desired motion (e.g., "walking", "swinging an axe", "waving hello") |
| `mode` | No | Animation representation: `rot_trans` (default, per-bone rotation + translation, most faithful) or `rot_only` (rotation + root translation only, for retargeting to other skeletons / engine pipelines that ignore bone translation) |
| `num_variants` | No | Number of candidate animations to generate (1-8, default: 4) |
| `loop` | No | Return to the initial pose: each clip plays forward then mirrors back to the rest pose for a seamless loop (default: true). Best for one-way motions (crouch, punch, wave); reads oddly for cyclic gaits like walking |
| `augment_prompt` | No | Rewrite the prompt into a detailed motion caption behind the scenes (default: true) |
| `request_id` | No | Optional client-provided identifier for this request |

**Returns:** `animations`, an array of candidates, each with `clip_name`, `glb_url` (animation-only GLB), `preview_url` (mp4), `mode`, `seed`, `motion`, `fit_rmse`

**Credits:** 0.2 per generation (one charge returns all variants; introductory price, 80% off the 1-credit list price)
**Processing time:** 60-120 seconds

---

### Animate 3D Model from Preset (`animate3DModelPreset`)

Apply a curated animation preset to an **already-rigged** 3D model (retargeting). Unlike `animate3DModel`, the motion comes from a professionally curated clip library instead of a text prompt, so exactly **one** clip is returned. The model must have a humanoid-template rig: rig it with `rig_type` `humanoid_template` or `humanoid_template_hands` first. Only presets that expose a `clip_url` in `listAnimationPresets` can be applied.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | URL or base64-encoded **rigged** GLB (humanoid-template rig) |
| `preset_id` | Yes | id of a preset from `listAnimationPresets`; only presets with a `clip_url` qualify |
| `crop_loop` | No | Trim the animation to the span that loops seamlessly. Omit for Auto: cyclic motions (walk, idle) loop, one-shots (jump, cast) keep their full arc |
| `in_place` | No | Remove the animation's net travel so the character moves on the spot, the treadmill clip a game engine expects, with the engine driving locomotion. Omit to follow `crop_loop` |
| `request_id` | No | Optional client-provided identifier for this request |

**Returns:** `animations`, the same envelope as `animate3DModel`, containing one clip with `clip_name`, `preset_id`, `glb_url` (animation-only GLB, joints named as on your model), `preview_url` (mp4), `seed`, `motion`, `fit_rmse`

**Credits:** 0.2 per retarget
**Processing time:** 30-90 seconds

---

### Sprite Animation (`animateSprite`)

Create animated spritesheets from static images. Use this when you can describe the motion in text; use `transferMotion` to copy an exact motion from a reference video or a named preset, and `generatePose` first when the source image is not yet in the pose the animation should start from.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `initial_image` | Yes | URL or base64 of the starting frame |
| `motion_prompt` | Yes | Animation description (e.g., "walking cycle", "idle breathing", "attack slash") |
| `image_type` | No | `sprite` (default), `sprite-vfx`, `ui_asset` |
| `frames` | No | `4`, `9`, `16`, `25`, `36` (default), `49`, `64` |
| `frame_size` | No | `0` (default, max resolution), `32`, `64`, `96`, `128`, `192`, `256`, `384`, `-1` (AI 1.5× upscale), `-9` (match input frame) |
| `loop` | No | Seamless loop (default: true) |
| `crop` | No | Crop frames to fit content (default: true); smaller spritesheets but inconsistent frame sizes across animations |
| `margin_ratio_horizontal` | No | Horizontal padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`). Useful for animations that extend sideways, e.g. sword slashes or punches |
| `margin_ratio_vertical` | No | Vertical padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`). Useful for animations that extend up or down, e.g. jumps |
| `margin_ratio` | No | *Deprecated* - uniform padding on both axes, equivalent to setting both per-axis params to the same value. Cannot be combined with the per-axis params (fails with 400) |
| `margin_ratio_mode` | No | `auto` (default), `manual`, `none` |
| `augment_prompt` | No | Augment the motion prompt behind the scenes (default: true) |
| `model` | No | `hydra` (default; most capable, generates audio), `forge` (best for basic animations and relatively simple sprites), `forge-pixel` (low-res pixel art). Legacy, avoid for new work: `blitz`, `eagle`, `eagle-audio` (alias `standard`→`blitz`) |
| `duration` | No | Default `3`s. Hydra: `3`–`5`s in 0.5 steps; Forge / Forge Pixel: `1`–`5`s in 0.5 steps. A model that does not offer the requested default falls back to its shortest |
| `final_image` | No | Ending frame for interpolation |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact, before background removal (default: false) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `spritesheet_url` (the sheet), `video_url` (an mp4 of the animation - pass it to `transferMotion` as `video` or to `createSpriteAudio` as `spritesheet_video_url`), `audio_b64` (hydra only: a generated sound effect, so there is no need to call `createSpriteAudio`), `num_frames`, `num_cols`, `num_rows`, and when requested `gif_url`, `individual_frame_urls`, `spritesheet_with_background_url`, `individual_frame_with_background_urls`

**Credits:** rate × seconds, with a per-model floor - Hydra: 3/sec (shortest 3s, so 9 minimum; the default 3s = 9); Forge and Forge Pixel: 1.5/sec with a 4-credit minimum (3s = 4.5). Legacy: Blitz 1.9/sec, Eagle 2.6/sec, Eagle with Audio 3.1/sec, all with a 4-credit minimum.
**Processing time:** 30-90 seconds

---

### Keyframe Animation (`animateSpriteKeyframes`)

Animate a sprite through up to three fixed keyframes (`initial_image`, `middle_image`, `final_image`), producing a spritesheet that interpolates through the provided frames in order. Runs on `hydra` (default), `forge` or `forge-pixel` - the models that support a middle keyframe. The motion prompt is optional here: when omitted, the motion is derived purely from the keyframes. For just a start and end frame, `animateSprite` with `final_image` does the same job.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `initial_image` | No* | URL or base64 of the first keyframe. *At least one of `initial_image` or `middle_image` is required (a `final_image` alone is rejected) |
| `middle_image` | No* | URL or base64 of the middle keyframe the animation passes through |
| `final_image` | No | URL or base64 of the final keyframe |
| `motion_prompt` | No | Optional animation description (e.g., "attack slash"); without it the keyframes drive the motion |
| `image_type` | No | `sprite` (default), `sprite-vfx`, `ui_asset` |
| `model` | No | `hydra` (default; generates audio), `forge`, or `forge-pixel` for pixel-art sprites |
| `frames` | No | `4`, `9`, `16`, `25`, `36` (default), `49`, `64` |
| `frame_size` | No | `0` (default, max resolution), `32`, `64`, `96`, `128`, `192`, `256`, `384`, `-1` (AI 1.5× upscale), `-9` (match input frame) |
| `loop` | No | Seamless loop (default: true) |
| `crop` | No | Crop frames to fit content (default: true); smaller spritesheets but inconsistent frame sizes |
| `margin_ratio_horizontal` | No | Horizontal padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`) |
| `margin_ratio_vertical` | No | Vertical padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`) |
| `margin_ratio` | No | *Deprecated* - uniform padding on both axes; cannot be combined with the per-axis params (fails with 400) |
| `margin_ratio_mode` | No | `auto` (default), `manual`, `none` |
| `augment_prompt` | No | Augment the motion prompt behind the scenes (default: true) |
| `duration` | No | Default `3`s. Hydra: `3`–`5`s; Forge / Forge Pixel: `1`–`5`s in 0.5 steps. Three keyframes need the room: at 1s the opening transition is compressed away |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact, before background removal (default: false) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite` (`spritesheet_url`, `video_url`, `audio_b64` on hydra, `gif_url`, `num_frames`, `num_cols`, `num_rows`, ...)

**Credits:** same as `animateSprite` - Hydra 3/sec (3s = 9); Forge / Forge Pixel 1.5/sec with a 4-credit minimum (1s–2.5s = 4, 3s = 4.5, 4s = 6)
**Processing time:** 30-90 seconds

---

### Animation Presets (`listAnimationPresets`)

List available animation presets, used by `transferMotion` (onto a sprite) and `animate3DModelPreset` (onto a rigged 3D model). Returns preset animations, perspectives, and directions; no video URLs are exposed.

**Returns:**
- `animations`: Array of presets with `id`, `name`, `category`, `description`, `duration`, `preview_url`, and `clip_url` on the presets that can be retargeted onto a rigged 3D model
- `perspectives`: Array with `id`, `name`, `description` (all animations support all perspectives)
- `directions`: `["N", "NE", "E", "SE", "S", "SW", "W", "NW"]` (all animations support all directions)

**Credits:** Free

---

### Motion Transfer (`transferMotion`)

Transfer motion from a video or animation preset onto a static sprite, producing an animated spritesheet. Use it when you want an exact motion copied - a standard walk or attack cycle from `listAnimationPresets`, or your own clip; use `animateSprite` to describe the motion in text instead.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded sprite image |
| `video` | No | URL of the video to use as motion source: the `video_url` of a spritesheet from `animateSprite`, or your own clip. Videos up to 4 seconds work best. Either `video` or `preset_id` + `perspective` + `direction` must be provided; when both are sent the video is used |
| `preset_id` | No | ID of an animation preset to use instead of a video URL, from `listAnimationPresets`. When using a preset, `perspective` and `direction` are required |
| `direction` | No | Facing direction for the preset, as a compass point: `N`, `NE`, `E`, `SE`, `S`, `SW`, `W`, `NW`. Required with a preset |
| `perspective` | No | Camera perspective of the preset clip, the same set for every preset: `high` (tactical, steep top-down), `horizon` (side view at eye level), `isometric` (diagonal top-down with depth), `low` (hero, low angle), `top` (directly overhead). Required with a preset |
| `frames` | No | `4`, `9`, `16`, `25`, `36` (default), `49`, `64` |
| `frame_size` | No | `0` (default, max resolution), `32`, `64`, `96`, `128`, `192`, `256`, `384` |
| `loop` | No | Trim animation for a seamless loop (default: true) |
| `crop` | No | Crop frames to fit content (default: true) |
| `margin_ratio_horizontal` | No | Horizontal padding around the sprite (0.0–1.0). Useful for animations that extend sideways, e.g. sword slashes or punches |
| `margin_ratio_vertical` | No | Vertical padding around the sprite (0.0–1.0). Useful for animations that extend up or down, e.g. jumps |
| `margin_ratio` | No | *Deprecated* - uniform padding on both axes (0.0–1.0, default 0.15 when no margin is given). Cannot be combined with the per-axis params (fails with 400) |
| `margin_ratio_mode` | No | `manual` (default), `none` |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact, before background removal (default: false) |
| `model` | No | `forge` (default; cost-effective, works best with presets and matching poses), `forge-pixel` (pixel art), `hydra` (most capable, generates audio). Legacy, avoid for new work: `tango` |
| `duration` | No | Animation length in seconds (default 1.5; on hydra, which starts at 3s, an omitted duration becomes 3). Forge / Forge Pixel: `1`–`5`s in 0.5 steps; Hydra: `3`–`5`s. A longer reference clip or preset is compressed to fit, so pass the preset's own `duration` to keep its timing |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite` (`spritesheet_url`, `video_url`, `audio_b64` on hydra, `gif_url`, `individual_frame_urls`, `spritesheet_with_background_url`, `individual_frame_with_background_urls`, `num_frames`, `num_cols`, `num_rows`)

**Credits:** rate × seconds - Forge / Forge Pixel (default): 2/sec with a 4-credit minimum (the default 1.5s = 4, 3s = 6); Hydra: 3/sec (3s = 9). Legacy Tango: 4/sec. The produced length follows the reference clip, and you are never charged for more than you requested

---

### Edit Spritesheet (`editSpritesheet`)

Edit a spritesheet you previously generated: re-prompt its animation, outpaint beyond the frame (zoom out), or repair a bad loop. Pass back the `spritesheet_url` you received from `animateSprite`, `animateSpriteKeyframes`, `transferMotion`, or an earlier edit; it must be a spritesheet you generated in the last 7 days (external URLs are not accepted). Omit `duration`, `frames`, `frame_size`, `crop` and `model` to keep the source spritesheet's values.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `spritesheet_url` | Yes | URL of a spritesheet you generated in the last 7 days |
| `edit_mode` | No | `prompt` (default), re-prompt the animation; `outpaint`, extend beyond the frame; `fix_loop`, repair a bad loop |
| `prompt` | No | Edit instruction. Required for `prompt` mode, optional for `outpaint`, not accepted for `fix_loop` |
| `images` | No | Up to 5 reference images (URL or base64) to guide the edit |
| `duration` | No | Output length in seconds (per model, as on `animateSprite`). Omit to keep the source spritesheet's duration, which is also what the flat `fix_loop` / `outpaint` rate is billed on |
| `model` | No | `forge` (default), `forge-pixel`, `hydra`. Only used by `prompt` mode; `outpaint` and `fix_loop` run a fixed pipeline and ignore it |
| `crop` | No | Crop frames to fit content. Omit to keep the source's setting |
| `loop` | No | Trim animation for seamless loop (default: true) |
| `frames` | No | Frames in the output spritesheet: `4`, `9`, `16`, `25`, `36`, `49`, `64`. Defaults to the source's frame count |
| `frame_size` | No | Frame size in pixels: `32`–`384`, or `0` for max resolution. Defaults to the source's frame size |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact (default: false) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite` (`spritesheet_url`, `video_url`, `gif_url`, `num_frames`, `num_cols`, `num_rows`, ...). Audio is generated only by `prompt` edits on hydra; a fix on a hydra sheet comes back without `audio_b64`

**Credits:** `fix_loop` and `outpaint`: flat 1 credit per second of output, no minimum, model ignored (a 3s sheet = 3). `prompt` edits: per model - Forge / Forge Pixel 2/sec with a 4-credit minimum (3s = 6), Hydra 3/sec (3s = 9)

---

### Spritesheet Sound Effect (`createSpriteAudio`)

Generate a sound effect for a spritesheet animation you previously generated. Spritesheets made with `hydra` already come with `audio_b64`, so call this for `forge` / `forge-pixel` output, or to replace hydra's sound.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `spritesheet_video_url` | Yes | The `video_url` you received from `animateSprite`, `animateSpriteKeyframes`, `transferMotion` or `editSpritesheet`; it must belong to a spritesheet you generated in the last 7 days |
| `prompt` | No | What the sound should be, e.g. "metallic sword swing with a whoosh" |
| `request_id` | No | Client-provided ID to find the job again; must not be one you already used for another generation |

**Returns:** `url` of the generated audio file. The audio is also attached to the spritesheet, so it shows up as `audio_b64` on that item in `listGenerations`

**Credits:** 3 per call

---

### Video Generation (`createVideo`)

Generate short videos from a source image and a motion prompt (image-to-video). A source `image` is required: to make a video from text alone, first `createImage` and animate that, or use `createVideoFromReferences`. Both current models generate a soundtrack, so no separate audio step is needed.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64 starting frame |
| `prompt` | Yes | Motion description (e.g., "camera zooms in", "character walks forward") |
| `duration` | No | Default `5`s. Griffin / Griffin HD: `5`–`15`s in 1s steps |
| `model` | No | `griffin` (default; 480p, with soundtrack), `griffin-hd` (720p, with soundtrack; cannot be upscaled afterwards). Legacy, avoid for new work: `blitz`, `eagle`, `eagle-audio` (alias `standard`→`blitz`) |
| `final_image` | No | Ending frame for interpolation |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `duration`, `has_audio`

**Credits:** rate × seconds - Griffin: 1.5/sec (the default 5s = 7.5); Griffin HD: 2/sec (5s = 10). Legacy: Blitz 1/sec, Eagle 1.3/sec, Eagle with Audio 1.8/sec.

---

### Video from References (`createVideoFromReferences`)

Generate a video from 1-5 reference images and a text prompt. Unlike `createVideo`, which animates a single source image, this composes a new scene that borrows characters, objects, and style from the references.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | Text description of the video to generate |
| `images` | Yes | 1 to 5 reference images (URL or base64) |
| `duration` | No | Video length in seconds: `5`–`15` (default 5) |
| `model` | No | `griffin` (default; 480p, with soundtrack), `griffin-hd` (720p). Legacy, avoid for new work: `eagle`, `eagle-audio` |
| `aspect_ratio` | No | `default` (model chooses), `ar_1_1`, `ar_16_9`, `ar_9_16`, `ar_4_3`, `ar_3_4`, `ar_21_9` |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `duration`, `has_audio`

**Credits:** rate × seconds, higher than `createVideo` for the same model - Griffin: 2.5/sec (the default 5s = 12.5); Griffin HD: 4/sec (5s = 20). Legacy: Eagle 1.5/sec, Eagle with Audio 2/sec.

---

### Edit Video (`editVideo`)

Edit a video you previously generated with a text prompt and optional reference images (video-to-video). Pass back the `url` you received from `createVideo`, `createVideoFromReferences`, or an earlier edit; it must be a video you generated in the last 7 days (external URLs are not accepted).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `video` | Yes | URL of a video you generated in the last 7 days |
| `prompt` | Yes | Edit instruction describing the desired change |
| `images` | No | Up to 5 reference images (URL or base64) to guide the edit |
| `duration` | No | Output length in seconds: `5`–`15`. Omit to keep the source video's duration |
| `model` | No | `griffin-hd` (default; 720p, with soundtrack). Legacy, avoid for new work: `eagle` |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `duration`, `has_audio`

**Credits:** Griffin HD: 2 credits/sec (5s = 10)

---

### Upscale Video (`upscaleVideo`)

Upscale a video you previously generated to twice its resolution (2x). Pass back the `url` you received from `createVideo`, `createVideoFromReferences`, or `editVideo`; it must be a video you generated in the last 7 days (external URLs are not accepted). Both dimensions of the source must be under 960 pixels: `griffin` (480p) output qualifies, `griffin-hd` (720p) output does not (landscape or portrait), so generate on `griffin` if you intend to upscale. A too-large source fails the job and the held credits are refunded.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `video` | Yes | URL of a video you generated in the last 7 days, both dimensions under 960 pixels |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url` (2x width and height, same duration), `duration`, `has_audio`

**Credits:** Flat rate by duration, independent of model: 0.2 credits/sec (5s = 1)

---

### Sound Effect Generation (`createSoundEffect`)

Generate game sound effects from text descriptions.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `description` | Yes | Sound description (e.g., "laser gun firing", "footsteps on gravel", "coin pickup") |
| `duration` | No | 0-10 seconds (0 = automatic) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 2 per sound

---

### Ambiance Generation (`createAmbiance`)

Generate ambient soundscapes and background atmospheres from text descriptions.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `description` | Yes | Ambiance description (e.g., "windy forest at dusk", "busy medieval market", "spaceship engine hum") |
| `duration` | No | Duration in seconds (0 = automatic, based on the description) |
| `loop` | No | Generate a seamlessly looping ambience (default: true) |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 2 per generation

---

### Music Generation (`createMusic`)

Generate background music and themes.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `description` | Yes | Music description (e.g., "epic orchestral battle theme", "calm piano melody", "8-bit chiptune") |
| `lyrics` | No | Optional lyrics for vocal tracks |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 3 per track

---

### Audio Transform (`createAudioTransform`)

Remix or transform an existing audio sample using a text prompt.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `sample` | Yes | URL or base64-encoded source audio sample to remix (15MB max) |
| `prompt` | Yes | Description guiding the remix (e.g., "make it sound like an 80s synthwave track") |
| `duration` | No | Duration in seconds: `0` (automatic, matches the source) or `10`–`180` in steps of 10 |
| `modification_strength` | No | How strongly the source is modified, 0–1 (default: 0.6; 0 = close to original) |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 3 per generation

---

### Voice Generation (`createVoice`)

Generate unique character voices.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `voice_description` | Yes | Character description (e.g., "gruff old warrior", "cheerful young girl") |
| `text` | Yes | Text to speak (max 200 characters) |
| `type` | No | `human` (default) or `non-human` |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 1 per voice

---

### Text-to-Speech (`createSpeech`)

Clone a voice from an audio sample.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `text` | Yes | Text to speak (max 1000 characters) |
| `sample` | Yes | URL or base64 audio sample for voice cloning |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 1 per generation

---

### Text-to-Speech Preset (`createSpeechPreset`)

Use preset voices for text-to-speech.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `text` | Yes | Text to speak (max 1000 characters) |
| `voice_preset_id` | Yes | `Serious woman`, `Wise woman`, `Calm woman`, `Patient man`, `Determined man`, `Deep voice man`, `Teen boy`, `Sweet girl`, etc. |
| `emotion` | No | `Default`, `Happy`, `Sad`, `Angry`, `Fearful`, `Surprised`, `Neutral` |
| `language` | No | `auto`, `English`, `Spanish`, `French`, `German`, `Japanese`, `Korean`, etc. |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 1 per generation

---

### Check a Job (`getApiJob`)

Poll the status of a generation job started by any tool. Every job returns `{id, status}`; call `getApiJob` with that id until `status` is `succeeded` (then read `result`, shaped exactly like the tool's documented output) or `failed` (then read `error`).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `id` | Yes | Job id returned by the generation tool |
| `wait` | No | Seconds to long-poll for a terminal state (0-60, default: 0). With `wait` the call is held open and returns the moment the job finishes |

**Returns:** `id`, `status` (`queued`, `running`, `succeeded`, `failed`, `canceled`), `result` on success, `error` (`{code, subcode, message, retriable}`) on failure, plus `poll_after_ms` on non-terminal responses (wait at least that long before polling again). Generations typically finish within a few minutes; a job still queued after 15 minutes can be cancelled and resubmitted

**Credits:** Free

---

### List Jobs (`listApiJobs`)

List the generation jobs you started through the API or MCP, most recent first. Web app jobs are not included.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `status` | No | Comma-separated statuses to include (`queued`, `running`, `succeeded`, `failed`, `canceled`). Defaults to all |
| `limit` | No | Maximum number of jobs to return (default: 50, capped at 100) |

**Returns:** Array of job objects with `id`, `status`, and timestamps

**Credits:** Free

---

### Cancel a Job (`cancelApiJob`)

Cancel a job that is still queued and get its credits refunded. Jobs that are already running cannot be canceled (the call fails with 409).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `id` | Yes | Job id to cancel |

**Returns:** The canceled job

**Credits:** Free, and the queued job's credits are refunded

---

### Generation History (`listGenerations`)

List your generation history across both the API and the Ludo web studio, with filtering, text search and pagination. This replaces the per-type results tools (`getImageResults`, `getSpriteResults`, `getVideoResults`, `getAudioResults`, `get3DModelResults`), which have been removed from the MCP and deprecated in the REST API.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `type` | Yes | `image`, `spritesheet`, `video`, `audio`, `3d` |
| `source` | No | `api` (your API/MCP generations, last 7 days only), `web` (your web studio generations, no time limit), or `all` (default) |
| `search` | No | Free-text search. Every whitespace-separated term must match the item's tags or one of its text fields (prompt, hints, style, label, ...), so "dwarf axe" narrows to items matching both |
| `request_id` | No | Only return items tagged with this `request_id` when you generated them |
| `date_from` | No | Only items generated at or after this time (unix seconds) |
| `date_to` | No | Only items generated at or before this time (unix seconds) |
| `page_size` | No | Items per page, 1-100 (default: 20) |
| `page_number` | No | 1-based page number (default: 1) |

**Returns:** `items`, `page`, `page_size`, `has_more`. Keep paging while `has_more` is true

**Credits:** Free

---

## How Generation Calls Work

Every generation runs on a job queue, and **over MCP every generation tool is asynchronous**. A generation call returns `{id, status: "queued"}` right away instead of blocking, so nothing is held open while a GPU runs and no MCP transport can time out mid-generation. You then collect the result with `getApiJob`. There is no `async` parameter on the MCP tools: it is not a knob you need to set.

(The REST API works the same way if you call it directly rather than through MCP: requests default to async and return `202` with a job id. Synchronous responses are deprecated there, but passing `async: false` in the payload still opts into them and stays supported indefinitely.)

### Collecting a result

```
# Submit the work, get a job id straight back
animateSprite with initial_image="url", motion_prompt="walking"
-> { "id": "job_abc123", "status": "queued" }

# Long-poll until it finishes (up to 60s per call), then read result
getApiJob with id="job_abc123", wait=30
-> { "id": "job_abc123", "status": "succeeded", "result": { "spritesheet_url": "...", ... } }
```

`result` is exactly the response the tool documents; on failure read `error` instead. Without `wait`, poll every few seconds and respect the `poll_after_ms` hint on non-terminal responses. Use `listApiJobs` to see what is still in flight and `cancelApiJob` to drop a job that has not started yet (its credits are refunded).

### How credits are charged

Credits are held when a job is accepted. For duration-priced tools (sprites and video) the final charge is `max(rate × produced seconds, the model's minimum charge)`, never more than for the duration you requested; the difference is refunded when the job completes. If the job fails or you cancel it while queued, everything is refunded. Each tool's **Credits** line above gives the rate and minimum; the server also states the contract once in its `instructions` field.

## Fair Use Limits

- **50 generations queued or running per account.** Submitting beyond that returns `429` with code `PENDING_JOBS_LIMIT`; wait for jobs to finish, then submit again.
- **150 requests per 5 minutes** on the read endpoints (job status, job listing, generation history, feature documentation), per API key. A `429` carries `Retry-After`.
- **60 requests per 5 minutes, and 1,000 per day,** on documentation search (`searchDocs`), per API key.

## Finding Results Later

Every generation tool also accepts an optional `request_id`. It tags the result so you can find it again afterwards, for example from a different session:

```
# Tag a generation
animateSprite with request_id="my-anim-001", initial_image="url", motion_prompt="walking"

# Later, look it up again
listGenerations with type="spritesheet", request_id="my-anim-001"
```

`listGenerations` is the general history tool: filter by `type`, by `source` (`api`, `web` or `all`), by free-text `search`, or by date, and page through the results. API-generated results are available for 7 days.

---

## Asset URL Expiration

All generated asset URLs (images, spritesheets, videos, audio, 3D models) point to Google Cloud Storage links that expire after **7 days**. Any asset that needs to outlive that window, especially assets destined for production use, must be downloaded and saved locally (or re-uploaded to permanent storage) right away. Never store the returned URLs as permanent references.

The server also announces this to MCP clients via its `instructions` field during the initialize handshake, so models connected through clients that surface server instructions will be reminded automatically.

---

## Example Prompts

### Creating Game Assets

```
Create a pixel art knight character with sword and shield, side view, 16-bit style
```

```
Generate an isometric treasure chest icon for a mobile RPG
```

```
Create a low-poly stylized tree for a casual mobile game
```

### Animations

```
Animate this character with a smooth walking cycle, 16 frames
```

```
Create an idle breathing animation for this character sprite
```

```
Rig this 3D character with a humanoid template using Mixamo joint names, then apply a walk animation preset to it
```

```
Animate this fire sprite as a looping VFX effect
```

### Audio

```
Create a satisfying coin pickup sound effect for a platformer
```

```
Generate an epic orchestral boss battle theme, intense and dramatic
```

```
Create a voice for a wise old wizard saying "The journey begins now"
```

## API Documentation

Full API documentation with all parameters and response formats: [api.ludo.ai/api-documentation](https://api.ludo.ai/api-documentation)

## Support

- [Documentation](https://api.ludo.ai/api-documentation)
- [Ludo AI Website](https://ludo.ai)
- [GitHub Issues](https://github.com/Ludo-AI/ludo-mcp/issues)

## License

Proprietary - See [ludo.ai/terms](https://ludo.ai/terms) for terms of service.