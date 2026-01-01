# LLM-first：所有文字訊息都交給 Gemini 回覆（LangChain）

## 🎯 目標
不再用「特殊文字指令」觸發。
只要使用者傳任何文字訊息，LINE Bot 都會把文字送去 Gemini（LangChain）並回覆 Gemini 生成的內容。

---

## ✅ 成果行為
- 使用者傳任意文字 → Bot 回覆 Gemini 的回應
- 不需要輸入 `gemini` 指令
- Webhook 一樣維持簽章驗證與 Reply API 回覆

---

## 🧠 架構差異：Command-based vs LLM-first

### A. 指令觸發（上一階段）
- 只有「特定文字」才會呼叫 Gemini
- 優點：省 token、較可控
- 缺點：使用者要記指令

### B. LLM-first（本篇）
- 所有文字都丟給 Gemini
- 優點：像真正聊天機器人、使用者體驗自然
- 缺點：成本較高、要做風險控管（限長、過濾、timeout）

---

## 🔁 流程（MVP）
LINE 使用者 → Webhook → 解析 events → 取得 user_text → getGeminiReply(user_text) → Reply API 回覆

---

## 🧩 核心邏輯（節錄）

```python
if msg.get("type") == "text":
    user_text = msg.get("text", "").strip()
    if user_text:
        gemini_reply = getGeminiReply(user_text)
        payload = {
            "replyToken": event["replyToken"],
            "messages": [makeTextMessage(gemini_reply)]
        }
        replyMessage(payload)
```
## 🤖 Gemini（LangChain）整合

### 初始化

- API key 放在 `config.ini`
- 使用 `ChatGoogleGenerativeAI`
- `llm.invoke(prompt)` 取得回覆

```python
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
    api_key=GEMINI_API_KEY,
    temperature=0.7,
)
```
### 回覆函式（含 system prompt）
```python
prompt = f"""你是一個友善、幽默且樂於助人的 AI 助手。
請根據使用者的訊息，給出適當且有幫助的回覆。

使用者訊息：{user_message}"""
msg = llm.invoke(prompt)
return msg.content
```
### 下一步」的建議方向：
🎯**加輸入長度限制**（例如 > 300 字就先請他簡短）
🎯 **把 system prompt 抽成常數**（避免 prompt 寫散）


