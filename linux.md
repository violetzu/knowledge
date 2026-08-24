# 目錄
### [使用者&權限](#使用者&權限)
### [修改主機名稱](#修改主機名稱)
### [ssh 改 port](#ssh-改-port)


# 安裝ssh
在 Ubuntu 上安裝 SSH
```
sudo apt update
sudo apt install openssh-server -y
```
啟動並確認狀態
```sh
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
```

開放防火牆（若有啟用 ufw）
```sh
sudo ufw allow ssh
```

# 使用者&權限

顯示所有使用者
```sh
awk -F: '$3>=1000 && $1!="nobody" {print $1, $3}' /etc/passwd
```
刪除使用者
```sh
sudo userdel -r <username>
```

# 修改主機名稱
1) 用 hostnamectl 修改
```sh
sudo hostnamectl set-hostname 新主機名稱
```
2) 確認修改結果
```
hostnamectl
```
3) 手動檢查/更新 /etc/hosts
```sh
sudo nano /etc/hosts
```
找到類似 `127.0.1.1   舊主機名稱` <br>改為 `127.0.1.1   新主機名稱` 


# ssh 改 port

1) 確認是否有 socket activation 機制
```sh
systemctl status ssh.socket
```
> 若顯示 could not be found,代表是傳統模式,直接跳到第 3 步

2) 停用 ssh.socket(若存在且啟用中)
```sh
#停止並停用 ssh.socket
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket

# 確保 ssh.service 是 enabled(可以開機自動啟動)
sudo systemctl enable ssh.service
```

3) 編輯 sshd_config
```bash
sudo nano /etc/ssh/sshd_config
```
輸入指令後 Enter 完 SSH 配置文件就會打開。找到讀取 #Port 22 這一行。接下來，刪除 # 並將 22 替換為您要使用的新 SSH port 號(如下方圖示)。

<img width="1298" height="553" alt="image" src="https://github.com/user-attachments/assets/cb9a95f5-9477-4d54-9bf7-96bafeacc276" />

將 SSH port 更改為 30678：

<img width="1320" height="553" alt="image" src="https://github.com/user-attachments/assets/9d32e6da-e140-4551-9910-3a358427a609" />


5) 測試設定檔語法
```sh
sudo sshd -t
```

7) 重啟 SSH 服務
```sh
sudo systemctl restart ssh.service
```

8)  確認
```
sudo ss -tulnp | grep 30678
```
確認有出現監聽紀錄,且執行的 process 是 sshd(而不是 systemd)

**務必開新視窗測試連線再關舊視窗**
