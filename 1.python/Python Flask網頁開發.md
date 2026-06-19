# Python Flask 網頁開發入門指引

> [!ABSTRACT]
> 本章介紹 Python 輕量級網頁框架 Flask 的基礎架構，涵蓋路由系統、HTTP 請求處理、Jinja2 模板引擎渲染，以及 RESTful API 響應（JSON）的實作方法，幫助開發者快速建構與部署 Web 應用。

---

## 一、 什麼是 Flask？

**Flask** 是基於 Python 撰寫的微型網頁框架（Micro Framework）。它之所以被稱為「微型」，是因為它保持了核心的輕量與簡單，不強制要求特定的資料庫套件或目錄結構，給予開發者極大的自由度。

### 1. 安裝指令
使用 pip 安裝 Flask 套件：
```bash
pip install flask
```

### 2. 極簡 Flask 應用程式
建立一個 `app.py` 檔案並寫入以下程式碼：

```python
from flask import Flask

# 初始化 Flask 應用程式
app = Flask(__name__)

# 定義路由 (Route) 與對應的視圖函式 (View Function)
@app.route("/")
def home():
    return "歡迎來到 Flask 網頁世界！"

# 當直接執行此檔案時啟動伺服器
if __name__ == "__main__":
    # debug=True 可在代碼修改時自動重啟，並在網頁上顯示詳細錯誤訊息
    app.run(debug=True, port=5000)
```

---

## 二、 路由系統與動態 URL 詳解

路由用於將特定的網址（URL）對應到 Python 的視圖處理函式（View Function）。

### 1. 基礎路由與唯一網址規則
Flask 的路由尾端斜線（Slash）設定會影響瀏覽器的連線行為：
```python
# 規則 A：有尾端斜線
@app.route("/about/")
def about():
    return "關於頁面"

# 規則 B：無尾端斜線
@app.route("/contact")
def contact():
    return "聯絡頁面"
```
> [!IMPORTANT]
> **尾端斜線匹配原則**
> - 對於**規則 A** (`/about/`)：當使用者輸入 `/about` 時，Flask 會**自動重導向**至 `/about/`，確保網址唯一性。
> - 對於**規則 B** (`/contact`)：當使用者輸入 `/contact/` 時，將會引發 **404 Not Found** 錯誤。建議統一規劃網址命名風格。

### 2. 動態變數路由與轉換器 (Converters)
可在網址中嵌入變數，格式為 `<型態:變數名>`。Flask 內建五種常用的變數轉換器：

| 轉換器名稱 | 說明 | 範例 |
| :--- | :--- | :--- |
| **`string`** | 預設型態，接受任何不包含斜線 `/` 的字串 | `/user/<username>` |
| **`int`** | 只接受正整數 | `/post/<int:post_id>` |
| **`float`** | 只接受正浮點數 | `/price/<float:value>` |
| **`path`** | 類似 string，但接受包含斜線 `/` 的完整路徑 | `/files/<path:filepath>` |
| **`uuid`** | 只接受標準 UUID 格式字串 | `/api/<uuid:user_id>` |

```python
# 示範路徑與整數轉換器
@app.route("/download/<path:filepath>")
def download_file(filepath):
    return f"正在下載檔案，路徑為: {filepath}"
```

### 3. HTTP 方法綁定與新型裝飾器
除了使用 `@app.route(methods=["POST"])` 之外，新版 Flask（自 2.0 起）提供了更簡潔的 HTTP 方法裝飾器：
```python
# 傳統寫法
@app.route("/submit", methods=["POST"])
def submit_legacy():
    return "提交成功"

# 新型語法糖快捷寫法
@app.get("/items")
def list_items():
    return "取得所有項目列表 (GET)"

@app.post("/items")
def create_item():
    return "新增單一項目 (POST)"
```

### 4. 動態網址建置與重導向 (url_for & redirect)
在實際開發中，**絕不建議在程式碼中硬編碼網址**，應使用 `url_for()` 依據**函數名稱**動態生成 URL。這能避免因網址變更而導致的斷鏈問題。

```python
from flask import Flask, redirect, url_for

app = Flask(__name__)

@app.route("/admin")
def admin_panel():
    return "歡迎進入管理員後台"

@app.route("/guest/<guest_name>")
def guest_hello(guest_name):
    return f"訪客 {guest_name}，您好"

@app.route("/user/<name>")
def login_redirect(name):
    if name == "admin":
        # 1. 重導向至 admin_panel() 對應的路由網址
        return redirect(url_for("admin_panel"))
    else:
        # 2. 重導向時傳遞動態路由參數 (guest_name)
        return redirect(url_for("guest_hello", guest_name=name))
```

