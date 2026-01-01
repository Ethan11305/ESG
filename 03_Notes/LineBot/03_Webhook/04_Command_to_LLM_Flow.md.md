# 指令觸發 → Gemini 回覆（LangChain）

## 🎯 本篇目標
當使用者在 LINE Bot 中輸入「特定文字指令」時，
系統會呼叫 Gemini（透過 LangChain）產生回覆，並將結果回傳給使用者。

---

## 🧠 整體流程（MVP）

使用者（LINE）
→ LINE Messaging API
→ Webhook（Flask）
→ 指令判斷（text）
→ Gemini API（LangChain）
→ Reply API
→ 使用者收到回覆

---

## 🧩 觸發條件設計

目前支援的指令：

| 使用者輸入 | 系統行為 |
|----------|--------|
| 出去玩囉 | 回傳貼圖 |
| gemini / Gemini | 呼叫 Gemini 產生文字回覆 |

文字處理原則：
- 使用 `.strip()` 去除空白
- 使用 `.lower()` 忽略大小寫

---

## 🧩 核心程式邏輯（節錄）

```python
text = (msg.get("text", "") or "").strip()

if text == "出去玩囉":
    payload = {
        "replyToken": event["replyToken"],
        "messages": [getPlayStickerMessage()]
    }
    replyMessage(payload)
    return "ok"

elif text.lower() == "gemini":
    gemini_text = getGeminiReply(
        "朋友說『你好』時，用輕鬆有趣的方式回一句"
    )
    payload = {
        "replyToken": event["replyToken"],
        "messages": [makeTextMessage(gemini_text)]
    }
    replyMessage(payload)
    return "ok"
```

## 🤖 Gemini 整合方式（LangChain）

- 使用 `ChatGoogleGenerativeAI`
- API Key 由 `config.ini` 管理
- 不依賴環境變數
- 使用 `llm.invoke(prompt)` 取得回覆文字

```python
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    api_key=GEMINI_API_KEY,
    temperature=0.7,
)
```

