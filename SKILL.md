---
name: image-gen
description: >-
  This skill should be used when the user asks to "generate an image",
  "create an image", "draw a picture", "幫我生成圖片", "產生圖片",
  "用 Grok 生成圖片", "用 Gemini 生成圖片", "make me an image",
  or discusses AI image generation via Grok or Gemini web platforms.
version: 0.1.0
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
4. `mcp__playwright__browser_type` → ref=chat-input, text=optimized prompt, submit=true
   - **Prompt prefix**: Prepend `Generate an image: ` to the optimized prompt to ensure
     Grok enters image generation mode rather than discussing the concept.
5. `mcp__playwright__browser_wait_for` → time=15
   (Aurora generates in 3-5s, but allow buffer for network and rendering)
6. `mcp__playwright__browser_snapshot` → check if image appeared in the response
7. If image is visible (look for `img` elements or image containers in response area):
   - `mcp__playwright__browser_take_screenshot` → capture the response area with the image
   - Parse the snapshot for download links if available
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
4. `mcp__playwright__browser_type` → ref=chat-input, text=optimized prompt, submit=true
   - **Prompt prefix**: Prepend `Create an image of: ` to the optimized prompt
5. `mcp__playwright__browser_wait_for` → time=30
   (Thinking mode can take 20-30s, standard 10-15s)
6. `mcp__playwright__browser_snapshot` → check if image appeared in the response
7. If image is visible:
   - `mcp__playwright__browser_take_screenshot` → capture the response with the image
   - Look for download or expand buttons in the snapshot
8. If generation failed (content policy block, error):
   - Gemini has stricter content policies than Grok
   - Report the specific error
   - Suggest Grok as fallback if the content is within Grok's policies

### Step 4 — Return Results

After successful generation:

1. Take a screenshot of the generated image area using `mcp__playwright__browser_take_screenshot`
2. Present the screenshot to the user
3. Include a brief summary:
   - Platform used (Grok / Gemini)
   - The optimized prompt that was submitted
   - Any relevant notes (resolution, limitations)
4. Ask the user if they want:
   - Variations (re-run with modified prompt)
   - Switch platform (try the other platform with the same prompt)
   - Prompt adjustments (modify specific aspects)

## Error Handling & Fallback

| Error | Action |
|-------|--------|
| Grok rate limit (~10/2hr) | Inform user, offer Gemini fallback |
| Gemini content policy block | Inform user, offer Grok fallback |
| Page not loaded / login required | Detect login page via snapshot, instruct user to log in manually |
| Image not appearing after wait | Extend wait by 15s, re-snapshot; if still missing, report timeout |
| Unexpected page layout | Take screenshot for debugging, report to user |
| JavaScript errors | Use `mcp__browser-tools__getConsoleErrors` to diagnose |
| Network failures | Use `mcp__browser-tools__getNetworkErrors` to diagnose |

## Platform Limits Quick Reference

| | Grok | Gemini |
|---|---|---|
| Free tier | ~10 images / 2hr | 100/day standard, 2/day thinking |
| Pro tier | 50-100 / 2hr (Premium+) | 1,000/day (AI Pro) |
| Speed | 3-5 seconds | 10-30 seconds |
| Resolution | Fixed 4:3 (1024x768) | Up to 4K |
| Text rendering | Unreliable | Excellent |
| Content policy | Permissive | Strict |
| Best for | Speed, portraits, creative | Text, precision, professional |

## Important Notes

- Both platforms require the user to be logged in via the browser that Playwright controls.
  The Playwright browser session persists — once logged in, subsequent calls reuse the session.
- Always take a screenshot as proof of generation, even if the accessibility tree shows the image.
- Grok's Aurora engine produces fixed 4:3 ratio images — inform the user if different ratios are needed.
- Gemini's interface may vary by language/locale (the URL includes `hl=zh-TW` for Traditional Chinese).
- If both platforms fail, suggest using the **image-prompt** skill standalone to get an optimized
  prompt for manual use on other platforms.
- When the user asks to generate multiple images, process them one at a time to avoid rate limits.

## Additional Resources

### Reference Files
- **`references/platform-comparison.md`** — Detailed platform comparison, rate limits,
  content policies, decision matrix, and troubleshooting guide
