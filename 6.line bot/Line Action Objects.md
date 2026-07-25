# LINE Action Objects (動作物件)

> [!ABSTRACT]
> 本篇筆記整理了 LINE Messaging API 中各種 **Action Objects (動作物件)** 的類型、用途與 JSON 設定範例。Action Objects 常用於按鈕、樣板、Flex Message、Quick Replies 以及 Rich Menus 中，定義使用者點擊後的互動行為。

---

## 什麼是 Action Objects？
當使用者點擊 LINE 訊息中的按鈕、圖片或選單時，LINE 會根據該區塊所綁定的 **Action Object** 執行相對應的動作（例如：發送文字、開啟網頁、彈出時間選擇器等）。

---

## 核心 Action 類型詳解

### 1. Postback Action (背景傳送動作)
- **用途**：最重要且常用的動作。使用者點擊後，**不會**在聊天視窗中發送任何文字，但 LINE 會直接發送一個 Webhook 事件（包含自定義的 `data`）至開發者的 Webhook URL。
- **優勢**：適合進行背景邏輯處理（如選取商品規格、換頁、點擊確認），避免聊天視窗被無意義的系統指令洗版。
- **JSON 範例**：
  ```json
  {
    "type": "postback",
    "label": "選擇此規格",
    "data": "action=select&item_id=101&color=red",
    "displayText": "您已選擇紅色規格" 
  }
  ```
  *(註：`displayText` 是可選欄位，設定後會在聊天視窗顯示使用者點擊後的氣泡訊息。)*

---

### 2. Message Action (訊息動作)
- **用途**：最直覺的動作。使用者點擊後，會以使用者的身分在對話框內**直接發送一則設定好的文字訊息**。
- **常用情景**：常見的 FAQ 常見問題點擊（如：點擊「聯絡客服」按鈕，直接自動發送「我想聯絡客服」文字給機器人）。
- **JSON 範例**：
  ```json
  {
    "type": "message",
    "label": "聯絡客服",
    "text": "我要聯絡客服"
  }
  ```

---

### 3. URI Action (網址動作)
- **用途**：點擊後直接在 LINE 內部瀏覽器（或外部瀏覽器）**開啟指定的網址**，亦支援 LINE 專屬的 URL Scheme（如開啟相機、個人檔案等）。
- **JSON 範例**：
  ```json
  {
    "type": "uri",
    "label": "前往官網",
    "uri": "https://example.com"
  }
  ```

---

### 4. Datetime Picker Action (日期時間選擇器)
- **用途**：點擊後會彈出 LINE 原生的日期/時間選擇視窗。使用者選定後，系統會將選定的時間以 Postback 事件發送回開發者的伺服器。
- **模式 (`mode`)**：
  - `date`：僅選擇日期
  - `time`：僅選擇時間
  - `datetime`：選擇日期與時間
- **JSON 範例**：
  ```json
  {
    "type": "datetimepicker",
    "label": "預約時間",
    "data": "action=reserve&room_id=5",
    "mode": "datetime",
    "initial": "2026-07-02T14:00",
    "max": "2026-12-31T23:59",
    "min": "2026-01-01T00:00"
  }
  ```

---

## 僅限 Quick Reply 支援的 Action 類型
以下三種 Actions **只能** 用在 **Quick Reply (快速回覆)** 的按鈕中：

### 5. Camera Action (開啟相機)
- **用途**：點擊後直接在 LINE 中打開相機，方便使用者拍照並傳送給機器人。
- **JSON 範例**：
  ```json
  {
    "type": "camera",
    "label": "開啟相機拍照"
  }
  ```

### 6. Camera Roll Action (開啟相片庫)
- **用途**：點擊後直接在 LINE 中打開手機相簿，供使用者挑選照片傳送。
- **JSON 範例**：
  ```json
  {
    "type": "cameraRoll",
    "label": "從相簿選取照片"
  }
  ```

### 7. Location Action (傳送位置)
- **用途**：點擊後彈出 LINE 原生地圖定位畫面，讓使用者選取並傳送當前的位置。
- **JSON 範例**：
  ```json
  {
    "type": "location",
    "label": "分享我的位置"
  }
  ```
