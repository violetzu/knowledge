## 闔上螢幕不休眠
```sh
sudo mkdir -p /etc/systemd/logind.conf.d
sudo nano /etc/systemd/logind.conf.d/lid.conf
```
```sh
HandleLidSwitch=ignore               # 使用電池=闔蓋時完全不理會這個動作
HandleLidSwitchExternalPower=ignore  # 使用電源=闔蓋時完全不理會這個動作
HandleLidSwitchDocked=ignore         # 已連接外部顯示器=闔蓋時完全不理會這個動作
```
重啟服務
```
sudo systemctl kill -s HUP systemd-logind   # 只重新載入設定
```
驗證是否生效
```
systemd-analyze cat-config systemd/logind.conf | grep -i lid
```
