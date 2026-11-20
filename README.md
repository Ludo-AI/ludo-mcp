# Ludo AI MCP Server

Generate game assets using AI through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/).

## Features

| Category | Capabilities |
|----------|-------------|
| **Images** | Sprites, icons, screenshots, backgrounds, UI assets, textures, background removal, upscaling, pixel-art conversion, normal maps |
| **3D Models** | Character images ready for a 3D model (from a prompt, or your own image redrawn as a 3D render), convert 2D images to GLB models with PBR textures (prop or the more detailed hero model, rigged in the same call if you like), auto-rig models (skeleton + skin weights, engine-ready joint naming), text-driven skeletal animation, reusable animations generated from a text prompt and kept on your account, retarget curated animation presets or your own animations onto rigged models (mirrored if you like) |
| **Directional Sprites** | Render a 3D model as styled sprites from up to 8 directions that line up, in looks from pixel art to painted to 3D render, animated with a preset, one of your generated animations or your own clip, holding fitted accessories (swords, shields, helmets...), previewed for free before you render |
| **Animation** | Animated spritesheets from static sprites (4-64 frames), keyframe animation through up to three fixed frames, motion transfer from video, presets or your generated animations, spritesheet editing (re-prompt, outpaint, loop fixing, finer-detail re-render), the same animation re-rendered from a new view, free re-export with other sheet settings, sound effects for a spritesheet |
| **Video** | Generate short videos from images or reference images (5-15 seconds on Griffin, 3-10 on Ignis, with audio), prompt-driven video editing, 2x upscaling |
| **Audio** | Sound effects, background music, character voices, TTS |
| **Jobs & History** | Async job queue: submit, poll or long-poll, list and cancel jobs (queued jobs refund their credits), plus paginated generation history across the API and the web app |
| **Account** | Credit balance, plan and auto top-up settings, to check before an expensive generation; plans, prices and what each call costs; buy extra credits with a payment page or a Link token, the user approving each payment |
| **Documentation** | Search or read Ludo's own feature guidance before generating: animation modes, model choice, margins, known limitations |
| **Feedback** | Report a limitation, bug or missing capability straight to the Ludo team |

## Quick Start

### 1. Get an API Key

