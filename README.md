# Ludo AI MCP Server

Generate game assets using AI through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/).

## Features

| Category | Capabilities |
|----------|-------------|
| **Images** | Sprites, icons, screenshots, backgrounds, UI assets, textures, background removal |
| **3D Models** | Convert 2D images to GLB models with PBR textures |
| **Animation** | Animated spritesheets from static sprites (4-64 frames), motion transfer from video or presets |
| **Video** | Generate short videos from images (1-15 seconds, varies by model) |
| **Audio** | Sound effects, background music, character voices, TTS |

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
| `creative_edit` | No | Higher-quality output that may not match the input pixel-for-pixel (default: true) |
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

### Sprite Animation (`animateSprite`)

Create animated spritesheets from static images.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `initial_image` | Yes | URL or base64 of the starting frame |
| `motion_prompt` | Yes | Animation description (e.g., "walking cycle", "idle breathing", "attack slash") |
| `image_type` | No | `sprite`, `sprite-vfx`, `ui_asset` |
| `frames` | No | `4`, `9`, `16`, `25`, `36` (default), `49`, `64` |
| `frame_size` | No | `32`, `64`, `96`, `128`, `192`, `256` (default), `384`, `0` (max resolution), `-1` (AI 1.5× upscale), `-9` (match input frame) |
| `loop` | No | Seamless loop (default: true) |
| `crop` | No | Crop frames to fit content; smaller spritesheets but inconsistent frame sizes |
| `margin_ratio` | No | Padding around the sprite as a ratio 0.0–1.0 (only used when `margin_ratio_mode` is `manual`) |
| `margin_ratio_mode` | No | `auto` (default), `manual`, `none` |
| `augment_prompt` | No | Augment the motion prompt behind the scenes (default: true) |
| `model` | No | `blitz` (default), `eagle`, `eagle-audio`, `chaos`. Legacy aliases: `standard`→`blitz`, `new`→`chaos` |
| `duration` | No | Depends on model — Blitz: `1.2`–`4`s (1.2, 1.5, 2, 2.5, 3, 3.5, 4); Eagle / Eagle with Audio: `1`–`4`s; Chaos: `4`s |
| `final_image` | No | Ending frame for interpolation |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact, before background removal (default: false) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `spritesheet_url`, `video_url`, `gif_url`, `individual_frame_urls`, `spritesheet_with_background_url`, `individual_frame_with_background_urls`, `num_frames`, `num_cols`, `num_rows`

**Credits:** Varies by duration and model. Standard model: 1.9 credits/sec with a 4-credit minimum (e.g. ~1.2s = 4, 3s ≈ 5.7, 4s ≈ 7.6). Higher-quality models cost more per second.
**Processing time:** 30-90 seconds

---

### Animation Presets (`listAnimationPresets`)

List available animation presets for use with motion transfer. Returns preset animations, perspectives, and directions — no video URLs are exposed.

**Returns:**
- `animations` — Array of presets with `id`, `name`, `category`, `description`, `duration`, `preview_url`
- `perspectives` — Array with `id`, `name`, `description` (all animations support all perspectives)
- `directions` — `["N", "NE", "E", "SE", "S", "SW", "W", "NW"]` (all animations support all directions)

**Credits:** Free

---

### Motion Transfer (`transferMotion`)

Transfer motion from a video or animation preset onto a static sprite, producing an animated spritesheet.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded sprite image |
| `video` | No | URL of the video to use as motion source. You can use videos from the animateSprite endpoint or provide your own. Videos up to 4 seconds will produce better results. Either `video` or `preset_id` + `perspective` + `direction` must be provided. |
| `preset_id` | No | ID of an animation preset to use instead of a video URL. Use the animation-presets endpoint to list available presets. When using a preset, `perspective` and `direction` are required. |
| `direction` | No | Direction for the animation preset. When using a preset, `direction` is required. Values: `N`, `NE`, `E`, `SE`, `S`, `SW`, `W`, `NW` |
| `perspective` | No | Perspective ID to use with the animation preset. When using a preset, `perspective` is required. |
| `frames` | No | Number of frames in the output spritesheet |
| `frame_size` | No | Size of each frame in pixels |
| `loop` | No | Trim animation for seamless loop |
| `crop` | No | Crop frames to fit content |
| `margin_ratio` | No | Padding around sprite (0.0-1.0) |
| `margin_ratio_mode` | No | `manual` (default), `none` |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |
| `spritesheet_with_background` | No | Also return the spritesheet with background intact, before background removal (default: false) |
| `request_id` | No | Client-provided ID to retrieve results later |

