## 跳過網路 安裝
在安裝畫面停在 「讓我們連線到網際網路」 的那一頁：
`Shift + F10` → 會跳出黑色 CMD 視窗
```sh
OOBE\BYPASSNRO
```
Windows 會自動重新開機。

回到同一頁時，下方會多出一個 「我沒有網際網路」
點它 ➜ 再點 「繼續進行受限設定」。

## （備用）
```sh
taskkill /F /IM oobenetworkconnectionflow.exe
```

## 啟用office
```sh
irm https://get.activated.win/ | iex
```