API access comes with every paid plan. Create a key at [app.ludo.ai/api-keys](https://app.ludo.ai/api-keys); it is shown only once.

Using a coding agent (Claude Code, Codex, Cursor)? The [Ludo CLI](https://github.com/Ludo-AI/ludo-cli) is the better fit: the agent loads only the commands it needs, and files go in and out by path.

### 2. Configure Your MCP Client

The server speaks Streamable HTTP at `https://mcp.ludo.ai/mcp`. Every client sends the key in a header:

```
Authorization: ApiKey YOUR_API_KEY
```

Setups for eleven clients, including Windsurf, Cline, Zed and Kiro, are on [ludo.ai/developers/mcp](https://ludo.ai/developers/mcp).

#### Claude Code

```sh
claude mcp add --transport http ludo https://mcp.ludo.ai/mcp --header "Authorization: ApiKey YOUR_API_KEY"
```

Add `-s user` to use it in every project.

#### Claude Desktop and claude.ai

Open **Customize > Connectors**, click **Add**, then **Add custom connector**. Name it Ludo, use the URL `https://mcp.ludo.ai/mcp`, and under **Request headers** add `Authorization: ApiKey YOUR_API_KEY`. (`claude_desktop_config.json` only starts local servers; a remote server goes in as a connector.)

#### Codex

Add to `~/.codex/config.toml`:

```toml
[mcp_servers.ludo]
url = "https://mcp.ludo.ai/mcp"

[mcp_servers.ludo.http_headers]
Authorization = "ApiKey YOUR_API_KEY"
```

#### Cursor

Add to `~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project):

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

#### VS Code

Add to `.vscode/mcp.json`, then use Copilot Chat in Agent mode. VS Code asks for the key once and stores it securely:

```json
{
  "inputs": [
    { "type": "promptString", "id": "ludo-api-key", "description": "Ludo.ai API key", "password": true }
  ],
  "servers": {
    "ludo": {
      "type": "http",
      "url": "https://mcp.ludo.ai/mcp",
      "headers": { "Authorization": "ApiKey ${input:ludo-api-key}" }
    }
  }
}
```

#### Gemini CLI

```sh
gemini mcp add -s user --transport http --header "Authorization: ApiKey YOUR_API_KEY" ludo https://mcp.ludo.ai/mcp
```

#### Clients that only run local servers

Bridge with [`mcp-remote`](https://github.com/geelen/mcp-remote) (needs Node.js). Keep the header argument free of spaces and pass the key through `env`:

```json
{
  "mcpServers": {
    "ludo": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.ludo.ai/mcp", "--header", "Authorization:${LUDO_AUTH}"],
      "env": { "LUDO_AUTH": "ApiKey YOUR_API_KEY" }
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
| `hydra` | Sprite animation, keyframes, motion transfer and spritesheet editing (the default for all four) | Most capable all-around model; its spritesheets come with a generated sound effect (`audio_url`, an mp3). 3 credits/s, shortest animation 3s (9 credits minimum) |
| `griffin-hd` (Griffin) | Sprite animation, keyframes and spritesheet editing | Highest resolution and best image fidelity. Flat 10 credits per animation; up to 5s |
| `ignis` | Sprite animation, keyframes and motion transfer | Cheapest model, for simple body shapes and simple actions; its spritesheets come with a generated sound effect (`audio_url`). On motion transfer it takes presets only (`preset_id`): a reference `video` fails. 1.5 credits/s on all three, 1s to 5s, no minimum charge |
| `ignis-pixel` | Sprite animation, keyframes and motion transfer | Same sprite pricing as `ignis`; for low-res pixel-art sprites |
| `ignis` | Video generation (`createVideo`) | Budget videos with audio, 576p, for simple scenes and quick tests. 1 credit/s, 3s to 10s |
| `griffin-hd` (Griffin) | Video generation, video from references and video editing (the default for all three) | Cinematic video in any style with a soundtrack, 768p. 2 credits/s on all three, 5s to 15s. Landscape and portrait output is too large for `upscaleVideo` |

`forge` and `forge-pixel` are legacy since 2026-11-16 and will be retired on 2026-12-16: use `ignis` and `ignis-pixel` instead. Until then they keep running sprite animation and keyframes (1.5 credits/s), motion transfer and spritesheet editing (2 credits/s).

`griffin` (480p) was retired from video generation and video from references on 2026-10-27. Requests that name it are rejected; use `griffin-hd`.

**Legacy models** - `hydra-plus` (since 2026-11-02), `blitz`, `eagle`, `eagle-audio`, `tango` (and the alias `standard`) - are still accepted so existing integrations keep working, but they are scheduled for removal. Do not use them for new work. The server marks them `LEGACY` in every tool description and lists them last.


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
| `doc` | No | Document to read. Omit for the table of contents. One of `assistant`, `game-ideator`, `image-generator`, `project`, `account`, `faq`, `3d-generator`, `video-generator`, `sprite-generator`, `audio-generator`, `api-mcp`, `game-asset-generation`, `directional-sprites`, `asset-toolbox` |
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
| `image_type` | Yes | `sprite`, `icon`, `screenshot`, `art`, `asset`, `sprite-vfx`, `sprite-tiling-horizontal`, `sprite-tiling-vertical`, `scatter-pattern`, `logo`, `ui_asset`, `fixed_background`, `side_scrolling_background`, `vertical_scrolling_background`, `parallax_layer`, `texture`, `tile`, `item-icon`, `portrait`, `card-art`, `splash`, `3d`, `generic` |
| `art_style` | No | `Micro Pixel`, `8-Bit Pixel`, `16-Bit Pixel`, `32-Bit Pixel`, `Free Pixel`, `Low Poly`, `Stylized 3D`, `Voxel Art`, `Flat Design`, `Anime`, `Western Cartoon`, `Hand-Painted`, `Photorealistic 3D`, `Cel-Shaded`, etc. |
| `perspective` | No | `Side-Scroll`, `Top-Down`, `Isometric`, `First-Person`, `Third-Person`, `2.5D` |
| `aspect_ratio` | No | `default`, `ar_1_1`, `ar_4_3`, `ar_16_9`, `ar_19_9`, `ar_3_4`, `ar_9_16`, `ar_9_19` |
| `n` | No | Number of variations (1-8, default: 1) |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
| `save_normals` | No | Also make a normal map of each image, for 2.5D lighting, returned as `normal_map_url` (default: false). Free |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 0.5 per image

---

### Edit Image (`editImage`)

Modify an existing image using text instructions (smart editing).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image to edit |
| `prompt` | No* | Description of changes (e.g., "remove the background", "make it darker", "add clouds to the sky"). *Required, except with `mask_mode` `erase`. With a mask, describe what should fill the masked area |
| `reference_image` | No | URL or base64 reference image for style/content guidance. Not used with `mask_mode` `erase` |
| `mask` | No | URL or base64-encoded PNG that limits the edit to one area of the image: white marks the area to change, black the area to keep. It must be exactly the same width and height as `image`, or the call returns 400. Without a mask the whole image is re-rendered. A masked edit costs more (see Credits) |
| `mask_mode` | No | Only with `mask`. `replace` (default) redraws the masked area from `prompt`; `erase` removes what is in the masked area and takes no prompt |
| `image_type` | No | What kind of image the source is: `sprite`, `icon`, `screenshot`, `art`, `asset`, `sprite-vfx`, `sprite-tiling-horizontal`, `sprite-tiling-vertical`, `scatter-pattern`, `logo`, `ui_asset`, `fixed_background`, `side_scrolling_background`, `vertical_scrolling_background`, `parallax_layer`, `texture`, `tile`, `item-icon`, `portrait`, `card-art`, `splash`, `3d`, `generic`. Not needed when `image` is the URL of an image you generated on Ludo in the last 30 days, which keeps its own type. Set it for an upload or a link from elsewhere, so a sprite is edited as a sprite and keeps its transparent background |
| `n` | No | Number of variations (1-4, default: 1) |
| `save_normals` | No | Also make a normal map of each image, for 2.5D lighting, returned as `normal_map_url` (default: false). Free |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 0.5 per image; 1 per image with a mask

---

### Generate with Style (`generateWithStyle`)

Generate new content while maintaining the visual style of a reference image.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `style_image` | Yes | URL or base64-encoded reference image for style matching |
| `prompt` | Yes | Description of what to generate (e.g., "a warrior character", "a treasure chest") |
| `image_type` | No | `sprite` (default), `icon`, `screenshot`, `art`, `asset`, `sprite-vfx`, `sprite-tiling-horizontal`, `sprite-tiling-vertical`, `scatter-pattern`, `logo`, `ui_asset`, `fixed_background`, `side_scrolling_background`, `vertical_scrolling_background`, `parallax_layer`, `texture`, `tile`, `item-icon`, `portrait`, `card-art`, `splash`, `3d`, `generic` |
| `aspect_ratio` | No | `default`, `ar_1_1`, `ar_4_3`, `ar_16_9`, `ar_19_9`, `ar_3_4`, `ar_9_16`, `ar_9_19`. The shape of `style_image` is not carried over, so set this when the output needs a particular shape |
| `n` | No | Number of variations (1-4, default: 1) |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
| `save_normals` | No | Also make a normal map of each image, for 2.5D lighting, returned as `normal_map_url` (default: false). Free |
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
| `save_normals` | No | Also make a normal map of each image, for 2.5D lighting, returned as `normal_map_url` (default: false). Free |
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
| `save_normals` | No | Also make a normal map of each image, for 2.5D lighting, returned as `normal_map_url` (default: false). Free |
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

### Upscale Image (`upscaleImage`)

The same image at a higher resolution, up to 4x, with the result capped at 2048px on its longest side. The source must be below 2048px on its longest side; a larger one is rejected with HTTP 400 before anything is charged.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image, below 2048px on its longest side |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** a single result with `url`

**Credits:** 0.5 per image

### Convert to Pixel Art (`convertToPixelArt`)

Any art, broken AI pixel art or pixel art comes back as pixel art at about the input's own size. Synchronous: the call answers with the result directly, with no job to poll.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image, up to 15MB |
| `pixel_size` | No | Size of one pixel-art pixel in the input's pixels (at least 2); larger is coarser. Omit to use the image's own pixel grid, which must be detectable (otherwise HTTP 400 asks for a `pixel_size`). On an input already on a grid it snaps to whole steps of that grid |
| `palette_size` | No | At most this many colours (2-256); omit to keep the colours |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `pixel_size` (yours, or the detected one, which can be fractional), and `columns` / `rows`, the art's size in pixel-art pixels: resize to columns x rows with nearest-neighbour for one image pixel per art pixel

**Credits:** Free (20 calls per 5 minutes, 120 an hour, 1,000 a day)

### Normal Map (`createNormalMap`)

Make the normal map of an image, for 2.5D lighting: a lossless WebP of the same size and alpha as the image (transparent pixels are flat), in the OpenGL convention (red = right, green = up, blue = towards the viewer; Unity and Godot read it as is, Unreal expects green flipped). Pixel art gets one normal per art pixel, and the map of a seamless tile tiles the same way. To get maps with new images or spritesheets, pass `save_normals: true` to the generation instead.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image, opaque or a cut-out, up to 15MB. An image you generated on Ludo in the last 30 days, given by its URL, is read with its type and pixel grid; for any other image pixel art is detected |
| `request_id` | No | Optional client-provided identifier for this request |

**Returns:** `url` (the image the map was made from) and `normal_map_url`

**Credits:** Free (20 calls per 5 minutes, 120 an hour, 1,000 a day)

---

### 3D Model Generation (`create3DModel`)

Convert a 2D image to a 3D GLB model with textures, optionally rigged in the same call.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image, taken as the front view |
| `model` | No | `prop` (default): low-cost, for everyday props and environment pieces. `hero`: far more detailed and accurate, for close-up assets, but slower and more expensive |
| `back_image` | No | `prop` only: the same subject seen from behind (URL or base64), so the back of the model follows it instead of being invented. `hero` rejects it |
| `texture_type` | No | `pbr` (default), `simple`, `none` |
| `texture_size` | No | `1024`, `2048` (default) |
| `target_num_faces` | No | Triangle count 1,000-200,000 (default: 50,000) |
| `rig` | No | Also rig the model (skeleton + skin weights) in the same call (default: false). It gets the general rig, which suits any shape; there is no rig type to pick here (`rigModel` offers the others). Animation presets and Directional Sprites' `preset_id` retarget onto it, though not onto every model |
| `joint_naming` | No | With `rig`: the bone naming convention, as in `rigModel` (default: `smpl`) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `model_url` (GLB file), snapshot images from different angles, and `rigged`

**Credits:** 4 per model with `prop`, 15 with `hero`; `rig` adds 1, charged only when the model comes back rigged
**Processing time:** 60-120 seconds with `prop`; `hero` is slower

---

### Image for a 3D Model (`create3DModelImage`)

A character image ready for `create3DModel`, the first step of Directional Sprites on the web. `create3DModel` converts the image it is given as it is; this makes one for it first. Pass a prompt, your own image, or both: an image is redrawn as a 3D render, and the prompt then says what to change. A redraw is a new picture each time, so call again for another try.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | One of the two | Without an image: a description of the character (e.g. "a knight in green armour, full body, standing"). With an image: the changes to make (e.g. "add a red cape and a sword"), optional. Up to 1,000 characters |
| `image` | One of the two | URL or base64-encoded image of your own, redrawn as a 3D render |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** a single result with `url`, to pass as `create3DModel`'s `image`

**Credits:** 0.5 per image

---

### Rig 3D Model (`rigModel`)

Generate a skeleton and skin weights for an existing 3D model so it can be animated. Non-destructive to the geometry: it returns a new rigged GLB. Rig a model **before** using `animate3DModel`.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | URL or base64-encoded GLB to rig |
| `rig_type` | No | Skeleton style prior: `general` (default, works for any asset), `humanoid` (anime-style characters, densest skeleton), `game` (classic game-character rig), or the pinned humanoid templates with named joints: `humanoid_template` (22 joints) and `humanoid_template_hands` (52 joints, five fingers per hand). The templates only suit two-armed, two-legged characters. Animation presets (`animate3DModelPreset`) retarget onto any rig type, though not onto every model |
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

**Credits:** 0.5 per generation (one charge returns all variants; promotional price)
**Processing time:** 60-120 seconds

---

### Generate an Animation (`createAnimation`)

Generate a reusable animation from a text prompt (text-to-motion): the skeletal motion of a humanoid character, such as "overhead sword slash" or "zombie walk", made for no model in particular. It is saved to your account and does not expire, so generate a move once and reuse it on every character. No model or sprite is animated by this call: pass an animation's `id` as `animation_id` wherever an animation preset goes, to `animate3DModelPreset` (retarget it onto a rigged GLB, which gives the GLB), `previewDirectionalSprites` and `createDirectionalSprites` (render it on a 3D model from up to 8 directions) or `transferMotion` (onto a 2D sprite, seen from a `perspective` and a `direction`). To animate one rigged model straight from a prompt, use `animate3DModel` instead.

Look through `listAnimationPresets` first: a preset is curated and ready to use. Generate an animation when no preset covers the move.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The move, in any language; a few words are enough (e.g., "overhead sword slash", "zombie walk", "jumps and lands in a crouch"). Describe only how the body moves: props, scenery, clothing and the camera are not part of an animation. Up to 1,000 characters |
| `num_variants` | No | How many animations to generate from the prompt (1-4, default: 1), each with its own seed and each charged. Pick between them by their `preview_url` |
| `seed` | No | Seed of the first variant (the next ones count up from it). Send an animation's `seed` with the same prompt and options to generate it again |
| `duration` | No | Length in seconds (0.67-10). Omit to let it be picked from the prompt; required when `augment_prompt` is `false` |
| `loop` | No | A seamless cycle (a walk, a run, an idle), 2 to 8 seconds long. Omit to let it be picked from the prompt |
| `start_idle` / `end_idle` | No | Start / end on the relaxed standing pose, so the animation chains after / into an idle. Not with `loop`. Omit to let it be picked from the prompt |
| `left_hand` / `right_hand` | No | That hand's pose for the whole animation: `relaxed`, `fist`, `grip`, `open`, `flat`, `point`, `thumbs_up`, `pinch`, `v_sign` or `claw` (`grip` for a held weapon, `fist` for a punch). Only these poses exist; hand shapes described in the prompt are not read. Omit to let it be picked from the prompt |
| `keep_facing` | No | Keep the character facing the same way, with no turning, which fixed-direction sprites need. Omit to let it be picked from the prompt (on unless turning is the point of the move) |
| `augment_prompt` | No | Rewrite the prompt for the motion model and infer every option left out (default: true). `false` sends the prompt as written, so write it in the model's own format: one plain English sentence of under 50 words that starts with "A person" and describes only how the body moves, e.g. "A person swings a sword overhead with the right arm and steps forward on the left foot". Use `left_hand` / `right_hand` for hand shapes, and set `duration` |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `animations`, one per variant, each with `id` (pass it as `animation_id`), `name`, `prompt`, `caption` (the description the motion model was given), `seed`, `duration`, `fps`, `loop`, `start_idle`, `end_idle`, `keep_facing`, `left_hand`, `right_hand`, `preview_url` (an mp4 of the move on a mannequin, seen from the low camera facing SW; it does not expire) and `created_at`. There is no GLB yet: `animate3DModelPreset` with the `animation_id` makes one for your model

**Credits:** 0.5 per animation (one per `num_variants`). Credits are held for every variant asked; the final charge is for the animations that came back, and everything is refunded if the job fails or is cancelled

---

### My Animations (`listMyAnimations`)

List your account's own animations, newest first: the ones generated with `createAnimation` or in the Ludo web app's My Animations. (Ludo's ready-made ones are in `listAnimationPresets`.) Animations do not expire, so call this before generating a move again: one made earlier, for another character or in the web app, can be reused as it is.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `limit` | No | How many animations to return, newest first (1-200, default: 50) |
| `before` | No | Only animations created before this time (unix seconds): the `created_at` of the last animation of the previous call |

**Returns:** `animations`, each with the `id` to pass as `animation_id`, its `name`, the `prompt` it was generated from, `duration`, `loop`, `preview_url` (an mp4 of the move on a mannequin) and `created_at`, the same fields as `createAnimation` returns

**Credits:** Free

---

### Animate 3D Model from Preset (`animate3DModelPreset`)

Apply a curated animation preset, or one of your generated animations, to an **already-rigged** 3D model (retargeting). Unlike `animate3DModel`, the motion comes from a professionally curated clip library (`listAnimationPresets`) or from an animation you made with `createAnimation`, so exactly **one** clip is returned. Any rig type can take a preset or an animation (rig the model first with `rigModel`, or `create3DModel` with `rig: true`), though retargeting does not succeed on every model; when it does not, the job returns an error. An animation that loops is trimmed to its seamless loop; a one-shot keeps its full arc.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | URL or base64-encoded **rigged** GLB (any rig type) |
| `preset_id` | No | id of a preset from `listAnimationPresets`; any preset id from that list works. Exactly one of `preset_id` and `animation_id` |
| `animation_id` | No | id of one of your animations (from `createAnimation` or `listMyAnimations`), retargeted onto the model like a preset. Exactly one of `preset_id` and `animation_id`; an animation that is not on your account returns `404` |
| `mirror` | No | Play the move on the other side, left and right swapped (a slash with the left arm from one with the right). Works with a preset and with an animation (default: false) |
| `in_place` | No | Remove the animation's net travel so the character moves on the spot, the treadmill clip a game engine expects, with the engine driving locomotion. Omit to apply it to animations that loop: a cycle stays on the spot, a one-shot keeps its travel |
| `request_id` | No | Optional client-provided identifier for this request |

**Returns:** `animations`, the same envelope as `animate3DModel`, containing one clip with `clip_name`, `preset_id` (from a preset) or `animation_id` (from one of your animations), `mirror` (when mirrored), `glb_url` (animation-only GLB, joints named as on your model), `preview_url` (mp4), `seed`, `motion`, `fit_rmse`

**Credits:** 0.5 per retarget
**Processing time:** 30-90 seconds

---

### Directional Sprites

Render a 3D model as styled sprites seen from up to 8 directions (where the camera stands around a character facing S). Every direction asked for in one call is rendered together, on the same frame size, scale and floor, so the sprites line up. The usual flow:

1. *(Optional)* A character image: `create3DModelImage` from a prompt, or your own image redrawn as a 3D render.
2. A rigged model: `create3DModel` with that image and `rig: true` (or `rigModel` for a model you already have). Any rig type can take an animation preset, though retargeting does not succeed on every model.
3. *(Optional)* An animation, at most one: a `preset_id` from `listAnimationPresets`, an `animation_id` from `createAnimation` or `listMyAnimations`, or an `animation_url` made for this model by `animate3DModelPreset` or `animate3DModel`. `mirror` plays a preset or an animation on the other side. With none, the sprites are a still pose.
4. *(Optional)* Accessories: `fitDirectionalAccessory` for each item (a sword, a shield, a helmet...), adjusted until it looks right, then `attachDirectionalAccessories` for one model holding them all. Use its `model_url` as `model` from here on.
5. `listDirectionalSpriteStyles`: pick a look.
6. `previewDirectionalSprites`: check it, free.
7. `createDirectionalSprites` for animated spritesheets, or `createDirectionalStills` for still images.

#### `fitDirectionalAccessory`

Fit one item GLB to a rigged model, held by one bone: a hand, the head, the back or the hips. The item is first placed automatically for its `kind` (a sword by its handle, a gun by its pistol grip, a shield on the forearm, a helmet over the head, a backpack on the back), and the hand closes round it. Adjust it relative to that automatic fit and call again until it looks right: each call starts from the automatic fit, so send the full adjustment every time. Nothing is saved. Free and synchronous (not a job): a few seconds with previews, about a second without.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | The rigged model (`create3DModel` with `rig: true`, or `rigModel`), URL or base64 GLB, up to 30MB |
| `item` | Yes | The item, any 3D model (`create3DModel` without rig), URL or base64 GLB, up to 30MB |
| `name` | No | What the item is, in words ("rusty sword"); used to guess `kind` when it is omitted |
| `kind` | No | `blade`, `gun`, `staff`, `bow`, `shield`, `helmet`, `hat`, `back`, ...: decides how it is first held |
| `point` / `bone` | No | Where it goes: an attach point (the response lists the ones this skeleton has), or a bone by name. Default: where its kind goes |
| `size` | No | % of the automatic fit's size (100 = as fitted) |
| `move` | No | `{forward, up, out}` in % of the character's height, e.g. `{"up": 3}` |
| `turn` | No | `{forward, up, out}` in degrees |
| `flip` | No | Hold it by its other end |
| `grip` | No | Close the hand round it (skeletons with finger bones) |
| `preview` | No | `false` returns only the fit and its checks |
| `preview_preset_id` | No | The animation preset the previews show |

**Returns:** the fit as applied (`kind`, `point`, `bone`, `size`, `move`, `turn`, `flip`, `grip`), the `placement` to pass to `attachDirectionalAccessories`, `checks` (how far the item sinks into the body, how far its grip is from the palm, how much of the head pokes through a helmet), `points` (this skeleton's attach points) and, unless `preview` is false, `model_url` (the model holding the item, kept 7 days) and `previews` (4 walking frames each from SW and SE). Previews share `previewDirectionalSprites`' one-at-a-time slot; when they could not be made, `preview_error` says why and the fit is still valid.

**Credits:** Free (30 calls per 5 minutes, 200 an hour, 1,000 a day)

#### `attachDirectionalAccessories`

Put up to 8 items on the model, each at the `placement` `fitDirectionalAccessory` returned (or, without one, at the automatic fit plus the same `kind` / `point` / `bone` / `size` / `move` / `turn` / `flip` / `grip` adjustments). At most one item closes each hand: the first one listed. Free and synchronous.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | The rigged model, URL or base64 GLB, up to 30MB. The URL of one of your 3D models also saves the items on it |
| `items` | Yes | Every item the model holds: `item` (URL or base64 GLB, as sent to the fit), `name`, and `placement` or the adjustments above |
| `save_on_model` | No | `false` only builds the GLB, without saving the items on your model |
| `keep_saved` | No | When the model is one of your 3D models, keep the items it already has saved, except those held by a bone one of these items takes, and put them in the GLB too (listed in `kept`) (default: true). `false` replaces them all with these items |

**Returns:** `model_url` (one GLB of the model holding every item, kept 7 days: pass it as `model` to `previewDirectionalSprites`, `createDirectionalSprites` or `createDirectionalStills`), `saved` (whether the items were saved on your model, which replaces its previous items and shows them in the Ludo web app's Directional Sprites) and `items`

**Credits:** Free (20 calls per 5 minutes, 300 a day)

#### `listDirectionalSpriteStyles`

Free. Returns `styles` (each with `id`, `label`, `description`, `family`, `pixel`, `thumbnail_url`, `version` and its `settings` / `advanced_settings`: a `name`, `kind` of `choice`, `bool` or `color`, `options` and `default`), plus the `perspectives` and `directions` to choose from.

#### `previewDirectionalSprites`

A few frames of the model in a style, from each direction asked for, to check a look before rendering. Synchronous (not a job) and free; nothing is saved to your generations. Returns `previews`, one per direction: `frame_urls` (transparent images: 4 frames when animated, 1 for a still model), `width`, `height`, `num_frames`, `fps` and `render_3d` (the style and settings as rendered). An account runs one preview at a time: starting another cancels the one still running, which then answers with no body (HTTP 204). Takes the model, animation, camera and style parameters below, and `target_frame_size` (32-384, default 256).

#### `createDirectionalSprites`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `model` | Yes | URL or base64-encoded GLB (rigged, to animate it) |
| `animation_url` | No | An animation made for this model: the `glb_url` from `animate3DModel` or `animate3DModelPreset` (URL or base64) |
| `preset_id` | No | A preset from `listAnimationPresets`, retargeted onto the model (any rig type can take one, though retargeting does not succeed on every model, and when it does not the call returns an error) |
| `animation_id` | No | One of your animations (from `createAnimation` or `listMyAnimations`), retargeted onto the model like a preset. An animation that is not on your account returns `404` |
| `mirror` | No | With `preset_id` or `animation_id`: play the move on the other side, left and right swapped. With any other animation, or none, it returns `400` (default: false) |
| `model_animation` | No | The name of an animation the GLB itself carries |
| `perspective` | No | `horizon`, `low`, `isometric` (default), `high`, `top` |
| `directions` | No | Up to 8 of `S`, `SW`, `W`, `NW`, `N`, `NE`, `E`, `SE` (default: `["SW"]`) |
| `style_id` | No | A style from `listDirectionalSpriteStyles` (default: `render_3d`) |
| `style_params` | No | The style's settings to change, by name |
| `style_version` | No | Pin a style version, e.g. the one a previous `render_3d` reports |
| `target_frame_size` | No | Sprite size in pixels, 32-640 (default: 640) |
| `num_frames` | No | Frames per sheet, 4-100 (default: 36) |
| `loop` | No | Sample the animation as a seamless loop; omit to follow the animation |
| `keep_horizontal_travel` / `keep_vertical_travel` | No | Keep the animation's ground travel (default: off) / its vertical motion (default: on) |
| `gif` / `individual_frames` | No | Also return each sheet as a GIF / its frames as separate images |
| `save_normals` | No | Also make a normal map of each sheet, for 2.5D lighting, returned as its `normal_map_url` (default: false): the 3D model's own surface normals on the same pixels. 0.5 credits per map on top of the render; a map that could not be made is not charged |
| `request_id` | No | Client-provided ID to retrieve results later |

At most one of `animation_url`, `preset_id`, `animation_id` and `model_animation`; with none, each sheet is a single frame.

**Returns:** `sheets`, one per direction in the order asked, each with `direction`, `spritesheet_url`, `video_url`, `num_frames`, `num_cols`, `num_rows`, `fps`, `render_3d`, and `gif_url` / `individual_frame_urls` when asked

**Credits:** 1 per direction

#### `createDirectionalStills`

The model as it is, one large transparent sprite image per direction, ready for `animateSprite` or `editImage`. Takes `model`, `perspective`, `directions`, `style_id`, `style_params`, `style_version`, `save_normals` (free here) and `request_id` as above.

**Returns:** `images`, one per direction, each with `direction`, `url`, `width`, `height` and `render_3d`

**Credits:** 0.5 per direction

---

### Sprite Animation (`animateSprite`)

Create animated spritesheets from static images. Use this when you can describe the motion in text; use `transferMotion` to copy an exact motion from a reference video or a named preset, and `generatePose` first when the source image is not yet in the pose the animation should start from.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `initial_image` | Yes | URL or base64 of the starting frame |
| `motion_prompt` | Yes | Animation description (e.g., "walking", "idle breathing", "attack slash", "casting a fireball with both hands"). See [Writing the motion prompt](#writing-the-motion-prompt) |
| `image_type` | No | `sprite` (default), `sprite-vfx`, `item-icon`, `ui_asset`, `logo`, `sprite-tiling-horizontal`, `sprite-tiling-vertical`, `scatter-pattern`, `parallax_layer`, `tile`, `texture`, `portrait`, `card-art` |
| `frames` | No | `4`, `9`, `16`, `25`, `36` (default), `49`, `64` |
| `frame_size` | No | `0` (default, max resolution), `32`, `64`, `96`, `128`, `192`, `256`, `384`, `-1` (AI 1.5× upscale), `-9` (match input frame) |
| `loop` | No | Seamless loop (default: true) |
| `crop` | No | Crop frames to fit content (default: true); smaller spritesheets but inconsistent frame sizes across animations |
| `margin_ratio_horizontal` | No | Horizontal padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`). Useful for animations that extend sideways, e.g. sword slashes or punches |
| `margin_ratio_vertical` | No | Vertical padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`). Useful for animations that extend up or down, e.g. jumps |
| `margin_ratio` | No | *Deprecated* - uniform padding on both axes, equivalent to setting both per-axis params to the same value. Cannot be combined with the per-axis params (fails with 400) |
| `margin_ratio_mode` | No | `auto` (default), `manual`, `none` |
| `augment_prompt` | No | Augment the motion prompt behind the scenes (default: true) |
| `model` | No | `hydra` (default; best motion and prompt adherence, generates audio), `griffin-hd` (Griffin: highest resolution and best image fidelity), `ignis` (cheapest, for simple body shapes and simple actions; generates audio), `ignis-pixel` (low-res pixel art). Legacy, avoid for new work: `forge`, `forge-pixel`, `hydra-plus`, `blitz`, `eagle`, `eagle-audio` (alias `standard`→`blitz`) |
| `duration` | No | Default `3`s. Hydra / Hydra+: `3`–`5`s in 0.5 steps; Griffin: `5`s only; Ignis / Ignis Pixel: `1`–`5`s in 0.5 steps. A model that does not offer the requested default falls back to its shortest |
| `final_image` | No | Ending frame for interpolation |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact, before background removal (default: false) |
| `use_pixel_grid` | No | Use the animation's pixel grid when one is detected (default: true). `false` always packs as ordinary art, for when the detected grid is wrong for the asset |
| `save_normals` | No | Also make a normal map of the spritesheet, for 2.5D lighting, returned as `normal_map_url` (default: false). 0.5 credits per map on top of the generation; a map that could not be made is not charged |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `spritesheet_url` (the sheet), `video_url` (an mp4 of the animation - pass it to `transferMotion` as `video` or to `createSpriteAudio` as `spritesheet_video_url`), `audio_url` (hydra, hydra-plus and ignis: URL of a generated sound effect (mp3), so there is no need to call `createSpriteAudio`), `num_frames`, `num_cols`, `num_rows`, `duration` (seconds, to play the sheet at its real speed), and when requested `gif_url`, `individual_frame_urls`, `spritesheet_with_background_url`, `individual_frame_with_background_urls`

**Credits:** rate × seconds - Hydra: 3/sec (shortest 3s, so 9 minimum; the default 3s = 9); Ignis and Ignis Pixel: 1.5/sec, no minimum (3s = 4.5). Griffin: flat 10 per animation. Legacy: Forge and Forge Pixel 1.5/sec, no minimum; Hydra+ 4/sec (shortest 3s, so 12 minimum); Blitz 1.9/sec, Eagle 2.6/sec, Eagle with Audio 3.1/sec, all with a 4-credit minimum.
**Processing time:** 30-90 seconds

#### Writing the motion prompt

Applies to `animateSprite` and `animateSpriteKeyframes`.

- Describe exactly and unambiguously what the character should do. Specific or complex actions are fine; there is no need to simplify them.
- Don't over-describe: too much detail can harm the animation. Name an action the model already knows instead of breaking it into steps ("walking", never "move the left foot forward, then the right foot").
- Don't restate what the image or keyframes already show (the character, its equipment, the art style, background, lighting, camera). Leave frame count and timing to `frames` and `duration`.
- Negative phrasing ("no background", "do not move the camera", "without a weapon") works only on `hydra` and `hydra-plus`. On every other model state only what should happen: on the legacy models (`forge` and `forge-pixel` included) naming something you don't want makes it more likely to appear.

---

### Keyframe Animation (`animateSpriteKeyframes`)

Animate a sprite through up to three fixed keyframes (`initial_image`, `middle_image`, `final_image`), producing a spritesheet that interpolates through the provided frames in order. Runs on `hydra` (default), `griffin-hd`, `ignis` or `ignis-pixel` (and the legacy `forge`, `forge-pixel` and `hydra-plus`) - the models that support a middle keyframe. The motion prompt is optional here: when omitted, the motion is derived purely from the keyframes. For just a start and end frame, `animateSprite` with `final_image` does the same job.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `initial_image` | Yes | URL or base64 of the first keyframe; the animation starts from it |
| `middle_image` | No | URL or base64 of the middle keyframe the animation passes through |
| `final_image` | No | URL or base64 of the final keyframe |
| `motion_prompt` | No | Optional animation description (e.g., "attack slash"); without it the keyframes drive the motion. See [Writing the motion prompt](#writing-the-motion-prompt) |
| `image_type` | No | `sprite` (default), `sprite-vfx`, `item-icon`, `ui_asset`, `logo`, `sprite-tiling-horizontal`, `sprite-tiling-vertical`, `scatter-pattern`, `parallax_layer`, `tile`, `texture`, `portrait`, `card-art` |
| `model` | No | `hydra` (default; generates audio), `griffin-hd` (Griffin: highest resolution and best image fidelity), `ignis`, or `ignis-pixel` for pixel-art sprites. Legacy: `forge`, `forge-pixel`, `hydra-plus` |
| `frames` | No | `4`, `9`, `16`, `25`, `36` (default), `49`, `64` |
| `frame_size` | No | `0` (default, max resolution), `32`, `64`, `96`, `128`, `192`, `256`, `384`, `-1` (AI 1.5× upscale), `-9` (match input frame) |
| `loop` | No | Seamless loop (default: true) |
| `crop` | No | Crop frames to fit content (default: true); smaller spritesheets but inconsistent frame sizes |
| `margin_ratio_horizontal` | No | Horizontal padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`) |
| `margin_ratio_vertical` | No | Vertical padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`) |
| `margin_ratio` | No | *Deprecated* - uniform padding on both axes; cannot be combined with the per-axis params (fails with 400) |
| `margin_ratio_mode` | No | `auto` (default), `manual`, `none` |
| `augment_prompt` | No | Augment the motion prompt behind the scenes (default: true) |
| `duration` | No | Default `3`s. Hydra / Hydra+: `3`–`5`s; Griffin: `5`s only; Ignis / Ignis Pixel: `1`–`5`s in 0.5 steps. Three keyframes need the room: at 1s the opening transition is compressed away |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact, before background removal (default: false) |
| `use_pixel_grid` | No | Use the animation's pixel grid when one is detected (default: true). `false` always packs as ordinary art, for when the detected grid is wrong for the asset |
| `save_normals` | No | Also make a normal map of the spritesheet, for 2.5D lighting, returned as `normal_map_url` (default: false). 0.5 credits per map on top of the generation; a map that could not be made is not charged |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite` (`spritesheet_url`, `video_url`, `audio_url` on hydra, hydra-plus and ignis, `gif_url`, `num_frames`, `num_cols`, `num_rows`, `duration`, ...)

**Credits:** same as `animateSprite` - Hydra 3/sec (3s = 9); Griffin flat 10; Ignis / Ignis Pixel 1.5/sec, no minimum (1s = 1.5, 3s = 4.5, 4s = 6); legacy Hydra+ 4/sec (3s = 12)
**Processing time:** 30-90 seconds

---

### Animation Presets (`listAnimationPresets`)

List available animation presets, used by `transferMotion` (onto a sprite) and `animate3DModelPreset` (onto a rigged 3D model). Returns preset animations, perspectives, and directions; no video URLs are exposed. When no preset covers a move, generate one with `createAnimation`; your own animations are in `listMyAnimations`.

**Returns:**
- `animations`: Array of presets with `id`, `name`, `category`, `description`, `duration` and `preview_url`; any of them can be retargeted onto a rigged 3D model (`animate3DModelPreset`) or used as a Directional Sprites `preset_id`
- `perspectives`: Array with `id`, `name`, `description` (all animations support all perspectives)
- `directions`: `["N", "NE", "E", "SE", "S", "SW", "W", "NW"]` (all animations support all directions)

**Credits:** Free

---

### Motion Transfer (`transferMotion`)

Transfer motion from a video, an animation preset or one of your generated animations onto a static sprite, producing an animated spritesheet. Use it when you want an exact motion copied - a standard walk or attack cycle from `listAnimationPresets`, a move you made with `createAnimation`, or your own clip; use `animateSprite` to describe the motion in text instead.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded sprite image |
| `video` | No | URL of the video to use as motion source: the `video_url` of a spritesheet from `animateSprite`, or your own clip. Videos up to 4 seconds work best. One of `video`, `preset_id` and `animation_id` must be provided (the last two with `perspective` + `direction`); when a video is sent with one of them, the preset or animation is used |
| `preset_id` | No | ID of an animation preset to use instead of a video URL, from `listAnimationPresets`. When using a preset, `perspective` and `direction` are required. At most one of `preset_id` and `animation_id` |
| `animation_id` | No | One of your animations (from `createAnimation` or `listMyAnimations`) to use instead of a video or a preset. An animation can be seen from any view, so `perspective` and `direction` are required, as with a preset: pick the view the sprite is drawn from. An animation that is not on your account returns `404` |
| `mirror` | No | With `preset_id` or `animation_id`: play the move on the other side, left and right swapped (a slash with the left arm from one with the right). Not available with a `video`, where it returns `400` (default: false) |
| `direction` | No | Where the camera stands around the character for the preset or animation, as a compass point: `N`, `NE`, `E`, `SE`, `S`, `SW`, `W`, `NW`. Required with a preset or an animation |
| `perspective` | No | Camera perspective the preset or animation is seen from, the same set for every preset: `high` (tactical, steep top-down), `horizon` (side view at eye level), `isometric` (diagonal top-down with depth), `low` (hero, low angle), `top` (directly overhead). Required with a preset or an animation |
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
| `use_pixel_grid` | No | Use the animation's pixel grid when one is detected (default: true). `false` always packs as ordinary art, for when the detected grid is wrong for the asset |
| `model` | No | `hydra` (default; most capable, generates audio), `ignis` (cheapest; for simple motion, presets only - a reference `video` fails), `ignis-pixel` (pixel art). Legacy, avoid for new work: `forge`, `forge-pixel`, `tango` |
| `duration` | No | Animation length in seconds. Omitted, it is 1.5 where the model offers it and otherwise the model's shortest, so 3s on `hydra` (the default). Hydra: `3`–`5`s in 0.5 steps; Ignis / Ignis Pixel: `1`–`5`s. A longer reference clip or preset is compressed to fit, so pass the preset's own `duration` to keep its timing |
| `prompt` | No | Optional extra instructions for the motion transfer (e.g. "keep the cape still") |
| `save_normals` | No | Also make a normal map of the spritesheet, for 2.5D lighting, returned as `normal_map_url` (default: false). 0.5 credits per map on top of the generation; a map that could not be made is not charged |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite` (`spritesheet_url`, `video_url`, `audio_url` on hydra, `gif_url`, `individual_frame_urls`, `spritesheet_with_background_url`, `individual_frame_with_background_urls`, `num_frames`, `num_cols`, `num_rows`, `duration`)

**Credits:** rate × seconds - Hydra (default): 3/sec (the default 3s = 9); Ignis / Ignis Pixel: 1.5/sec, no minimum (1.5s = 2.3, 3s = 4.5). Legacy: Forge / Forge Pixel 2/sec, no minimum; Tango 4/sec with a 4-credit minimum. The produced length follows the reference clip, and you are never charged for more than you requested

---

### Edit Spritesheet (`editSpritesheet`)

Edit a spritesheet you previously generated: re-prompt its animation, outpaint beyond the frame (zoom out), repair a bad loop, or re-render a `hydra` sheet with finer detail (improve). Pass back the `spritesheet_url` you received from `animateSprite`, `animateSpriteKeyframes`, `transferMotion`, or an earlier edit; it must be a spritesheet you generated in the last 7 days (external URLs are not accepted). Omit `duration`, `frames`, `frame_size`, `crop` and `model` to keep the source spritesheet's values.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `spritesheet_url` | Yes | URL of a spritesheet you generated in the last 7 days |
| `edit_mode` | No | `prompt` (default), re-prompt the animation; `outpaint`, extend beyond the frame; `fix_loop`, repair a bad loop: it regenerates only the seam (the other frames look the same but are not bit-identical), so run it only on a spritesheet whose result had `bad_loop` true, and keep the source if the new seam is worse; `improve`, re-render with finer detail (the extra pass `hydra-plus` runs when it generates). `improve` takes only a spritesheet generated with `hydra`: any other source returns 400, including an already improved sheet or a loop fix or outpaint of one |
| `prompt` | No | Edit instruction. Required for `prompt` mode, optional for `outpaint`, not accepted for `fix_loop` or `improve` |
| `images` | No | Up to 5 reference images (URL or base64) to guide the edit |
| `duration` | No | Output length in seconds (per model, as on `animateSprite`). Omit to keep the source spritesheet's duration. It does not change the price of `fix_loop`, `outpaint` or `improve`, which bill the source spritesheet's video length |
| `model` | No | `hydra` (default), `griffin-hd` (Griffin, up to 5s). Legacy: `forge`, `forge-pixel`. `ignis` and `ignis-pixel` do not edit. Omit to keep the source spritesheet's model. Only used by `prompt` mode; `outpaint`, `fix_loop` and `improve` run a fixed pipeline and ignore it |
| `crop` | No | Crop frames to fit content. Omit to keep the source's setting |
| `loop` | No | Trim animation for seamless loop (default: true) |
| `frames` | No | Frames in the output spritesheet: `4`, `9`, `16`, `25`, `36`, `49`, `64`. Defaults to the source's frame count |
| `frame_size` | No | Frame size in pixels: `32`–`384`, or `0` for max resolution. Defaults to the source's frame size |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact (default: false) |
| `use_pixel_grid` | No | `false` always packs as ordinary art, for when the detected pixel grid is wrong for the asset. Omit to keep the source spritesheet's setting |
| `save_normals` | No | Also make a normal map of the spritesheet, for 2.5D lighting, returned as `normal_map_url` (default: false). 0.5 credits per map on top of the generation; a map that could not be made is not charged |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite` (`spritesheet_url`, `video_url`, `gif_url`, `num_frames`, `num_cols`, `num_rows`, ...). New audio is generated only by `prompt` edits on hydra; `fix_loop`, `outpaint` and `improve` keep the source spritesheet's `audio_url` when it had one

**Credits:** `fix_loop`, `outpaint` and `improve`: flat 1 credit per second of output, held for and capped at the source spritesheet's video length, no minimum, model ignored (a 3s sheet = 3, whatever `duration` you pass). `prompt` edits: per model - Hydra 3/sec (3s = 9), Griffin flat 10, legacy Forge / Forge Pixel 2/sec, no minimum (3s = 6)

### Rotate Spritesheet (`rotateSpritesheet`)

Re-render a spritesheet you previously generated from a new view: the same motion and timing, seen from a different angle (e.g. a front-facing walk cycle turned into the same walk seen from the back). Pass back the `spritesheet_url` you received from `animateSprite`, `animateSpriteKeyframes`, `transferMotion`, `editSpritesheet` or an earlier call; it must be a spritesheet you generated in the last 7 days (external URLs are not accepted). The animation covers the source's own length (up to 5 seconds; a longer source is sped up to fit), so omit `duration` unless you need to change that.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `spritesheet_url` | Yes | URL of a spritesheet you generated in the last 7 days |
| `image` | Yes | The character exactly as it must appear in the new view (URL or base64). It becomes the first frame of the animation. `rotateSprite` on the animation's first frame (or any image of the character) produces one |
| `final_image` | No | Exact last frame (URL or base64). Defaults to the first frame, for cyclic or returning animations |
| `prompt` | No | Optional instructions: the character's facing in the new view ("its back to the camera"), where the action goes ("the kick goes to the left of the frame"), or anything else about the motion |
| `duration` | No | Length in seconds: `3`-`5` in 0.5 steps. Omit to cover the source's own length |
| `model` | No | `hydra` (default). Legacy: `hydra-plus` |
| `loop` | No | Trim animation for a seamless loop. Omit to keep the source's setting |
| `crop` | No | Crop frames to fit content (default: true) |
| `frames` | No | Frames in the output spritesheet: `4`, `9`, `16`, `25`, `36`, `49`, `64`. Omit to keep the source's frame count |
| `frame_size` | No | Sprite size in pixels: `32`, `64`, `96`, `128`, `192`, `256`, `384`, or `0` for max resolution. Omit to keep the source's setting |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact (default: false) |
| `use_pixel_grid` | No | `false` always packs as ordinary art, for when the detected pixel grid is wrong for the asset. Omit to keep the source spritesheet's setting |
| `save_normals` | No | Also make a normal map of the spritesheet, for 2.5D lighting, returned as `normal_map_url` (default: false). 0.5 credits per map on top of the generation; a map that could not be made is not charged |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite` (`spritesheet_url`, `video_url`, `gif_url`, `num_frames`, `num_cols`, `num_rows`, ...)

**Credits:** rate × seconds - Hydra 3/sec (shortest 3s, so 9 minimum); legacy Hydra+ 4/sec (shortest 3s, so 12 minimum). Credits are held for the source's length, or for 3 seconds when it is shorter; the final charge is the rate × seconds of video generated, never more than was held

### Adjust Spritesheet (`adjustSpritesheet`)

Re-export a spritesheet you generated through the API in the last 7 days with different sheet settings, without generating the animation again: the same video re-cut into a new sheet. Use it to fix framing (an effect clipped by True Size: a fixed `frame_size` or 0), change the frame count or size for your engine, upscale 1.5x (`frame_size` -1), turn loop trimming on or off, or pack it as ordinary art when its detected pixel grid is wrong (`use_pixel_grid: false`). It cannot change the motion, margins or content: use `editSpritesheet` for that. Omit a setting to keep the source's value.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `spritesheet_url` | Yes | A `spritesheet_url` from `animateSprite`, `animateSpriteKeyframes`, `transferMotion`, `editSpritesheet`, `rotateSpritesheet` or an earlier adjust. External URLs are not accepted |
| `frames` | No | Frames in the new sheet |
| `frame_size` | No | Sprite size in pixels; `0` max resolution, `-1` AI 1.5x upscaling, `-9` True Size |
| `loop` | No | Trim for a seamless loop |
| `crop` | No | Crop frames to their content (ignored with True Size) |
| `use_pixel_grid` | No | `false` packs as ordinary art when the detected pixel grid is wrong for the asset |
| `gif` / `individual_frames` / `spritesheet_with_background` | No | Extra outputs, as on `animateSprite` |
| `save_normals` | No | Also make a normal map of the new spritesheet, returned as `normal_map_url` (default: false). Adjusting is free, but the map costs 0.5 credits; a map that could not be made is not charged |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** same shape as `animateSprite`, with the source's `audio_url` when it had one. The source is left as it is

**Credits:** Free (20 calls per 5 minutes, 120 an hour, 1,000 a day)

---

### Spritesheet Sound Effect (`createSpriteAudio`)

Generate a sound effect for a spritesheet animation you previously generated. Spritesheets made with `hydra` or `ignis` already come with `audio_url`, so call this for a spritesheet that has none, or to replace the generated sound.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `spritesheet_video_url` | Yes | The `video_url` you received from `animateSprite`, `animateSpriteKeyframes`, `transferMotion` or `editSpritesheet`; it must belong to a spritesheet you generated in the last 7 days |
| `prompt` | No | What the sound should be, e.g. "metallic sword swing with a whoosh" |
| `request_id` | No | Client-provided ID to find the job again; must not be one you already used for another generation |

**Returns:** `url` of the generated audio file. The audio is also attached to the spritesheet, so it shows up as `audio_url` on that item in `listGenerations`

**Credits:** 3 per call

---

### Video Generation (`createVideo`)

Generate short videos from a source image and a motion prompt (image-to-video). A source `image` is required: to make a video from text alone, first `createImage` and animate that, or use `createVideoFromReferences`. The current model generates a soundtrack, so no separate audio step is needed.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64 starting frame |
| `prompt` | Yes | Motion description (e.g., "camera zooms in", "character walks forward") |
| `duration` | No | Default `5`s. Griffin HD: `5`–`15`s in 1s steps; Ignis: `3`–`10`s in 1s steps |
| `model` | No | `griffin-hd` (default; 768p, with soundtrack), `ignis` (budget; 576p, with audio, for simple scenes). Legacy, avoid for new work: `blitz`, `eagle`, `eagle-audio` (alias `standard`→`blitz`) |
| `final_image` | No | Ending frame for interpolation |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `duration`, `has_audio`

**Credits:** rate × seconds - Griffin HD: 2/sec (the default 5s = 10); Ignis: 1/sec (3s = 3, 10s = 10). Legacy: Blitz 1/sec, Eagle 1.3/sec, Eagle with Audio 1.8/sec.

---

### Video from References (`createVideoFromReferences`)

Generate a video from 1-5 reference images and a text prompt. Unlike `createVideo`, which animates a single source image, this composes a new scene that borrows characters, objects, and style from the references.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | Text description of the video to generate |
| `images` | Yes | 1 to 5 reference images (URL or base64) |
| `duration` | No | Video length in seconds: `5`–`15` (default 5) |
| `model` | No | `griffin-hd` (default; 768p, with soundtrack). Legacy, avoid for new work: `eagle`, `eagle-audio` |
| `aspect_ratio` | No | `default` (model chooses), `ar_1_1`, `ar_16_9`, `ar_9_16`, `ar_4_3`, `ar_3_4`, `ar_21_9` |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `duration`, `has_audio`

**Credits:** rate × seconds, the same rate as `createVideo` - Griffin HD: 2/sec (the default 5s = 10). Legacy: Eagle 1.5/sec, Eagle with Audio 2/sec.

---

### Edit Video (`editVideo`)

Edit a video you previously generated with a text prompt and optional reference images (video-to-video). Pass back the `url` you received from `createVideo`, `createVideoFromReferences`, or an earlier edit; it must be a video you generated in the last 7 days (external URLs are not accepted).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `video` | Yes | URL of a video you generated in the last 7 days |
| `prompt` | Yes | Edit instruction describing the desired change |
| `images` | No | Up to 5 reference images (URL or base64) to guide the edit |
| `duration` | No | Output length in seconds: `5`–`15`. Omit to keep the source video's duration |
| `model` | No | `griffin-hd` (default; 768p, with soundtrack). Legacy, avoid for new work: `eagle` |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `url`, `duration`, `has_audio`

**Credits:** Griffin HD: 2 credits/sec (5s = 10)

---

### Upscale Video (`upscaleVideo`)

Upscale a video you previously generated to twice its resolution (2x). Pass back the `url` you received from `createVideo`, `createVideoFromReferences`, or `editVideo`; it must be a video you generated in the last 7 days (external URLs are not accepted). Both dimensions of the source must be under 960 pixels: `griffin-hd` (768p) landscape and portrait output is larger than that, so a current-model video qualifies only when it is square. A too-large source fails the job and the held credits are refunded.

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
| `loop` | No | Generate a sound effect that loops seamlessly (default: false) |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
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
| `duration` | No | Duration in seconds: `0` (default, automatic, based on the description) or `10`-`180` in steps of 10 |
| `loop` | No | Generate music that loops seamlessly (default: false) |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
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
| `loop` | No | Generate a remix that loops seamlessly (default: false) |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 3 per generation

---

### Voice Generation (`createVoice`)

Generate unique character voices.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `voice_description` | No* | Character description (e.g., "gruff old warrior", "cheerful young girl"). *Required unless `image` is given; with an image, a few words of hints |
| `image` | No | URL or base64-encoded image of the speaking character (concept art, at most 15MB). The voice is designed from it, and `voice_description` becomes optional |
| `text` | Yes | Text to speak (max 100 characters) |
| `type` | No | `human` (default) or `non-human` |
| `augment_prompt` | No | Augment the prompt behind the scenes (default: true) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 2 per voice

---

### Text-to-Speech (`createSpeech`)

Clone a voice from an audio sample.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `text` | Yes | Text to speak (max 250 characters) |
| `sample` | Yes | URL or base64 audio sample for voice cloning |
| `delivery` | No | How the line is said, e.g. "angry, shouting" or "whispering, out of breath" (max 50 characters) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** 2 per generation

---

### Text-to-Speech Preset (`createSpeechPreset`)

Use preset voices for text-to-speech.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `text` | Yes | Text to speak (max 250 characters) |
| `voice_preset_id` | Yes | `Serious woman`, `Wise woman`, `Calm woman`, `Patient man`, `Determined man`, `Deep voice man`, `Teen boy`, `Sweet girl`, etc. |
| `emotion` | No | `Default`, `Happy`, `Sad`, `Angry`, `Fearful`, `Disgusted`, `Surprised`, `Neutral`. Only for the descriptive voices (e.g. `Serious woman`); a first-name voice returns HTTP 400 for any emotion other than `Default` |
| `delivery` | No | How the line is said, e.g. "angry, shouting" or "whispering, out of breath" (max 50 characters). Only for the first-name voices (e.g. `Rachel`); a descriptive voice returns HTTP 400 - use `emotion` instead |
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

### Account (`getAccount`)

Show your credit balance, plan and auto top-up settings. Check it before an expensive generation instead of waiting for a `402`. No parameters.

**Returns:**
- `credits`:
  - `available`: what generations can spend right now (`plan_credits` + `extra` - `used`).
  - `resets_at`: when the plan credits refill (unix seconds).
  - `team_pool`: true when the balance is your team's shared pool.
  - `paused`: true while the subscription is paused.
  - `grace_until`: set when the subscription has ended; remaining credits stay spendable until then.
- `plan`: `name` and `api_access`.
- `auto_topup`: `enabled`, `threshold` and `credits`. It is null for team members, because the team owner manages it.
- `billing_url`: where to change the plan or auto top-up in the web app. Extra credits can also be bought through the API: see [Buy Extra Credits](#buy-extra-credits-gettopupquote-createtopuplink-gettopuplink-paytopup).

**Credits:** Free

### Pricing (`getPricing`)

What Ludo's paid service costs and includes, for an agent advising a user: every plan with its monthly and annual price, credits, seats and queue concurrency; credit top-ups and their price brackets; what each API call costs; and the `rules` to know before telling a user to pay (the API, MCP and CLI need an active paid plan, and there every generation costs credits). Plans are bought in the web app (`links.billing`), not through the API. Works with or without an API key. No parameters.

**Returns:** `currency`, `summary`, `plans` (cheapest first), `top_ups`, `rules`, `costs`, `links`

**Credits:** Free

### Buy Extra Credits (`getTopupQuote`, `createTopupLink`, `getTopupLink`, `payTopup`)

When the account runs out of credits, an agent can buy a top-up: extra credits at the plan's top-up price (the same as the web app's slider), which stay while the plan is active. A person approves every purchase. These tools never charge the card saved on the account; that only happens through auto top-up, which the user turns on in the web app. Offer a top-up when the credits run out, and buy one only when the user asks. On the ChatGPT connector these tools are not offered.

**`getTopupQuote`**: the price of a top-up, and the one-time Link (`link-cli`) spend request to pay it without a browser. Free, read-only.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `credits` | Yes | How many credits, in steps of `step_credits` between `min_credits` and `max_credits` |

**Returns:** `credits`, `amount` (in the currency's main unit), `amount_minor`, `currency`, `tier` (the plan whose price applies), `min_credits`, `max_credits`, `step_credits`, `network_id`, `test_mode`, `link_cli_command`

**`createTopupLink`**: a Stripe payment page for that many credits, for the user to open and pay. Show them `checkout_url`. Nothing is charged until they pay, and the credits are added the moment they do. One page is open at a time.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `credits` | Yes | How many credits |
| `replace` | No | End the account's open top-up page first and make a new one |

**Returns:** `checkout_url`, `session_id`, `credits`, `amount`, `currency`, `expires_in` (seconds; a page lasts 30 minutes)

**`getTopupLink`**: where a page from `createTopupLink` is. Free, read-only.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `session_id` | Yes | The `session_id` from `createTopupLink` |

**Returns:** `status` (`open`: not paid yet; `paid`: paid, credits on their way; `completed`: the credits are in; `expired`: never paid), `credits`, `checkout_url`, `invoice_id`

**`payTopup`**: pay with a one-time Link Shared Payment Token (`spt_...`) that the user approved in Link for exactly the amount `getTopupQuote` gave. For agents that run Link's CLI; otherwise use `createTopupLink`. A token buys one top-up. Link's agent wallet is not available in every country: see [Stripe's page on shared payment tokens](https://docs.stripe.com/agentic-commerce/concepts/shared-payment-tokens).

| Parameter | Required | Description |
|-----------|----------|-------------|
| `credits` | Yes | The same amount as the quote |
| `payment_token` | Yes | The approved token, for exactly the quoted amount |

**Returns:** `status` (`completed`: the credits are in; `paid`: the money is taken and the credits are on their way), `credits`, `amount`, `currency`, `invoice_id`, `extra_credits` (the account's top-up credits after this one)

**Credits:** Free to call; the top-up itself is paid on the page or with the token

### Send Feedback (`submitApiFeedback`)

Report a problem with the Ludo API itself to the Ludo team. Use it only for API bugs (an error, a failed or stuck job, a crash, a response that does not match the documentation) and API limitations (a parameter, operation or format you needed that the API does not support, and that you could not work around). Never use it to report model limitations or the quality of a result: an image, sprite, animation, 3D model, video or sound that looks wrong, ignores part of the prompt, has artifacts or is not what you wanted is not an API bug. Retry with a different prompt, model or settings instead. Describe what you were trying to do and what went wrong. Send one report per issue, leave out personal data, secrets and file contents, and if you are an agent, tell the user what you sent.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `category` | Yes | `bug`, `limitation`, `feature_request`, `praise` or `other`. Generated content that looks wrong is a model limitation, not a bug: do not report it |
| `message` | Yes | What you were trying to do, what happened, and what would have helped (up to 4,000 characters) |
| `operation` | No | The tool the feedback is about, e.g. `animateSprite` |
| `job_id` | No | The job it is about, when there is one. Same as a one-item `job_ids` |
| `job_ids` | No | The ids of the jobs it is about (up to 10): every job that shows the problem, plus the one that worked if you are comparing |
| `agent` | No | The agent or app sending it, e.g. `Claude Code` |

**Returns:** `ok` and the report's `id`

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
- **150 requests per 5 minutes** on the read endpoints (job status, job listing, generation history, account, feature documentation), per API key. A `429` carries `Retry-After`.
- **60 requests per 5 minutes, and 1,000 per day,** on documentation search (`searchDocs`), per API key.
- **The free synchronous tools** have their own limits, given with each one above: `convertToPixelArt`, `adjustSpritesheet`, `fitDirectionalAccessory` and `attachDirectionalAccessories`.

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
Rig this 3D character using Mixamo joint names, then apply a walk animation preset to it
```

```
Generate an overhead sword slash animation, apply it to my rigged knight, and render it as directional sprites from SW and SE
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