**Returns:** `spritesheet_url`, `video_url`, `gif_url`, `individual_frame_urls`, `spritesheet_with_background_url`, `individual_frame_with_background_urls`, `num_frames`, `num_cols`, `num_rows`

**Credits:** Varies by duration: 4 credits/sec with a 4-credit minimum (default 1.5s = 6, up to 16 at 4s)

---

### Video Generation (`createVideo`)

Generate short videos from images.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64 starting frame |
| `prompt` | Yes | Motion description (e.g., "camera zooms in", "character walks forward") |
| `duration` | No | Depends on model (defaults to the model's shortest) — Blitz: `2`–`12`s; Eagle / Eagle with Audio: `1`–`15`s; Chaos: `4`–`12`s |
| `model` | No | `blitz` (default), `eagle`, `eagle-audio`, `chaos`. Legacy aliases: `standard`→`blitz`, `new`→`chaos` |
| `final_image` | No | Ending frame for interpolation |
| `request_id` | No | Client-provided ID to retrieve results later |

**Credits:** Varies by duration and model. Standard model: 1 credit/sec (3s = 3, 5s = 5, 8s = 8, 10s = 10). Higher-quality models cost more per second.

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
| `modification_strength` | No | How strongly the source is modified (0 = close to original) |
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

### Retrieve Image Results (`getImageResults`)

Retrieve your recent API-generated images.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `request_id` | No | Filter to a specific request |

**Returns:** Array of image objects with `url`, `request_id`, `created_at`

**Credits:** Free

---

### Retrieve Sprite Results (`getSpriteResults`)

Retrieve your recent API-generated spritesheets.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `request_id` | No | Filter to a specific request |

**Returns:** Array of spritesheet objects with `request_id`, `created_at`

**Credits:** Free

---

### Retrieve Video Results (`getVideoResults`)

Retrieve your recent API-generated videos.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `request_id` | No | Filter to a specific request |

**Returns:** Array of video objects with `url`, `request_id`, `created_at`

**Credits:** Free

---

### Retrieve Audio Results (`getAudioResults`)

Retrieve your recent API-generated audio.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `request_id` | No | Filter to a specific request |

**Returns:** Array of audio objects with `request_id`, `created_at`

**Credits:** Free

---

### Retrieve 3D Model Results (`get3DModelResults`)

Retrieve your recent API-generated 3D models.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `request_id` | No | Filter to a specific request |

**Returns:** Array of 3D asset objects with `request_id`, `created_at`

**Credits:** Free

---

## How Generation Calls Work

Every generation tool is **synchronous**: the call blocks until the asset is ready and returns the result (with its URLs) directly in the response. There is no separate polling step, and dropping the connection before the call returns cancels the request. Long-running operations such as 3D models and animations can take 30–120 seconds, so allow a generous client-side timeout.

## Retrieving Results Later

Every generation tool also accepts an optional `request_id`. This does **not** make the call asynchronous — it tags the result so you can fetch it again afterwards from the matching results endpoint (for example from a different session, or to list everything tied to one request). The result is persisted only after the synchronous call has completed successfully.

```
# Generate a sprite animation — this call blocks and returns the spritesheet directly
animateSprite with request_id="my-anim-001", initial_image="url", motion_prompt="walking"

# Later, re-fetch that same result (e.g. in another session) by its request_id
getSpriteResults with request_id="my-anim-001"
```

Results are available for 7 days, and each results endpoint is free and returns up to 100 recent API-generated assets.

---

## Asset URL Expiration

All generated asset URLs (images, spritesheets, videos, audio, 3D models) are temporary and expire after **7 days**. If you need to keep an asset, download it before it expires.

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