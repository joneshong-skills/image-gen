# Observations — image-gen

## Pending

### 2026-02-11 — Grok output dimensions not matching documented 4:3

- **Category**: tech
- **Evidence**: Generated images were 784x1168 (≈2:3 portrait) when SKILL.md documents "Fixed 4:3 (1024x768)". Prompt was a character portrait with no explicit aspect ratio instruction.
- **Research**: Grok API supports 13+ aspect ratios (May 2025). Web UI prompt-based ratio control reported but not fully confirmed for free tier. API features may not map 1:1 to web UI behavior.
- **Confidence**: Low
- **Trigger**: If 2+ more executions produce non-4:3 output from Grok web UI, update SKILL.md line 198 and platform-comparison.md line 45. Investigate whether prompt wording or a new UI selector is responsible.

## Resolved
