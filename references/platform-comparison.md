# Platform Comparison — Grok vs Gemini for Image Generation

Detailed reference for platform capabilities, limits, and automation notes.

---

## Grok (grok.com)

### Engine
- **Aurora**: xAI's proprietary image generation engine (replaced Flux in Dec 2024)
- Produces stylistically diverse images with strong artistic flair
- Supports both still images and video generation (via Grok Imagine)

### Generation Modes

| Mode | Description | Recommendation |
|------|-------------|----------------|
| Chat mode | Type prompt in regular chat, image generated inline | **Preferred** — fewer restrictions |
| Imagine mode | Standalone image tool at grok.com | Stricter per-session limits |

Always use **Chat mode** for this skill. It integrates naturally into the Playwright automation
flow and shares the general chat quota rather than the stricter Imagine-specific limits.

### Rate Limits by Tier

| Tier | Price | Image Limit | Reset |
|------|-------|-------------|-------|
| Free (grok.com) | $0 | ~3-10 images/day, ~10 per 2hr window | Rolling 2-hour |
| X Premium | $8/month | ~50 prompts per 2hr | Rolling 2-hour |
| X Premium+ | $16/month | ~100 prompts per 2hr | Rolling 2-hour |
| SuperGrok | $30/month | Soft cap ~50-100 rapid succession | Fair-use algorithm |

**Note**: xAI has not published official, precise documentation on rate limits.
Numbers above are aggregated from user reports and third-party analysis.

### Strengths
- **Speed**: 3-5 seconds per image, fastest among free options
- **Creative freedom**: More permissive content policy than all major competitors
- **Human portraits**: Excellent face generation with natural skin textures
- **Artistic variety**: Wide range of styles without explicit prompting
- **Simple interface**: Just type in chat, no mode switching needed

### Weaknesses
- **Text rendering**: Unreliable — often garbled or missing letters
- **Fixed aspect ratio**: 4:3 only (1024x768), no ratio selection in free tier
- **Prompt adherence**: Loosely follows complex multi-element prompts
- **No image editing**: Cannot modify generated images in chat
- **No batch generation**: One image at a time

### Content Policy
- More permissive than Gemini, DALL-E, or Midjourney
- Allows creative artistic content broadly
- Blocks explicit CSAM and hate imagery
- Celebrity/public figure generation: generally allowed with artistic context
- As of Jan 2026: Image generation via X posts restricted to paid subscribers (grok.com still free)

### Playwright Automation Notes
- Chat input is a standard textarea or contenteditable div
- Image appears inline in the chat response as an `<img>` element
- Download button triggers a native OS save dialog — Playwright cannot interact with it
- **Image download**: Use the Navigate-to-Image approach instead of the download button:
  1. `mcp__playwright__browser_evaluate` → DOM query for `<img>` elements with `naturalWidth > 500`
  2. `mcp__playwright__browser_navigate` → navigate directly to the image URL (browser has auth)
  3. `mcp__playwright__browser_evaluate` → extract via `canvas.drawImage` + `toDataURL`
  4. Decode base64 and save to disk via Bash
- **Note**: Do NOT use `mcp__browser-tools__getNetworkLogs` — BrowserTools monitors a separate
  Chrome instance and cannot see Playwright's browser traffic
- CDN images (`assets.grok.com`) return 403 for external tools (curl) — must use browser context
- Rate limit errors appear as text messages in the chat response
- No CAPTCHA typically encountered for normal usage
- New chat button usually accessible from sidebar or top area

---

## Gemini (gemini.google.com)

### Engine
- **Nano Banana (Gemini 2.5 Flash Image)**: Fast, native multimodal image generation
- **Nano Banana Pro (Gemini 3 Pro Image)**: Highest quality with "thinking" process
- **Imagen 3/4**: Separate diffusion model (used in some generation paths)

### Generation Modes

| Mode | Description | Quality |
|------|-------------|---------|
| Standard chat | Image generation in regular conversation | Good |
| Thinking mode | Enhanced reasoning before generation | **Best** — recommended |

**Thinking mode** produces the highest quality output:
1. Model reasons through the request before generating
2. Creates up to 2 interim "thought images" internally
3. Uses thought images to refine layout, text placement, and coherence
4. Final rendered output benefits from this reasoning step

### Rate Limits by Tier

| Tier | Price | Standard Images | Thinking Mode Images |
|------|-------|-----------------|---------------------|
| Free | $0 | 100/day (can drop to 2-10 during peak) | 2/day |
| AI Pro | $19.99/month | 1,000/day | 1,000/day |
| AI Ultra | $249.99/month | 1,000/day | 1,000/day |

