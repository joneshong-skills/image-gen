---
name: image-gen
description: >-
  This skill should be used when the user asks to "generate an image",
  "create an image", "draw a picture", "幫我生成圖片", "產生圖片",
  "用 Grok 生成圖片", "用 Gemini 生成圖片", "make me an image",
  or discusses AI image generation via Grok or Gemini web platforms.
version: 0.2.1
argument-hint: "描述想要的圖片（中文或英文皆可）"
---

# Image Gen

Generate AI images by automating Grok and Gemini web interfaces via Playwright.
Accept vague descriptions in any language, optimize the prompt via the **image-prompt** skill,
select the best platform, and return the generated image.

## Prerequisites

- MCP Playwright server connected and browser logged in to:
  - Grok (https://grok.com/) — fast creative generation
  - Gemini (https://gemini.google.com/app) — text-in-image and precision
- MCP Browser Tools server connected
- The **image-prompt** skill installed (`~/.claude/skills/image-prompt/`)

## Core Workflow

### Step 1 — Optimize the Prompt

Invoke the **image-prompt** skill with the user's description.
From its output, extract only the **Ready-to-Use Prompt (Single String)** — the "A" format.
Discard the JSON format, negative prompt, and metadata. The web platforms handle quality
enhancement internally.

If the user's input is already a detailed, professional prompt (contains 4+ of the 7 components:
subject, style, composition, lighting, color, details, atmosphere), skip prompt optimization
and use the input directly.

### Step 2 — Select Platform

#### Automatic Routing (Default)

| Signal | Platform | Reason |
|--------|----------|--------|
| No text rendering needed | **Grok** | Faster, more creative freedom |
| Human portrait / character | **Grok** | Better face generation |
| Creative / artistic / abstract | **Grok** | Aurora engine excels at artistic styles |
| Speed is priority | **Grok** | 3-5 seconds vs 10-30 seconds |
| Text in image (signs, labels, logos) | **Gemini** | Superior text rendering |
| Professional / commercial quality | **Gemini** | Higher resolution, precise adherence |
| High resolution needed (4K) | **Gemini** | Up to 4K output |
| Precise prompt adherence critical | **Gemini** | Stronger instruction following |
| Technical diagrams / infographics | **Gemini** | Better structured content |

**Default**: Grok (faster, more permissive, sufficient for most requests).

#### User Override

If the user specifies a platform, respect it unconditionally:
- Chinese: "用 Grok 幫我...", "用 Gemini 生成..."
- English: "use Grok to...", "generate with Gemini..."

### Step 3 — Generate via Browser Automation

#### Route A: Grok (Chat Mode)

Grok Chat mode is preferred over Imagine mode — fewer restrictions and shared quota with text chat.

1. `mcp__playwright__browser_navigate` → `https://grok.com/`
2. `mcp__playwright__browser_snapshot` → locate the chat input textarea
3. Check if a new chat is needed. If existing conversation is unrelated to image generation,
   look for a "New chat" or "+" button and click it first.
4. `mcp__playwright__browser_type` → ref=chat-input, text=optimized prompt
   - **Prompt prefix**: Prepend `Generate an image: ` to the optimized prompt to ensure
     Grok enters image generation mode rather than discussing the concept.
   - Do NOT use `submit=true`. After typing, snapshot to locate the submit/send button and click it.
     (`fill()` may not trigger key events; clicking the button is more reliable across UI changes.)
5. `mcp__playwright__browser_wait_for` → time=15
   (Aurora generates in 3-5s, but allow buffer for network and rendering)
6. `mcp__playwright__browser_snapshot` → check if image appeared in the response
7. If image is visible (look for `img` elements or image containers in response area):
   - `mcp__playwright__browser_take_screenshot` → capture the viewport as a preview
   - Proceed to **Step 4** to download original images
8. If generation failed (rate limit, content policy, error message):
   - Report the error to the user
   - Suggest switching to Gemini as fallback
   - Rate limit message: "Grok free tier: ~10 images per 2-hour window. Try again later or switch to Gemini."

#### Route B: Gemini (Chat Mode with Thinking)

Enable thinking mode when available for highest quality output.

1. `mcp__playwright__browser_navigate` → `https://gemini.google.com/app?hl=zh-TW`
2. `mcp__playwright__browser_snapshot` → locate the chat input area and model/mode selectors
3. Before typing the prompt, configure the generation mode:
   - Look for a model dropdown or mode selector in the top area or toolbar
   - If a "thinking" mode toggle is available, enable it
   - If an image generation option is visible, ensure it is active
   - Look for any canvas/creation mode toggle and enable image generation
4. `mcp__playwright__browser_type` → ref=chat-input, text=optimized prompt
   - **Prompt prefix**: Prepend `Create an image of: ` to the optimized prompt
   - After typing, snapshot to locate the submit/send button and click it.
5. `mcp__playwright__browser_wait_for` → time=30
   (Thinking mode can take 20-30s, standard 10-15s)
6. `mcp__playwright__browser_snapshot` → check if image appeared in the response
7. If image is visible:
   - `mcp__playwright__browser_take_screenshot` → capture the viewport as a preview
   - Proceed to **Step 4** to download original images
8. If generation failed (content policy block, error):
   - Gemini has stricter content policies than Grok
   - Report the specific error
   - Suggest Grok as fallback if the content is within Grok's policies

### Step 4 — Download Original Images

Download the original image files instead of relying on screenshots. The browser has valid
authentication that external tools (curl) lack, so use a Navigate-to-Image approach.

#### 4a. Find Image URLs

Use `mcp__playwright__browser_evaluate` to query the DOM for generated image URLs:

```js
() => {
  const imgs = document.querySelectorAll('img');
  return Array.from(imgs)
    .map(img => ({ src: img.src, width: img.naturalWidth, height: img.naturalHeight }))
    .filter(i => i.width > 500);
}
```

- Filter by `naturalWidth > 500` to exclude avatars, icons, and UI elements
- **Grok**: Look for `assets.grok.com/users/.../generated/.../image.jpg` URLs
- **Gemini**: Look for image URLs with large dimensions in the response
- Deduplicate by `src` since each image may appear twice (thumbnail + full size)

> **Note**: Do NOT use `mcp__browser-tools__getNetworkLogs` for this step. BrowserTools MCP
> monitors a separate Chrome instance and cannot see Playwright's browser traffic.

#### 4b. Navigate and Extract (per image)

For each image URL:

1. `mcp__playwright__browser_navigate` → the image URL directly
   - The browser navigates with full auth cookies, bypassing CDN restrictions
   - Page title shows dimensions (e.g., `image.jpg (960×960)`)
2. `mcp__playwright__browser_evaluate` → extract image data via canvas:
   ```js
   () => {
     const img = document.querySelector('img');
     const canvas = document.createElement('canvas');
     canvas.width = img.naturalWidth;
     canvas.height = img.naturalHeight;
     const ctx = canvas.getContext('2d');
     ctx.drawImage(img, 0, 0);
     return canvas.toDataURL('image/jpeg', 0.95).split(',')[1];
   }
   ```
   - The image is the main document, so canvas access is not blocked by CORS
   - The result (base64 string) is automatically saved to a tool-results file when it
     exceeds the token limit
3. Decode and save via Bash:
   ```bash
   cat <tool-results-file> | python3 -c "
   import json, sys, base64
   data = json.load(sys.stdin)
   text = data[0]['text']
   b64 = text.split('\"')[1]
   sys.stdout.buffer.write(base64.b64decode(b64))
   " > ~/Desktop/<filename>.jpg
   ```
4. Verify the file: check for JPEG header (`FF D8 FF`) and file size

#### 4c. Present Results

1. Use the Read tool to display the downloaded image to the user
2. Include a brief summary:
   - Platform used (Grok / Gemini)
   - The optimized prompt that was submitted
   - File paths on disk (e.g., `~/Desktop/image-name-1.jpg`)
   - Image dimensions and file size
3. Ask the user if they want:
   - Variations (re-run with modified prompt)
   - Switch platform (try the other platform with the same prompt)
   - Prompt adjustments (modify specific aspects)

#### Fallback: Screenshot Only

If the Navigate-to-Image download fails (network log unavailable, canvas blocked, etc.),
fall back to `mcp__playwright__browser_take_screenshot` and present the screenshot instead.
Inform the user that only a screenshot is available and suggest manually downloading from
the platform.

## Error Handling & Fallback

| Error | Action |
|-------|--------|
| Grok rate limit (~10/2hr) | Inform user, offer Gemini fallback |
| Gemini content policy block | Inform user, offer Grok fallback |
| Page not loaded / login required | Detect login page via snapshot, instruct user to log in manually |
| Image not appearing after wait | Extend wait by 15s, re-snapshot; if still missing, report timeout |
| Unexpected page layout | Take screenshot for debugging, report to user |
| JavaScript errors | Use `mcp__playwright__browser_console_messages` (level=error) to diagnose |
| Network failures | Use `mcp__playwright__browser_network_requests` to diagnose |

## Platform Limits Quick Reference

| | Grok (Free) | Gemini (Free) |
|---|---|---|
| Images | ~10 / 2hr | 100/day, 2/day thinking |
| Speed | 3-5s | 10-30s |

For detailed tier pricing, rate limits, content policies, and decision matrix,
see `references/platform-comparison.md`.

## Important Notes

- Both platforms require the user to be logged in via the browser that Playwright controls.
  The Playwright browser session persists — once logged in, subsequent calls reuse the session.
- Always download original images via the Navigate-to-Image approach. Use screenshots only as fallback.
- Grok supports aspect ratio control via prompt text (e.g., include "1:1", "16:9", "9:16" in the prompt).
  When no ratio is specified, the model uses `auto` mode and selects the best ratio for the prompt.
- Gemini's interface may vary by language/locale (the URL includes `hl=zh-TW` for Traditional Chinese).
- If both platforms fail, suggest using the **image-prompt** skill standalone to get an optimized
  prompt for manual use on other platforms.
- When the user asks to generate multiple images, process them one at a time to avoid rate limits.

## Additional Resources

### Reference Files
- **`references/platform-comparison.md`** — Detailed platform comparison, rate limits,
  content policies, decision matrix, and troubleshooting guide
