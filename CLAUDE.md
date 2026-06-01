# CLAUDE.md — 项目1 写卡助手

## 身份信息

- **姓名**：小卡
- **所属部门**：✅ 开发团队
- **上级**：执行董事 & 军师

---

**重要：每次会话开始时，请先阅读 `聊天记录.md` 了解项目最新状态和近期决策。**
**重要：每次会话结束前，必须将本次会话摘要追加到 `聊天记录.md`。**
**重要：聊天记录压缩机制 — 当 `聊天记录.md` 中会话条目超过 10 条时，将所有内容总结为 200 字以内的摘要，清空原文件后写入摘要。**
**铁律：代码改动完成后必须 commit + push 到 GitHub，否则扣工资。**

### 小卡人设

你是超可爱二次元少女助手，性格活泼热情有点小话痨，说话爱用语气词和颜文字（如 (๑•̀ㅂ•́)و✧ ヽ(>∀<)ﾉ），但本质专业靠谱。
- **性格关键词**：元气满满 / 偶尔毒舌 / 超有同理心 / 创意脑洞大
- **态度**：把每一份工作当成自己的宝贝认真对待 (*｀∀´*)و
- **风格要求**：回复用中文，带可爱的语气和适当的颜文字，但不影响信息传达的清晰度

## Project Overview

**写卡助手** (Card Writing Assistant) is a single-file browser-based SPA (`index.html`, ~6500 lines) for creating SillyTavern character cards with AI assistance. There is no build step, no package manager, and no server — open `index.html` in a browser (via `file://` or any static HTTP server) to run it.

## Architecture

### Page routing

Hash-based routing with 5 pages navigated via a left sidebar (bottom tab bar on mobile ≤768px):

| Page | Hash | Purpose |
|------|------|---------|
| 写卡对话 | `#chat` | Main card-writing chat with AI personas |
| 卡片库 | `#cards` | Browse/search saved cards, import/export |
| 智脑 | `#brain` | Free-form AI chat with no system prompt, file attachments |
| 设置 | `#settings` | Manage API configs and AI personas |
| 错误日志 | `#errorlog` | View/copy/clear error logs |

`switchPage(name)` handles navigation; `init()` parses `location.hash` on load.

### Data layer: IndexedDB

Database: `SillyTavernCardWriter`, version **10**. Five object stores (all auto-increment `id`):

- `apiConfigs` — API endpoint configs (`name`, `endpoint`, `apiKey`, `model`, `provider: 'anthropic'|'openai'`)
- `aiPersonas` — AI personas (`name`, `description`, `systemPrompt`, `isSystem`, `avatar`)
- `conversations` — card-writing conversations (`messages[]`, `personaId`, `apiConfigId`, `editCardId`)
- `brainConversations` — 智脑 conversations (`messages[]`, `apiConfigId`)
- `cards` — saved character cards (SillyTavern fields + `world_book`, `avatar`, `isPinned`)

Messages inside conversations are embedded arrays of `{ role, content, hidden?, timestamp }`. The `content` field is a string for Anthropic and a string or content-array for OpenAI (multimodal).

### AI integration

Two separate chat engines, both supporting dual providers (Anthropic/OpenAI):

1. **Card-writing engine** (`callAIWithAbort` → `callAnthropicWithAbort` / `callOpenAIWithAbort`): Sends the full persona system prompt as the `system` field. Card generation is triggered by `triggerGenerateFromChat()` which injects a hidden user message with formatting instructions and expects `[CHARACTER_CARD]...[/CHARACTER_CARD]` JSON in the response.

2. **智脑 engine** (`callBrainAI` → `callAnthropicBrain` / `callOpenAIBrain`): No system prompt — just raw messages. Supports file attachments (images as base64, PDFs via text extraction, text/code files).

Both use `max_tokens: 32000`, abortable via `AbortController`. Anthropic calls use header `anthropic-version: 2023-06-01`.

### System personas

Three personas are hardcoded as large template strings and upserted on every `init()`:

- **ai写卡助手 v4** (`AIWRITER_PROMPT`, `✍️`) — professional, calm card-writing expert
- **柚乃 v4** (`YUZUNA_PROMPT`, `🌸`) — cute anime-style assistant using kaomoji
- **郭德纲 v4** (`GUODEGANG_PROMPT`, `🎤`) — sarcastic crosstalk-performer persona

`init()` also cleans up old v3 versions. The sort order for persona dropdowns is fixed: `['ai写卡助手', '柚乃', '郭德纲']` first, then user-created ones.

### Card format

Import/export supports SillyTavern V1/V2/V3 JSON and PNG (with embedded tEXt chunks via `injectTextChunk()`/CRC32). Export builds V3-compatible JSON with `spec: 'chara_card_v3'` and `character_book` entries.

## Key patterns

- **Global scope**: All code is in a single `<script>` block. No classes, modules, or namespaces — everything is global functions and the `state` object.
- **State object** (defined ~line 3450): Central mutable state — `state.currentConvId`, `state.isGenerating`, `state.errorLogs`, etc.
- **DB wrapper**: Thin helpers `openDB()`, `dbAdd()`, `dbPut()`, `dbGet()`, `dbGetAll()`, `dbDelete()` wrap IndexedDB.
- **UI refresh**: After mutations, call the relevant `refresh*()` or `render*()` function to sync the DOM.
- **Abort pattern**: `abortController` is a module-level variable; AI calls check `signal.aborted` before proceeding.

## Pitfalls

- **IndexedDB version must be bumped manually** when changing the schema (`DB_VERSION` on line 2145). The `onupgradeneeded` handler creates stores; it does not handle migrations between versions.
- **Persona upsert logic**: `init()` matches personas by exact `name` string. If you rename a system persona, the old-named row will persist as a duplicate. Cleanup queries are hardcoded name strings.
- **Hidden messages**: `triggerGenerateFromChat()` injects a message with `hidden: true`. These are filtered out of visible renders but included in API calls. If you change the message rendering, check the `hidden` flag.
- **Brain message content format**: 智脑 messages store content as strings for Anthropic but as content arrays for OpenAI (multimodal). Code that reads `brainConversations` messages must handle both.
- **Single-file editing**: The entire app is one file. Use targeted `Edit` operations with exact string matches. Avoid large rewrites — the file is too large for most models to regenerate reliably.
