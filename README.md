# Ludo AI MCP Server

Generate game assets using AI through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/).

## Features

| Category | Capabilities |
|----------|-------------|
| **Images** | Sprites, icons, screenshots, backgrounds, UI assets, textures, background removal |
| **3D Models** | Convert 2D images to GLB models with PBR textures |
| **Animation** | Animated spritesheets from static sprites (4-64 frames), motion transfer from video or presets |
| **Video** | Generate short videos from images (3-10 seconds) |
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

Generate sprites, icons, backgrounds, UI assets, and textures.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | Detailed description of the image |
| `image_type` | Yes | `sprite`, `icon`, `screenshot`, `art`, `asset`, `sprite-vfx`, `ui_asset`, `fixed_background`, `texture`, `3d`, `generic` |
| `art_style` | No | `Pixel Art (16-Bit)`, `Pixel Art (8-Bit)`, `Low Poly`, `Cartoonish`, `Stylized 3D`, `Flat Design`, `Anime/Manga`, `Voxel Art`, etc. |
| `perspective` | No | `Side-Scroll`, `Top-Down`, `Isometric`, `First-Person`, `Third-Person`, `2.5D` |
| `platform` | No | `Mobile`, `Desktop`, `Web` |
| `genre` | No | `Hypercasual`, `Action`, `RPG`, `Puzzle`, `Platformer`, etc. |
| `aspect_ratio` | No | `default`, `ar_1_1`, `ar_4_3`, `ar_16_9`, `ar_9_16` |
| `n` | No | Number of variations (1-8, default: 1) |

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

**Credits:** 0.5 per image

---

### Generate Pose (`generatePose`)

Generate a new pose for an existing sprite. **Use this BEFORE `animateSprite`** to get the best animation results - the starting pose should match your intended animation.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded sprite image |
| `pose` | Yes | Target pose: `Idle (Front)`, `Idle (Back)`, `Walk / Run (Left)`, `Attack Ready`, `Jumping`, `Crouching`, `Flying`, `Defending / Blocking`, or any custom description |
| `description` | No | Additional instructions to guide pose generation |
| `n` | No | Number of variations (1-4, default: 1) |

**Returns:** `url`, `pose`, `motion_prompt`

**Example workflow:**
1. Generate a "Walk / Run (Left)" pose with `generatePose`
2. Use the returned `motion_prompt` directly in `animateSprite` for optimal animation results

**Credits:** 0.5 per image

---

### Remove Background (`removeImageBackground`)

Remove the background from an image, returning a transparent PNG.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image |

**Returns:** `url` (transparent PNG)

**Credits:** 0.5 per image

---

### 3D Model Generation (`create3DModel`)

Convert a 2D image to a 3D GLB model with textures.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64-encoded image |
| `texture_type` | No | `pbr` (default), `simple`, `none` |
| `texture_size` | No | `1024`, `2048` (default), `4096` |
| `target_num_faces` | No | Triangle count 1,000-100,000 (default: 50,000) |
| `high_detail_shape` | No | Enable for complex shapes (slower) |

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
| `frame_size` | No | `64`, `128`, `256` (default), `0` (max resolution) |
| `loop` | No | Seamless loop (default: true) |
| `model` | No | `standard` (default) or `new` (higher quality) |
| `duration` | No | Standard: 1.2-3s, New: 4s |
| `final_image` | No | Ending frame for interpolation |
| `pixel_art_filter` | No | `none`, `small`, `medium`, `large` |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |

**Returns:** `spritesheet_url`, `video_url`, `gif_url`, `individual_frame_urls`, `num_frames`, `num_cols`, `num_rows`

**Credits:** 5 per animation
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
| `video` | No | URL of the video to use as motion source (up to 4 seconds for best results). Either `video` or `preset_id` + `direction` must be provided. |
| `preset_id` | No | ID of an animation preset (from `listAnimationPresets`). Requires `direction`. |
| `direction` | No | Direction for the preset animation: `N`, `NE`, `E`, `SE`, `S`, `SW`, `W`, `NW` |
| `perspective` | No | Perspective ID for the preset. Defaults to first available. |
| `num_frames` | No | Number of frames in the output spritesheet |
| `target_frame_size` | No | Size of each frame in pixels |
| `loop` | No | Trim animation for seamless loop |
| `crop` | No | Crop frames to fit content |
| `pixel_art_filter` | No | `none`, `small`, `medium`, `large` |
| `margin_ratio` | No | Padding around sprite (0.0-1.0) |
| `margin_ratio_mode` | No | `manual` (default), `none` |
| `gif` | No | Generate an animated GIF (default: false) |
| `individual_frames` | No | Extract individual frame images (default: false) |

**Returns:** `spritesheet_url`, `video_url`, `gif_url`, `individual_frame_urls`, `num_frames`, `num_cols`, `num_rows`

**Credits:** 5 per transfer

---

### Video Generation (`createVideo`)

Generate short videos from images.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `image` | Yes | URL or base64 starting frame |
| `prompt` | Yes | Motion description (e.g., "camera zooms in", "character walks forward") |
| `duration` | No | `3`, `5` (default), `8`, `10` seconds |
| `model` | No | `standard` (default) or `new` |
| `final_image` | No | Ending frame for interpolation |

**Credits:** 3s=5, 5s=8, 8s=12, 10s=15

---

### Sound Effect Generation (`createSoundEffect`)

Generate game sound effects from text descriptions.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `description` | Yes | Sound description (e.g., "laser gun firing", "footsteps on gravel", "coin pickup") |
| `duration` | No | 0-10 seconds (0 = automatic) |

**Credits:** 3 per sound

---

### Music Generation (`createMusic`)

Generate background music and themes.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `description` | Yes | Music description (e.g., "epic orchestral battle theme", "calm piano melody", "8-bit chiptune") |
| `lyrics` | No | Optional lyrics for vocal tracks |

**Credits:** 3 per track

---

### Voice Generation (`createVoice`)

Generate unique character voices.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `voice_description` | Yes | Character description (e.g., "gruff old warrior", "cheerful young girl") |
| `text` | Yes | Text to speak (max 200 characters) |
| `type` | No | `human` (default) or `non-human` |

**Credits:** 3 per voice

---

### Text-to-Speech (`createSpeech`)

Clone a voice from an audio sample.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `text` | Yes | Text to speak (max 1000 characters) |
| `sample` | Yes | URL or base64 audio sample for voice cloning |

**Credits:** 3 per generation

---

### Text-to-Speech Preset (`createSpeechPreset`)

Use preset voices for text-to-speech.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `text` | Yes | Text to speak (max 1000 characters) |
| `voice_preset_id` | Yes | `Serious woman`, `Wise woman`, `Calm woman`, `Patient man`, `Determined man`, `Deep voice man`, `Teen boy`, `Sweet girl`, etc. |
| `emotion` | No | `Default`, `Happy`, `Sad`, `Angry`, `Fearful`, `Surprised`, `Neutral` |
| `language` | No | `auto`, `English`, `Spanish`, `French`, `German`, `Japanese`, `Korean`, etc. |

**Credits:** 3 per generation

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