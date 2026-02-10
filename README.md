[English](README.md) | [繁體中文](README.zh.md)

# image-gen

A Claude Code skill that generates AI images by automating Grok and Gemini web interfaces via Playwright. Accepts vague descriptions in any language, optimizes the prompt, selects the best platform, and returns the generated image.

## What It Does

1. Takes a vague description from the user (any language)
2. Uses the **image-prompt** skill to optimize the description into a professional prompt
3. Routes to the best platform based on request characteristics:
   - **Grok**: Speed, creative art, human portraits, permissive content
   - **Gemini**: Text in images, high resolution, precision, professional quality
4. Automates the web interface via MCP Playwright to submit the prompt
5. Returns the generated image via screenshot

## Prerequisites

- **image-prompt** skill installed (`~/.claude/skills/image-prompt/`)
- MCP Playwright server connected
- MCP Browser Tools server connected
- Browser logged in to:
  - [Grok](https://grok.com/) (for Grok route)
  - [Gemini](https://gemini.google.com/app) (for Gemini route)

## Installation

```bash
git clone https://github.com/joneshong-skills/image-gen.git ~/.claude/skills/image-gen
```

## Usage

Once installed, ask Claude to generate images:

- *"Generate an image of a cat wearing a space suit on the moon"*
- *"幫我生成一張櫻花樹下的武士圖片"*
- *"Create a professional banner with the text 'HELLO WORLD'"* (routes to Gemini)
- *"用 Grok 幫我畫一個賽博龐克城市"* (forces Grok)
- *"Use Gemini to generate a high-res product photo"* (forces Gemini)

## Platform Comparison

| Feature | Grok | Gemini |
|---------|------|--------|
| Speed | 3-5 seconds | 10-30 seconds |
| Text rendering | Unreliable | Excellent |
| Content policy | Permissive | Strict |
| Resolution | Fixed 4:3 | Up to 4K |
| Free limit | ~10/2hr | 100/day standard, 2/day thinking |
| Best for | Speed, portraits, creative | Text, precision, professional |

## Project Structure

```
image-gen/
├── SKILL.md                    # Skill definition and workflow
├── README.md                   # This file
├── README.zh.md                # Traditional Chinese README
└── references/
    └── platform-comparison.md  # Detailed platform comparison and troubleshooting
```

## License

MIT
