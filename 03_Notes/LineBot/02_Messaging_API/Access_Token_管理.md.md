# 🔐 Access Token 管理說明

> 本文件僅記錄 **Token 的種類、用途、存放方式與安全規範**  
> ❌ **不存放任何實際 Token 值**

---

## 1️⃣ 為什麼要集中管理 Token？

在專案中集中管理 Access Token，目的在於：

- 避免將金鑰寫死在程式碼中（Hard Code）
- 防止誤上傳至 GitHub 造成金鑰外洩
- 方便在不同環境（本機 / 測試 / 正式）切換
- 符合實務上的 Security Best Practice
- 降低專案維護與交接成本

---

## 2️⃣ Token 儲存方式（目前採用）

### ✅ 使用 `config.ini`

```ini
[line-bot]
channel_access_token = ********
channel_secret = ********
gemini_api_key = ********
```

📌 原則：
- `config.ini` **不進 Git*
- 由 `configparser` 在程式中讀取
- 實際 Token 僅存在於「開發機 / 伺服器」

---
## 3️⃣ 程式中讀取方式（範例）
```python
import configparser

config = configparser.ConfigParser()
config.read("config.ini", encoding="utf-8")

LINE_TOKEN = config.get("line-bot", "channel_access_token")
LINE_SECRET = config.get("line-bot", "channel_secret")
GEMINI_KEY = config.get("line-bot", "gemini_api_key")

```
---
## 4️⃣ Token 類型與用途對照表

|Token 名稱|用途|使用位置|
|---|---|---|
|channel_access_token|LINE 回傳訊息 API|HTTP Header|
|channel_secret|LINE Webhook 驗章|HMAC 驗證|
|gemini_api_key|Gemini LLM 存取|LLM Client|

---
## 5️⃣ Git 安全設定
✅ clone 專案後需要補 `config.ini`
✅`config.ini` 必須加入 `.gitignore`

