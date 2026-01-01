# Sticker Message（貼圖訊息）

本筆記紀錄 LINE Bot 中「貼圖訊息（Sticker Message）」的
**最小可行格式、實作方式與常見錯誤**。

目前狀態：✅ 已實際測試成功  
觸發指令：「出去玩囉」 → 回傳貼圖

---

## 一、Sticker Message 的用途

Sticker Message 用於：
- 快速回應使用者（情緒型回覆）
- 指令成功的視覺回饋
- Bot 的互動感提升

貼圖本身不含文字，**只要 payload 正確即可回傳**。

---

## 二、Sticker Message 的最小 payload

### 必要欄位（缺一不可）

```json
{
  "type": "sticker",
  "packageId": "446",
  "stickerId": "1988"
}
```


### 三、Python 實作（建立貼圖訊息物件)

```python
def getPlayStickerMessage():
    """
    建立貼圖訊息物件
    packageId 和 stickerId 可參考 LINE 官方貼圖列表
    https://developers.line.biz/en/docs/messaging-api/sticker-list/
    """
    message = {
        "type": "sticker",
        "packageId": "446",
        "stickerId": "1988"
    }
    return message
```

此 function 的設計目標：

- 與 Webhook / 指令邏輯分離
    
- 方便之後替換不同貼圖
    
- 作為「訊息元件」被呼叫

### 四、搭配 Reply API 使用
Sticker Message 必須包在 `messages` 陣列中，並搭配 `replyToken`。
```python
payload = {
    "replyToken": event["replyToken"],
    "messages": [getPlayStickerMessage()]
}

replyMessage(payload)

```
### 五、實際觸發範例（已驗證）
```python
if event.get("type") == "message":
	msg = event.get("message", {})
    if msg.get("type") == "text":
	    text = msg.get("text", "")

	        # 當使用者傳送「出去玩囉」時，回傳貼圖
	        if text == "出去玩囉":
	            payload = {
                        "replyToken": event["replyToken"],
                        "messages": [getPlayStickerMessage()]
                    }
                    replyMessage(payload)
```
