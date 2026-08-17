## 闔上螢幕不休眠
```sh
sudo nano /etc/systemd/logind.conf
```
```sh
HandleLidSwitch=ignore               # 使用電池=闔蓋時完全不理會這個動作
HandleLidSwitchExternalPower=ignore  # 使用電源=闔蓋時完全不理會這個動作
HandleLidSwitchDocked=ignore         # 已連接外部顯示器=闔蓋時完全不理會這個動作
```
重啟服務
```
sudo systemctl restart systemd-logind
```

