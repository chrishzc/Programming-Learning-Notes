# 申請line bot 
line的系統裡，不同的服務是用channels做區別
1. 進入line developers
2. 右上角 使用LINE帳號登入console
3. Createa new providers
4. Createa Messaging API channel
5. 建立官方帳號(現在開機器人都要開設官方帳號)
6. 在官方帳號管理介面選擇messaging API
7. Createa Messaging API(如果有涉及金流，要有隱私、服務條款)
8. 設定好後後續設定可以回到 LINE Developers Console
# 設定Messaging API
1.  Channel access token: 機器人API token，issue後顯示，reissue 可以重置
2. Security: 可以使用IP，非IP位置不能管理
3. Roles: 可以新增其他管理者
# ngrok 
一鍵佈署、測試網站。省略網路佈署:IP、port、domain等等程序
1. 下載安裝後，輸入官網給的token
2. ngrok http "port號碼"得到公開網址就可以用了
# 鸚鵡答應機
使用老師給的app.py，裡面的/callback函式
1. 設定config.ini中的channel_access_token、channel_secret
2. 設定Messaging API中的Webhook URL==(ngrok網址+/callback)==並verify success
3. 開啟Webhook功能
4. ==此時傳送文字訊息給官方帳號，官方帳號會變成鸚鵡答應機==
![[{A6758E78-4A34-453B-B350-AFCC614D2AEF}.png]]
# 