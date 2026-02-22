# Observations — image-gen

## Pending

(none)

## Resolved

### 2026-02-12 — AppleScript save dialog approach (applied in v0.4.0)
- **Resolution**: Incorporated into SKILL.md Step 4. Chrome AppleScript triggers download button via `execute javascript`, System Events handles native save dialog. Dialog structure: `sheet 1 of window 1` → `splitter group 1` → `text field "儲存為："` + `pop up button "位置："` + `button "儲存"`. Replaced v0.3.0's Fetch-in-Page / Navigate-to-Image approach.

### 2026-02-12 — Download approach research (superseded by v0.4.0, archived)
- **Resolution**: Superseded by AppleScript approach in v0.4.0. Historical findings preserved: `browser_run_code` download fails (MCP extension architecture); Grok fetch-in-page works; Gemini requires Navigate-to-Image + canvas. These remain valid fallback approaches if AppleScript is unavailable, documented in SKILL.md "Fallback: Screenshot Only" section.

### 2026-02-11 — Grok free tier rate limits verified (dismissed)
- **Resolution**: Perplexity research (24 sources) confirmed grok.com standalone is still free. Rate limits: ~10-20 images/day or ~10 per 2-hour window (rolling reset). X platform restricted to paid subscribers (Jan 2026), but grok.com unaffected. Current SKILL.md documentation ("~10 images/2hr") remains accurate.

### 2026-02-11 — Gemini image model Nano Banana Pro verified (dismissed)
- **Resolution**: Perplexity research (27 sources) confirmed Nano Banana Pro (Gemini 3 Pro Image) launched Nov 2025. Consumer web uses "Thinking mode" toggle (no standalone model selector). Free tier: ~2 images/day for thinking mode, 1024x1024 max. Current SKILL.md already documents "2/day thinking" which is correct. No web interface changes in Feb 2026. No action needed.

### 2026-02-11 — Grok output dimensions not matching documented 4:3 (applied in v0.2.0)
- **Resolution**: User confirmed Grok supports prompt-based aspect ratio control. External research verified: xAI API supports 13+ ratios, `auto` mode selects best ratio for prompt, web UI responds to ratio keywords in prompt text (since May 2025). Updated SKILL.md and platform-comparison.md to reflect flexible ratios instead of "Fixed 4:3".
