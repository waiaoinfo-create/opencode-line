# OpenCode LINE Bot 整合建置報告

**專案名稱：** OpenCode LINE Bot Integration
**版本：** v1.0
**日期：** 2026-06-20
**作者：** waiaoinfo-create

---

## 目錄

1. [專案概述](#1-專案概述)
2. [系統架構](#2-系統架構)
3. [技術棧](#3-技術棧)
4. [前置準備作業](#4-前置準備作業)
5. [建置步驟](#5-建置步驟)
6. [服務說明](#6-服務說明)
7. [LINE Bot 功能說明](#7-line-bot-功能說明)
8. [AI 模型一覽](#8-ai-模型一覽)
9. [操作指南](#9-操作指南)
10. [常見問題](#10-常見問題)
11. [附錄](#11-附錄)

---

## 1. 專案概述

### 1.1 目的

將 **OpenCode AI 程式碼助理** 與 **LINE Messaging API** 進行整合，讓使用者可以透過 LINE 官方帳號，以聊天方式與 AI 進行互動，包含程式碼撰寫、問題諮詢、文件查詢等功能。

### 1.2 適用對象

- 需要在手機上隨時使用 AI 輔助的開發者
- 想在 LINE 群組中整合 AI 助理的團隊
- 對 AI 聊天機器人有興趣的一般使用者

### 1.3 主要功能

| 功能 | 說明 |
|------|------|
| 多輪對話 | 支援連續對話，保留上下文 |
| 模型切換 | 可自由切換不同 AI 模型 |
| 群組聊天 | 支援 LINE 群組，AI 自動判斷是否回應 |
| 多模型支援 | Anthropic、OpenAI、Google、DeepSeek 等 |
| 免費方案 | 內建 Big Pickle 模型，零成本使用 |

---

## 2. 系統架構

### 2.1 整體架構圖

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│          │     │          │     │          │     │          │     │          │
│ LINE App │────▶│  ngrok   │────▶│ line-bot │────▶│ opencode │────▶│ AI Model │
│          │     │  Tunnel  │     │  (Bun)   │     │  Server  │     │          │
│          │     │          │     │  :3000   │     │  :4096   │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘     └──────────┘
                                                          ↕
                                                    ┌──────────┐
                                                    │MCP Tools │
                                                    │(optional)│
                                                    └──────────┘
```

### 2.2 資料流說明

```
使用者傳送 LINE 訊息
        │
        ▼
LINE Platform → Webhook (POST /webhook)
        │
        ▼
ngrok Tunnel (將 localhost:3000 曝露為公開 HTTPS 網址)
        │
        ▼
line-bot (Bun/TypeScript)
  ├── 解析訊息內容
  ├── 判斷 session（群組/個人）
  ├── 查詢使用者記憶 (LINE displayName)
  ├── 加入時間上下文 (Bangkok TZ)
  └── 呼叫 OpenCode API
        │
        ▼
OpenCode Server (HTTP :4096)
  ├── POST /session/{id}/message
  ├── AI 處理回應
  └── 回傳結果
        │
        ▼
line-bot 接收回應
  ├── replyMessage（優先，免費）
  └── pushMessage（備援，配額有限）
        │
        ▼
使用者收到 LINE 訊息
```

---

## 3. 技術棧

### 3.1 使用技術

| 元件 | 技術 | 版本 |
|------|------|------|
| LINE Bot 服務 | Bun (JavaScript/TypeScript) | 1.x |
| AI 引擎 | OpenCode Server | latest |
| 容器化 | Docker + Docker Compose | 4.74.0 |
| 公開 Tunnel | ngrok | 3.3.1 |
| LINE SDK | @line/bot-sdk | v9 |
| 底層作業系統 | Alpine Linux (opencode) / Debian (line-bot) | - |

### 3.2 Docker 容器

| 容器名稱 | 基底映像 | 說明 |
|-----------|---------|------|
| opencode-server | ghcr.io/anomalyco/opencode:latest | OpenCode AI 引擎 |
| opencode-line-bot | oven/bun:1 | LINE Webhook 處理 |
| opencode-line-ngrok | ngrok/ngrok:latest | 公開隧道服務 |

---

## 4. 前置準備作業

### 4.1 所需帳號與憑證

| 項目 | 必要性 | 用途 | 取得方式 |
|------|--------|------|---------|
| LINE Channel Access Token | **必要** | LINE API 認證 | LINE Developers Console |
| LINE Channel Secret | **必要** | Webhook 簽名驗證 | LINE Developers Console |
| AI API Key | **必要 (擇一)** | 呼叫 AI 模型 | Anthropic / OpenAI / Google |
| ngrok Authtoken | 建議 | 固定 Tunnel 網址 | ngrok.com 註冊 |
| GitHub Token | 選填 | OpenCode 容器內使用 | GitHub Settings |

### 4.2 LINE Developers Console 設定步驟

**步驟 1：建立 Provider**
1. 前往 https://developers.line.biz/console/
2. 使用 LINE 帳號登入
3. 點擊「Create Provider」
4. 輸入 Provider 名稱（例如：My AI Bot）

**步驟 2：建立 Messaging API Channel**
1. 在 Provider 下點擊「Create Channel」
2. 選擇「Messaging API」
3. 填寫：
   - Channel name：你的 Bot 名稱
   - Channel description：簡短說明
   - Category：選擇合適類別
   - 其他欄位依需求填寫
4. 同意條款後建立

**步驟 3：取得憑證**
1. **Channel Secret**：Basic Settings 頁面下方
2. **Channel Access Token**：Messaging API 頁面 → Issue
3. **LINE OA URL**：Messaging API 頁面 → Your LINE Official Account URL

### 4.3 AI API Key 申請

| AI 服務 | 等級 | 申請連結 | 說明 |
|---------|------|---------|------|
| Big Pickle | **免費** | 內建 | OpenCode 內建免費模型，不需金鑰 |
| Claude Haiku | 💲 | https://console.anthropic.com | 低成本，速度快 |
| Claude Sonnet | 💲💲 | https://console.anthropic.com | 平衡效能與品質 |
| GPT-5 Mini | 💲 | https://platform.openai.com | 輕量模型 |
| Gemini Flash | 💲 | https://aistudio.google.com | Google 輕量模型 |
| DeepSeek Chat | 💲 | https://platform.deepseek.com | 低成本中文模型 |

---

## 5. 建置步驟

### 5.1 環境準備

**安裝 Docker Desktop**
```bash
winget install Docker.DockerDesktop
```

**安裝 ngrok**
```bash
winget install ngrok.ngrok
```

### 5.2 取得專案

```bash
git clone https://github.com/monthop-gmail/opencode-line.git
cd opencode-line
```

### 5.3 設定環境變數

建立 `.env` 檔案：

```env
# ─── LINE 憑證 ───
LINE_CHANNEL_ACCESS_TOKEN=你的Channel Access Token
LINE_CHANNEL_SECRET=你的Channel Secret
LINE_OA_URL=https://line.me/ti/p/~你的官方帳號ID

# ─── AI 金鑰（擇一） ───
ANTHROPIC_API_KEY=sk-ant-你的金鑰

# ─── ngrok（選填） ───
NGROK_AUTHTOKEN=你的ngrok authtoken
```

### 5.4 啟動服務

```bash
docker compose up -d --build
```

預期輸出：
```
[+] Building ... done
[+] Running 4/4
 ✔ Container opencode-server       Started
 ✔ Container opencode-line-bot     Started
 ✔ Container opencode-line-ngrok   Started
```

### 5.5 設定 LINE Webhook

**步驟 1：取得 ngrok 網址**
```bash
docker logs opencode-line-ngrok
```
輸出範例：`https://abcd-123-45-67-89.ngrok-free.app`

**步驟 2：設定 Webhook**
1. 前往 LINE Developers Console → Messaging API 頁
2. 在 **Webhook URL** 填入：`https://你的ngrok網址/webhook`
3. 開啟 **Webhook** 開關（Enabled）
4. 點擊 **Verify** 驗證連線
5. 確認顯示 **Success**

### 5.6 驗證

1. 開啟 LINE 官方帳號聊天室
2. 傳送文字訊息（如：「你好」）
3. 應收到 AI 回覆
4. 輸入 `/help` 可查看所有指令

---

## 6. 服務說明

### 6.1 容器配置

| 項目 | opencode-server | opencode-line-bot | opencode-line-ngrok |
|------|----------------|-------------------|-------------------|
| 連接埠 | :4096 | :3000 | 動態 |
| 重新啟動 | unless-stopped | unless-stopped | unless-stopped |
| 相依性 | 無 | opencode | line-bot |
| 健康檢查 | GET /global/health | GET / | 無 |

### 6.2 Docker Volume

| Volume | 掛載點 | 用途 |
|--------|--------|------|
| opencode-data | /root/.local/share/opencode | 儲存 auth.json (API 金鑰) |
| opencode-state | /root/.local/state/opencode | 儲存 model.json (預設模型) |
| bind mount | /root/.config/opencode/opencode.json | Provider 設定檔 (唯讀) |

### 6.3 常用 Docker 指令

```bash
# 啟動服務
docker compose up -d

# 重新建置並啟動
docker compose up -d --build

# 檢視線上 Bot
docker compose up -d --build line-bot

# 檢視線上 OpenCode
docker compose up -d --build opencode

# 查看日誌
docker logs opencode-line-bot --tail 30 -f
docker logs opencode-server --tail 30 -f
docker logs opencode-line-ngrok

# 停止服務
docker compose down

# 完全清除（含 Volume）
docker compose down -v
```

---

## 7. LINE Bot 功能說明

### 7.1 指令列表

| 指令 | 別名 | 功能 | 範例 |
|------|------|------|------|
| `/new` | - | 開始新對話 | `/new` |
| `/abort` | - | 取消進行中的 prompt | `/abort` |
| `/sessions` | - | 查看所有 session 狀態 | `/sessions` |
| `/model` | - | 查看當前模型 | `/model` |
| `/model <name>` | - | 切換 AI 模型 | `/model sonnet` |
| `/about` | `/who` | Bot 自我介紹 | `/about` |
| `/help` | `/คำสั่ง` | 顯示所有指令 | `/help` |
| `/playground` | `/pg` | Playground 資訊 | `/playground` |
| `/meditation` | `/jibjib`, `/meditate`, `/สมาธิ` | JIBJIB Meditation | `/meditation` |
| `/cny` | - | 新年祝福 | `/cny` |

### 7.2 群組聊天功能

- **Session 共用**：同群組成員共用一個 session
- **自動判斷**：AI 會自動判斷訊息是否要回應
- **SKIP 機制**：不相干的訊息自動跳過（`[SKIP]`）
- **自然互動**：不需指定 @bot，AI 自行判斷

### 7.3 使用者上下文

每次 prompt 會自動包含：

1. **使用者記憶**
   - LINE 顯示名稱 (displayName)
   - 訊息計數
   - 1 小時快取

2. **時間上下文**
   - 曼谷時區 (UTC+7)
   - ISO 8601 格式

3. **回覆上下文**
   - 引用訊息 (quotedMessageId)
   - 回覆鏈結

### 7.4 回覆機制

| 方式 | 優先級 | 限制 | 說明 |
|------|--------|------|------|
| replyMessage | 最高 | 免費，無限 | 使用 replyToken，即時回覆 |
| pushMessage | 備援 | 免費方案 300則/月 | replyToken 過期時使用 |

---

## 8. AI 模型一覽

### 8.1 可用模型

| 指令 | 模型名稱 | 供應商 | 費用等級 | 適合場景 |
|------|---------|--------|---------|---------|
| `/model pickle` | Big Pickle | opencode | **免費** 🆓 | 日常對話、測試 |
| `/model haiku` | Claude Haiku 4.5 | anthropic | 💲 | 快速問答、簡單任務 |
| `/model sonnet` | Claude Sonnet 4.6 | anthropic | 💲💲 | 程式碼、複雜分析 |
| `/model opus` | Claude Opus 4.6 | anthropic | 💲💲💲 | 高難度推理 |
| `/model deepseek` | DeepSeek Chat | deepseek | 💲 | 中文對話 |
| `/model reasoner` | DeepSeek Reasoner | deepseek | 💲 | 邏輯推理 |
| `/model gpt5mini` | GPT-5 Mini | openai | 💲 | 輕量任務 |
| `/model gpt5` | GPT-5.2 | openai | 💲💲 | 一般任務 |
| `/model gpt5pro` | GPT-5.2 Pro | openai | 💲💲💲 | 專業任務 |
| `/model codex` | GPT-5.2 Codex | openai | 💲💲 | 程式碼生成 |
| `/model geminiflash` | Gemini 3 Flash | google | 💲 | 快速回應 |
| `/model gemini` | Gemini 3 Pro | google | 💲 | 一般任務 |
| `/model gemini31` | Gemini 3.1 Pro | google | 💲 | 進階任務 |

### 8.2 預設模型

- **預設值**：Big Pickle（免費）
- **儲存方式**：每個群組/使用者獨立儲存（modelPrefs Map）
- **切換方式**：LINE 指令 `/model <名稱>`

---

## 9. 操作指南

### 9.1 日常使用

**一般對話**
```
使用者：幫我寫一個 Python 計算機程式
AI Bot：好的，這是簡單的計算機程式...
```

**切換模型**
```
使用者：/model sonnet
AI Bot：已切換到 Claude Sonnet 4.6
```

**開始新對話**
```
使用者：/new
AI Bot：已開始新對話！
```

### 9.2 查看日誌與除錯

```bash
# 即時追蹤 LINE Bot 日誌
docker logs opencode-line-bot --tail 30 -f

# 查看 OpenCode 引擎狀態
docker logs opencode-server --tail 30 -f

# 檢查 ngrok 連線網址
docker logs opencode-line-ngrok

# 檢查 Docker 容器狀態
docker ps -a

# 重新啟動單一服務
docker compose restart line-bot
```

### 9.3 更新服務

```bash
# 拉取最新映像並重啟
docker compose pull
docker compose up -d

# 或重新建置
docker compose up -d --build
```

---

## 10. 常見問題

### Q1: 收不到 Bot 回覆？

1. 檢查 ngrok 是否正常：
   ```bash
   docker logs opencode-line-ngrok
   ```
2. 確認 LINE Webhook 設定正確
3. 檢查 LINE Bot 日誌是否有錯誤：
   ```bash
   docker logs opencode-line-bot --tail 50
   ```
4. 確認 .env 中的金鑰是否正確

### Q2: ngrok URL 變了？

- 每次 ngrok 重啟 URL 都會變
- 建議註冊 ngrok 帳號並設定 Authtoken
- 設定後需更新 LINE Webhook URL

### Q3: 如何更換 AI 模型？

在 LINE 聊天室輸入：
```
/model sonnet
```

### Q4: Docker 無法啟動？

```bash
# 檢查 Docker 服務狀態
docker info

# 清除並重新建置
docker compose down
docker compose up -d --build
```

### Q5: 容器重啟後對話不見了？

- Session 儲存在記憶體中 (Map)
- 重啟後會自動建立新的 session
- 只需傳送新訊息即可繼續使用

---

## 11. 附錄

### A. 相關連結

| 資源 | 網址 |
|------|------|
| GitHub 原始碼 | https://github.com/monthop-gmail/opencode-line |
| OpenCode 官方 | https://opencode.ai |
| LINE Developers | https://developers.line.biz |
| Anthropic Console | https://console.anthropic.com |
| ngrok Dashboard | https://dashboard.ngrok.com |
| Docker Desktop | https://www.docker.com/products/docker-desktop/ |

### B. 環境變數完整對照

| 變數名稱 | 必要 | 預設值 | 說明 |
|---------|------|--------|------|
| LINE_CHANNEL_ACCESS_TOKEN | ✅ | - | LINE Messaging API Token |
| LINE_CHANNEL_SECRET | ✅ | - | LINE Channel Secret |
| ANTHROPIC_API_KEY | △ | - | Anthropic API 金鑰 |
| DEEPSEEK_API_KEY | ✗ | - | DeepSeek API 金鑰 |
| GOOGLE_API_KEY | ✗ | - | Google AI API 金鑰 |
| QWEN_API_KEY | ✗ | - | Qwen API 金鑰 |
| NGROK_AUTHTOKEN | ✗ | - | ngrok 認證 Token |
| OPENCODE_PASSWORD | ✗ | changeme | OpenCode 伺服器密碼 |
| PROMPT_TIMEOUT_MS | ✗ | 120000 | Prompt 逾時時間(ms) |
| GITHUB_TOKEN | ✗ | - | GitHub Token (選填) |

> △ = 至少需要一個 AI Provider 金鑰（或使用內建 Big Pickle 免費模型）

### C. 變更記錄

| 版本 | 日期 | 變更說明 |
|------|------|---------|
| v1.0 | 2026-06-20 | 初版完成 |

---

*本報告由 OpenCode AI 輔助產生*