---

## 三、 處理 HTTP 請求 (GET / POST)

Flask 可以透過 `request` 物件來接收並處理客戶端傳送過來的資料。

```python
from flask import Flask, request

app = Flask(__name__)

# 路由預設只接收 GET 請求，若要處理 POST 需在 methods 參數中宣告
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        # 1. 接收來自 HTML 表單 (Form Data) 的資料
        username = request.form.get("username")
        password = request.form.get("password")
        return f"登入成功！歡迎 {username}"
    else:
        # 2. 接收來自網址參數 (Query Parameters，例如 /login?ref=google)
        ref_source = request.args.get("ref", "未知來源")
        return f"請登入 (來源: {ref_source})"
```

---

## 四、 模板引擎與網頁渲染 (Jinja2)

Flask 內建 **Jinja2** 模板引擎，它能將 Python 變數與邏輯帶入 HTML 檔案中動態生成網頁內容。

### 1. Jinja2 四大核心語法標記 (Delimiters)

為了在 HTML 檔案中區分純文字與模板控制語句，Jinja2 使用了四種不同的標記符號，這是最核心的語法：

| 語法標記 | 功能定義 | 說明與範例 |
| :--- | :--- | :--- |
| **`{{ 變數或表達式 }}`** | **資料輸出 (Output)** | 將變數值印在網頁上。例如：`{{ title }}` 會印出變數內容，也可以進行運算：`{{ score + 5 }}`。 |
| **`{% 控制語句 %}`** | **邏輯控制 (Control)** | 用於執行條件判斷（`if`）、迴圈（`for`）或模板繼承。注意：邏輯語句必須成對結束，例如 `{% endif %}`、`{% endfor %}`。 |
| **`{# 模板註解 #}`** | **內部註解 (Comment)** | Jinja2 的專屬註解。這些註解**不會**出現在最終生成的網頁 HTML 原始碼中，安全且適合寫開發備忘錄。 |
| **`{{ 變數 \| 過濾器 }}`** | **資料過濾 (Filter)** | 使用管道符號 `\|` 修改輸出格式。例如：`{{ name \| upper }}` 將字串轉大寫，`{{ text \| safe }}` 允許直接渲染 HTML。 |

---

### 2. 目錄結構規範
Flask 要求 HTML 模板檔案必須放置在專案根目錄底下的 `templates/` 資料夾中：
```text
專案目錄/
├── app.py
└── templates/
    └── index.html
```

### 3. 視圖函式與渲染
在 `app.py` 中呼叫 `render_template` 並傳入參數：

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/index")
def index():
    items = ["蘋果", "香蕉", "橘子"]
    return render_template("index.html", title="首頁", fruits=items)
```

### 4. HTML 模板範本 (`templates/index.html`)

```html
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <title>{{ title }}</title>
</head>
<body>
    <h1>水果清單</h1>
    <ul>
        <!-- 使用 Jinja2 的控制流程語法 -->
        {% for fruit in fruits %}
            <li>{{ fruit }}</li>
        {% else %}
            <li>目前無水果資訊</li>
        {% endfor %}
    </ul>
</body>
</html>
```

> [!TIP]
> **靜態檔案掛載**
> 網頁所需的 CSS、JavaScript 及圖片等靜態檔案，必須存放在專案目錄下的 `static/` 資料夾中。在 HTML 中可以使用以下語法引入：
> `<link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">`

---

## 五、 回傳 JSON 資料 (RESTful API)

如果您需要開發前後端分離的應用程式或 Web API，可以使用 `jsonify` 函式將 Python 字典直接轉為 JSON 格式回傳。

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/api/v1/status")
def get_status():
    data = {
        "status": "success",
        "code": 200,
        "message": "系統正常運行中",
        "features": ["routing", "templates", "json_api"]
    }
    # jsonify 會自動將字典轉為 JSON 字串，並在 HTTP Header 中設定 Content-Type: application/json
    return jsonify(data)
```

---

## 六、 💡 Obsidian 檢索與關聯小提示
- **[[docker compose 容器編排指引]]**：了解如何使用 Docker Compose 一鍵部署 Flask 網頁服務與 Redis 快取的雙容器系統。
- **[[1. 導論]] (網路爬蟲)**：了解網頁請求的運作機制，Flask 可做為測試您所寫爬蟲的本機伺服器。
