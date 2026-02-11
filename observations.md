# Observations — image-gen

## Pending

### 2026-02-11 — Grok free tier rate limits may have changed
- **Category**: tech
- **Evidence**: CNN Jan 2026 reports image gen on X restricted to paid subscribers. External research suggests grok.com standalone still free but possibly ~3 images/day (vs current doc ~10/2hr).
- **Research**: grok.com standalone confirmed still free. X platform integration restricted to paid users. Rate limit numbers unverified.
- **Confidence**: Low
- **Trigger**: Live test on grok.com to verify current free tier limits. If <5 images/day confirmed, update rate limit table.

### 2026-02-11 — Gemini image model updated to Nano Banana Pro
- **Category**: tech
- **Evidence**: Google DeepMind released Nano Banana Pro (Gemini 3 Pro Image Preview) with 2K/4K output, improved text rendering.
- **Research**: New model rolling out globally. UI flow may change: "Create images" with "Thinking" model. Free tier users may revert to original model after quota.
- **Confidence**: Medium
- **Trigger**: Next Gemini image generation attempt — verify if UI flow or model selection has changed.

## Resolved

### 2026-02-11 — Grok output dimensions not matching documented 4:3 (applied in v0.2.0)
- **Resolution**: User confirmed Grok supports prompt-based aspect ratio control. External research verified: xAI API supports 13+ ratios, `auto` mode selects best ratio for prompt, web UI responds to ratio keywords in prompt text (since May 2025). Updated SKILL.md and platform-comparison.md to reflect flexible ratios instead of "Fixed 4:3".
