# LINE 樣板訊息 (Template Messages)

> [!ABSTRACT]
> 本篇筆記詳細介紹 LINE Messaging API 中四種主流「樣板訊息」（Template Messages）的用途、互動欄位與完整 JSON 配置範例，以幫助開發者設計標準化的按鈕與輪播互動介面。

---

## 關於樣板型訊息
樣板型訊息（Template Messages）可以透過綁定 LINE 預設的按鈕、文字與圖片版型，限縮與引導使用者的操作流程，提供流暢的互動體驗。
> [!NOTE]
> 樣板訊息必須提供 `altText`（替代文字），當使用者的 LINE 版本不支援該樣板或在聊天列表預覽時，會顯示此替代文字。

---

## 一、 四大樣板訊息詳解與 JSON 範例

### 1. 按鈕樣板 (Buttons Template)
- **用途**：最常用的樣板，適合用於展示單一商品、功能選單，包含一張主圖、一個標題、一段內文以及最多 4 個動作按鈕。
- **欄位限制**：
  - 動作按鈕（`actions`）最多 4 個。
- **JSON 範例**：
  ```json
  {
    "type": "template",
    "altText": "這是一則按鈕樣板訊息",
    "template": {
      "type": "buttons",
      "thumbnailImageUrl": "https://example.com/bot/images/image.jpg",
      "imageAspectRatio": "rectangle",
      "imageSize": "cover",
      "imageBackgroundColor": "#FFFFFF",
      "title": "主功能選單",
      "text": "請選擇您要進行的操作",
      "defaultAction": {
        "type": "uri",
        "label": "查看詳細網頁",
        "uri": "https://example.com/page/123"
      },
      "actions": [
        {
          "type": "postback",
          "label": "立即購買",
          "data": "action=buy&itemid=123"
        },
        {
          "type": "postback",
          "label": "加入購物車",
          "data": "action=add&itemid=123"
        },
        {
          "type": "uri",
          "label": "前往官網",
          "uri": "https://example.com/page/123"
        }
      ]
    }
  }
  ```

---

### 2. 確認樣板 (Confirm Template)
- **用途**：專為「二選一」問答設計（例如：是/否、同意/不同意、確定/取消）。
- **欄位限制**：
  - 只能提供一個說明文字（`text`）與剛好 2 個動作按鈕。
  - 不支援圖片展示。
- **JSON 範例**：
  ```json
  {
    "type": "template",
    "altText": "這是一則確認樣板訊息",
    "template": {
      "type": "confirm",
      "text": "您確定要刪除這筆訂單嗎？",
      "actions": [
        {
          "type": "message",
          "label": "確定",
          "text": "確定刪除"
        },
        {
          "type": "message",
          "label": "取消",
          "text": "取消操作"
        }
      ]
    }
  }
  ```

---

### 3. 輪播樣板 (Carousel Template)
- **用途**：由多個「按鈕樣板」卡片橫向排列組合而成，使用者可以左右滑動切換。非常適合展示商品列表、推薦內容。
- **欄位限制**：
  - 卡片欄位（`columns`）最多支援 10 張卡片。
  - 每張卡片的動作按鈕（`actions`）數量必須相同，且最多只能設定 3 個。
- **JSON 範例**：
  ```json
  {
    "type": "template",
    "altText": "這是一則輪播樣板訊息",
    "template": {
      "type": "carousel",
      "columns": [
        {
          "thumbnailImageUrl": "https://example.com/bot/images/item1.jpg",
          "imageBackgroundColor": "#FFFFFF",
          "title": "熱銷商品 A",
          "text": "超高人氣，限時 8 折優惠中！",
          "defaultAction": {
            "type": "uri",
            "label": "查看商品",
            "uri": "https://example.com/item/111"
          },
          "actions": [
            {
              "type": "postback",
              "label": "立即購買",
              "data": "action=buy&itemid=111"
            },
            {
              "type": "uri",
              "label": "查看細節",
              "uri": "https://example.com/item/111"
            }
          ]
        },
        {
          "thumbnailImageUrl": "https://example.com/bot/images/item2.jpg",
          "imageBackgroundColor": "#FFFFFF",
          "title": "人氣商品 B",
          "text": "獨家新品，限量發售！",
          "defaultAction": {
            "type": "uri",
            "label": "查看商品",
            "uri": "https://example.com/item/222"
          },
          "actions": [
            {
              "type": "postback",
              "label": "立即購買",
              "data": "action=buy&itemid=222"
            },
            {
              "type": "uri",
              "label": "查看細節",
              "uri": "https://example.com/item/222"
            }
          ]
        }
      ],
      "imageAspectRatio": "rectangle",
      "imageSize": "cover"
    }
  }
  ```

---

### 4. 圖片輪播樣板 (Image Carousel Template)
- **用途**：與輪播樣板類似，但每張卡片「純圖片」展示，沒有標題與文字描述，點擊整張圖片即可觸發動作。適用於廣告 Banner、視覺導向的選單。
- **欄位限制**：
  - 卡片欄位（`columns`）最多支援 10 張卡片。
  - 每張卡片只能綁定 1 個點擊動作（`action`）。
- **JSON 範例**：
  ```json
  {
    "type": "template",
    "altText": "這是一則圖片輪播樣板訊息",
    "template": {
      "type": "image_carousel",
      "columns": [
        {
          "imageUrl": "https://example.com/bot/images/banner1.jpg",
          "action": {
            "type": "uri",
            "label": "查看活動 1",
            "uri": "https://example.com/promo/1"
          }
        },
        {
          "imageUrl": "https://example.com/bot/images/banner2.jpg",
          "action": {
            "type": "postback",
            "label": "領取折價券",
            "data": "action=coupon&id=free100"
          }
        }
      ]
    }
  }
  ```