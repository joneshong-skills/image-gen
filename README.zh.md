[English](README.md) | [繁體中文](README.zh.md)

# image-gen

一個 Claude Code 技能，透過 Playwright 自動化 Grok 和 Gemini 網頁介面來生成 AI 圖片。
接受任何語言的模糊描述，最佳化提示詞，選擇最適合的平台，並回傳生成的圖片。

## 功能特色

1. 接受使用者的模糊描述（支援任何語言）
2. 使用 **image-prompt** 技能將描述最佳化為專業提示詞
3. 根據需求特徵路由到最佳平台：
   - **Grok**：速度快、創意藝術、人像、寬鬆內容政策
   - **Gemini**：圖片中的文字、高解析度、精確度、專業品質
4. 透過 MCP Playwright 自動化網頁介面提交提示詞
5. 透過截圖回傳生成的圖片

## 前置條件

- 已安裝 **image-prompt** 技能（`~/.claude/skills/image-prompt/`）
- MCP Playwright 伺服器已連接
- MCP Browser Tools 伺服器已連接
- 瀏覽器已登入：
  - [Grok](https://grok.com/)（Grok 路線）
  - [Gemini](https://gemini.google.com/app)（Gemini 路線）

## 安裝

```bash
git clone https://github.com/joneshong-skills/image-gen.git ~/.claude/skills/image-gen
```

## 使用方式

安裝後，直接要求 Claude 生成圖片：

- *「幫我生成一張太空貓的圖片」*
- *「Generate an image of a samurai under cherry blossoms」*
- *「幫我做一個寫著 'HELLO WORLD' 的專業橫幅」*（路由到 Gemini）
- *「用 Grok 幫我畫一個賽博龐克城市」*（強制使用 Grok）
- *「用 Gemini 生成高解析度產品照」*（強制使用 Gemini）

## 平台比較

| 特性 | Grok | Gemini |
|------|------|--------|
| 速度 | 3-5 秒 | 10-30 秒 |
| 文字渲染 | 不可靠 | 優秀 |
| 內容政策 | 寬鬆 | 嚴格 |
| 解析度 | 固定 4:3 | 最高 4K |
| 免費額度 | ~10 張/2小時 | 100 張/天（標準）、2 張/天（思考模式） |
| 最適合 | 速度、人像、創意 | 文字、精確、專業 |

## 專案結構

```
image-gen/
├── SKILL.md                    # 技能定義及工作流程
├── README.md                   # 英文說明
├── README.zh.md                # 繁體中文說明（本檔案）
└── references/
    └── platform-comparison.md  # 詳細平台比較及疑難排解
```

## 授權

MIT
