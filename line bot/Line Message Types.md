# LINE Messaging API 訊息類型 (Message Types)

> [!ABSTRACT]
> 本篇筆記整理了 LINE Messaging API 支援的各種訊息格式類型（Message Types），包含基礎訊息、多媒體訊息，以及高互動性的樣板與 Flex Message，幫助開發者針對不同場景選擇最合適的互動方式。

---

## 一、 基礎訊息類型 (Basic Messages)

### 1. 文字訊息 (Text Message)
- **說明**：最基本的訊息格式，支援純文字、表情符號（Emoji）以及超連結。
- **限制**：單則訊息最大字數限制為 5,000 字。
- **JSON 範例**：
  ```json
  {
    "type": "text",
    "text": "哈囉，這是一則文字訊息！"
  }
  ```

### 2. 貼圖訊息 (Sticker Message)
- **說明**：發送 LINE 官方提供的貼圖。
- **限制**：開發者只能發送 LINE 官方指定清單內的貼圖（需指定 `packageId` 與 `stickerId`），無法發送使用者購買的第三方貼圖。
- **JSON 範例**：
  ```json
  {
    "type": "sticker",
    "packageId": "446",
    "stickerId": "1988"
  }
  ```

### 3. 位置訊息 (Location Message)
- **說明**：發送含有地址、經緯度與標題的地圖定位資訊。
- **常用情景**：提供門市位置、活動地點導航。
- **JSON 範例**：
  ```json
  {
    "type": "location",
    "title": "台北 101",
    "address": "110台北市信義區信義路五段7號",
    "latitude": 25.033964,
    "longitude": 121.564468
  }
  ```

---

## 二、 多媒體訊息類型 (Multimedia Messages)

### 1. 圖片訊息 (Image Message)
- **說明**：發送單張圖片。
- **限制**：
  - 必須提供圖片的 HTTPS URL。
  - 需要同時提供原圖連結（`originalContentUrl`，最大 10MB）與預覽圖連結（`previewImageUrl`，最大 1MB）。
- **JSON 範例**：
  ```json
  {
    "type": "image",
    "originalContentUrl": "https://example.com/original.jpg",
    "previewImageUrl": "https://example.com/preview.jpg"
  }
  ```

### 2. 影片訊息 (Video Message)
- **說明**：發送可播放的影片。
- **限制**：
  - 需要提供影片檔案連結（最大 200MB）與預覽圖片連結。
  - 支援格式通常為 MP4。
- **JSON 範例**：
  ```json
  {
    "type": "video",
    "originalContentUrl": "https://example.com/original.mp4",
    "previewImageUrl": "https://example.com/preview.jpg"
  }
  ```

### 3. 語音訊息 (Audio Message)
- **說明**：發送音訊檔，使用者可在對話框內直接點擊播放。
- **限制**：支援 M4A 格式，音訊長度限制最大為 1 分鐘，檔案大小最大為 200MB。
- **JSON 範例**：
  ```json
  {
    "type": "audio",
    "originalContentUrl": "https://example.com/original.m4a",
    "duration": 60000
  }
  ```

---

## 三、 進階與高互動性訊息類型 (Advanced Messages)

### 1. 影像地圖訊息 (Imagemap Message)
- **說明**：發送一張大圖，並在圖片上定義多個「可點擊區域」（Hotspots）。當使用者點擊不同區域時，可觸發開啟網頁連結（Link action）或發送特定文字（Message action）。
- **常用情景**：主選單、多選項導覽。

### 2. 樣板訊息 (Template Message)
提供預設排版形式的互動訊息，比一般文字更具結構性。包含以下四種：
- **Buttons (按鈕樣板)**：圖片 + 標題 + 內文 + 多個動作按鈕。
- **Confirm (確認樣板)**：簡單的二選一問答（例如：是/否、同意/不同意）。
- **Carousel (輪播樣板)**：橫向滑動的按鈕卡片組，適合展示商品列表。
- **Image Carousel (圖片輪播樣板)**：橫向滑動的圖片卡片，點擊圖片即可觸發動作。

- **JSON 範例 (以 Buttons 按鈕樣板為例)**：
  ```json
  {
    "type": "template",
    "altText": "這是一則按鈕樣板訊息",
    "template": {
      "type": "buttons",
      "thumbnailImageUrl": "https://example.com/image.jpg",
      "imageAspectRatio": "rectangle",
      "imageSize": "cover",
      "imageBackgroundColor": "#FFFFFF",
      "title": "功能選單",
      "text": "請選擇您要執行的動作",
      "actions": [
        {
          "type": "postback",
          "label": "立即購買",
          "data": "action=buy&itemid=123"
        },
        {
          "type": "uri",
          "label": "查看詳情",
          "uri": "https://example.com/page/123"
        }
      ]
    }
  }
  ```

### 3. Flex Message (客製樣板訊息)
- **說明**：基於 CSS Flexible Box (Flexbox) 概念設計的訊息格式。開發者可以透過 JSON 結構自由設計版面，包含文字大小、顏色、按鈕排版、圖片區塊等，具備極高的客製化彈性。
- **優勢**：外觀精美且現代化，是目前 LINE 機器人開發最推薦的主力訊息格式。
- **實用工具**：可使用 LINE 官方提供的 [Flex Message Simulator](https://developers.line.biz/flex-simulator/) 進行視覺化設計並導出 JSON。
- **JSON 範例 (基礎 Bubble 結構)**：
  ```json
  {
    "type": "flex",
    "altText": "這是一則 Flex Message",
    "contents": {
      "type": "bubble",
      "body": {
        "type": "box",
        "layout": "vertical",
        "contents": [
          {
            "type": "text",
            "text": "主標題",
            "weight": "bold",
            "size": "xl"
          },
          {
            "type": "text",
            "text": "副標題或內文說明...",
            "size": "sm",
            "color": "#666666",
            "margin": "md"
          }
        ]
      }
    }
  }
  ```
