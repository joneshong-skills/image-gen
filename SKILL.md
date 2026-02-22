---
name: image-gen
description: >-
  This skill should be used when the user asks to "generate an image",
  "create an image", "draw a picture", "幫我生成圖片", "產生圖片",
  "用 Grok 生成圖片", "用 Gemini 生成圖片", "make me an image",
  or discusses AI image generation via Grok or Gemini web platforms.
version: 0.4.1
tools: mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_type, mcp__playwright__browser_click, mcp__playwright__browser_evaluate, mcp__playwright__browser_wait_for, mcp__playwright__browser_take_screenshot
argument-hint: "描述想要的圖片（中文或英文皆可）"
---

# Image Gen

Generate AI images by automating Grok and Gemini web interfaces via Playwright.
Accept vague descriptions in any language, optimize the prompt via the **image-prompt** skill,
select the best platform, and return the generated image.

## Agent Delegation

All image generation processing delegates to the `media` agent (Haiku, maxTurns=10).
Main context handles user interaction and parameter clarification only.

```
Main context (parse request, clarify params)
  └─ Task(subagent_type: media, prompt: "[specific operation]...")
```

For batch operations, spawn parallel media agents (one per file).
For web UI-based generation (Grok, Gemini), use the `browser` agent as fallback when
direct API access is unavailable.

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

Download the original image files by clicking the platform's download button and handling
the native macOS save dialog. This uses the **macos-ui-automation** skill's two-step pattern:
Chrome AppleScript triggers the download → System Events handles the save dialog.

#### 4a. Click the Download Button

Use the **macos-ui-automation** skill's Chrome AppleScript pattern to trigger the download
button via JavaScript. Target buttons by their text content:

- **Grok**: buttons contain text `下載`
- **Gemini**: buttons contain text `下載原尺寸圖片` or aria-label with `download`
- Use `mouseenter` dispatch on the parent to reveal hover-only buttons before clicking
- When multiple images exist, target specific buttons (e.g., by index or parent context)

#### 4b. Handle the Native Save Dialog

Follow the **macos-ui-automation** skill's save dialog workflow:
1. Wait for dialog to appear (poll `sheet 1 of window 1`)
2. Set filename, navigate to default save path (`~/Downloads/claude_code_skill/`), click save
3. See `macos-ui-automation` SKILL.md § Step 3 for the full pattern

#### 4c. Verify and Present

1. Verify the file exists and has valid content:
   ```bash
   ls -lh ~/Downloads/claude_code_skill/<filename>.jpg
   ```
2. Use the Read tool to display the downloaded image to the user
3. Include a brief summary:
   - Platform used (Grok / Gemini)
   - The optimized prompt that was submitted
   - File paths on disk (e.g., `~/Downloads/claude_code_skill/image-name.jpg`)
   - Image dimensions and file size
4. Ask the user if they want:
   - Variations (re-run with modified prompt)
   - Switch platform (try the other platform with the same prompt)
   - Prompt adjustments (modify specific aspects)

#### Fallback: Screenshot Only

If the download button is not found or AppleScript fails (e.g., missing Accessibility
permission), fall back to `mcp__playwright__browser_take_screenshot` and present the
screenshot instead. Inform the user that only a screenshot is available and suggest
manually downloading from the platform.

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
- Always download original images via the AppleScript approach (see **macos-ui-automation** skill). Use screenshots only as fallback.
- Grok supports aspect ratio control via prompt text (e.g., include "1:1", "16:9", "9:16" in the prompt).
  When no ratio is specified, the model uses `auto` mode and selects the best ratio for the prompt.
- Gemini's interface may vary by language/locale (the URL includes `hl=zh-TW` for Traditional Chinese).
- If both platforms fail, suggest using the **image-prompt** skill standalone to get an optimized
  prompt for manual use on other platforms.
- When the user asks to generate multiple images, process them one at a time to avoid rate limits.

## Continuous Improvement

This skill evolves with each use. After every invocation:

1. **Reflect** — Identify what worked, what caused friction, and any unexpected issues
2. **Record** — Append a concise lesson to `lessons.md` in this skill's directory
3. **Refine** — When a pattern recurs (2+ times), update SKILL.md directly

### lessons.md Entry Format

```
### YYYY-MM-DD — Brief title
- **Friction**: What went wrong or was suboptimal
- **Fix**: How it was resolved
- **Rule**: Generalizable takeaway for future invocations
```

Accumulated lessons signal when to run `/skill-optimizer` for a deeper structural review.

## Additional Resources

### Reference Files
- **`references/platform-comparison.md`** — Detailed platform comparison, rate limits,
  content policies, decision matrix, and troubleshooting guide
