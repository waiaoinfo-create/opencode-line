# OpenCode LINE Bot 建置指南

## 系統架構

```
LINE App → ngrok Tunnel → line-bot (Bun, :3000) → OpenCode Server (:4096) → AI Model
                                                              ↕
                                                        MCP Tools
```

## 事前準備

你需要準備以下 3 樣東西：

### 1. LINE 官方帳號憑證

1. 前往 [LINE Developers Console](https://developers.line.biz/console/)
2. 登入後建立 **Provider**（如果還沒有）
3. 建立 **Messaging API Channel**
4. 在 Channel 設定頁取得：
   - `Channel Secret`（在 Basic Settings 頁）
   - `Channel Access Token`（在 Messaging API 頁 → Issue）

### 2. AI API 金鑰（擇一即可）

| AI 服務 | 費用 | 申請網址 |
|---------|------|---------|
| OpenCode Big Pickle | **免費** | 內建，不需金鑰 |
| Anthropic Claude | $ | https://console.anthropic.com |
| OpenAI GPT | $$ | https://platform.openai.com |
| Google Gemini | $ | https://aistudio.google.com |
| DeepSeek | $ | https://platform.deepseek.com |

### 3. ngrok 帳號（選填但建議）

1. 前往 https://ngrok.com 註冊免費帳號
2. 登入後在 Dashboard 複製 **Authtoken**
3. 填入 `.env` 的 `NGROK_AUTHTOKEN`
   - 有填：ngrok URL 固定不變
   - 沒填：每次重啟 URL 都會變，需重設 LINE Webhook

## 快速部署

### 第一步：設定環境變數

複製 `.env.example` 為 `.env`，填入你的憑證：

```env
# LINE 憑證
LINE_CHANNEL_ACCESS_TOKEN=你的Channel Access Token
LINE_CHANNEL_SECRET=你的Channel Secret
LINE_OA_URL=https://line.me/ti/p/~你的官方帳號ID

# AI 金鑰（至少填一個）
ANTHROPIC_API_KEY=sk-ant-你的金鑰

# ngrok（選填）
NGROK_AUTHTOKEN=你的ngrok authtoken
```

### 第二步：啟動服務

```bash
docker compose up -d --build
```

### 第三步：設定 LINE Webhook

1. 取得 ngrok 公開網址：
   ```bash
   docker logs opencode-line-ngrok
   ```
   看到類似 `https://xxxx-xxx-xxx-xxx.ngrok-free.app` 的網址

2. 到 LINE Developers Console → Messaging API 頁
3. 在 **Webhook URL** 填入：
   ```
   https://你的ngrok網址/webhook
   ```
4. 開啟 **Webhook** 開關
5. 點 **Verify** 測試連線

### 第四步：驗證

傳送訊息給你的 LINE 官方帳號，應該會收到 AI 回覆。

## 常用指令

| LINE 指令 | 功能 |
|-----------|------|
| `/new` | 開始新對話 |
| `/model` | 查看目前 AI 模型 |
| `/model sonnet` | 切換到 Claude Sonnet |
| `/model pickle` | 切換到免費模型 |
| `/help` | 顯示所有指令 |
| `/sessions` | 查看對話狀態 |

## 可用 AI 模型

| 指令 | 模型 | 供應商 | 費用 |
|------|------|--------|------|
| `/model pickle` | Big Pickle | opencode | **免費** |
| `/model deepseek` | DeepSeek Chat | deepseek | $ |
| `/model haiku` | Claude Haiku 4.5 | anthropic | $ |
| `/model sonnet` | Claude Sonnet 4.6 | anthropic | $$ |
| `/model gpt5mini` | GPT-5 Mini | openai | $ |
| `/model geminiflash` | Gemini 3 Flash | google | $ |

## Docker 服務

| 服務 | 說明 | 連接埠 |
|------|------|--------|
| opencode-server | OpenCode AI 引擎 | :4096 |
| opencode-line-bot | LINE Bot Webhook | :3000 |
| opencode-line-ngrok | 公開 Tunnel | 動態 |

## 查看日誌

```bash
# LINE Bot 日誌
docker logs opencode-line-bot --tail 30 -f

# OpenCode 日誌
docker logs opencode-server --tail 30 -f

# ngrok 日誌（查看公開網址）
docker logs opencode-line-ngrok
```

---

> 原始專案：https://github.com/monthop-gmail/opencode-line