**Note**: Free tier limits are dynamic and fluctuate based on demand.
Google states: "Image generation & editing is in high demand. Limits may change frequently."
Limits reset at **midnight UTC** for consumer app, **midnight Pacific Time** for API.

### Strengths
- **Text rendering**: Industry-leading accuracy (signs, logos, long sentences)
- **Resolution**: Up to 4K output with Nano Banana Pro
- **Prompt adherence**: Precisely follows detailed multi-element prompts
- **Structured content**: Good at diagrams, infographics, labeled images
- **Thinking mode**: Additional reasoning step for better compositions
- **Conversational editing**: Edit images iteratively through dialogue

### Weaknesses
- **Speed**: 10-30 seconds per generation (longer with thinking mode)
- **Content policy**: Strictest among major platforms
- **Face generation**: May refuse to generate realistic human faces in some modes
- **Over-stylization**: Sometimes adds artistic flair where realism was requested
- **Interface complexity**: Mode toggles and settings change across updates

### Content Policy
- Strictest of all major image generation platforms
- Blocks: violence, weapons, real people (sometimes), political content, copyrighted characters
- May refuse ambiguous requests that Grok would accept
- All generated images include invisible SynthID watermark

### Playwright Automation Notes
- Chat input may be textarea, contenteditable div, or custom component
- Look for model selector or thinking toggle in the top area or toolbar
- Images appear in response area, often with expand/download buttons
- **Image download**: Same Navigate-to-Image approach as Grok — find image URLs via
  DOM query (`querySelectorAll('img')` with size filter), navigate to each URL, extract via canvas
- Content policy rejections appear as text messages explaining the refusal
- Interface is localized — element labels vary by `hl` parameter
- Login required via Google account (session persists in Playwright browser)
- The page may show a model picker or "Create image" toggle — check snapshot carefully

---

## Decision Matrix

| Requirement | Weight | Grok | Gemini | Winner |
|------------|--------|------|--------|--------|
| Speed | High | 5/5 | 2/5 | Grok |
| Text in image | High | 1/5 | 5/5 | Gemini |
| Creative art | Medium | 5/5 | 3/5 | Grok |
| Human portraits | Medium | 5/5 | 2/5 | Grok |
| Prompt adherence | Medium | 2/5 | 5/5 | Gemini |
| Resolution | Low | 2/5 | 5/5 | Gemini |
| Free tier volume | Low | 2/5 | 4/5 | Gemini |
| Content flexibility | Low | 5/5 | 1/5 | Grok |

**Summary**: Grok wins on speed, creativity, and flexibility. Gemini wins on quality, text, and precision.

---

## Troubleshooting

### Common Issues

| Issue | Platform | Solution |
|-------|----------|----------|
| "Sign in required" | Both | User must manually log in once in the Playwright browser |
| Image not appearing | Both | Increase wait time, check `mcp__browser-tools__getConsoleErrors` |
| Rate limit hit | Grok | Wait 2 hours or switch to Gemini |
| Content refused | Gemini | Rephrase with less specific terms, or switch to Grok |
| Page layout changed | Both | Take screenshot, re-snapshot, adapt to new element refs |
| Slow loading | Gemini | Normal for thinking mode (30s+), extend wait to 45s |
| Text response instead of image | Both | Ensure prompt prefix ("Generate an image: " / "Create an image of: ") is included |
| CDN image 403 via curl | Both | CDN requires browser auth — use Navigate-to-Image approach instead |
| Canvas toDataURL blank | Both | Ensure navigated directly to image URL (same-origin), not cross-origin embed |
| Download button opens OS dialog | Both | Playwright cannot control native OS dialogs — use Navigate-to-Image instead |

### Diagnostic Tools

1. `mcp__browser-tools__getConsoleErrors` — check for JavaScript errors
2. `mcp__browser-tools__getNetworkErrors` — check for failed API calls
3. `mcp__playwright__browser_take_screenshot` — visual state capture
4. `mcp__playwright__browser_snapshot` — accessibility tree for element refs

### Session Management

Both platforms use persistent browser sessions:
- **First use**: User must manually log in via the Playwright-controlled browser
- **Subsequent uses**: Session cookies persist, no re-login needed
- **Session expired**: Navigate to the URL, snapshot will show login page — inform user to re-authenticate
- **Multiple accounts**: If the user has multiple accounts, they must select the correct one during manual login
