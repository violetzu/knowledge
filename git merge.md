# Git 合併分支（Merge）超簡單教學
假設妳現在在 main，然後妳想把 dev 的內容合併進來。
## 1. 先確認目前在哪個分支
```sh
git branch
```
如果不是在 main，切過去：
```sh
git checkout main
```
## 2. 拉最新的 main（避免衝突）
```sh
git pull
```
## 3. 合併 dev 到 main
```sh
git merge dev
```
如果沒有衝突 → 就會直接合併完成。
## ⚠️ 若出現 conflict（衝突）
```sh
<<<<<<< HEAD
(main 目前的內容)
=======
(dev 的內容)
>>>>>>> dev
```
## 4. 衝突解完後提交
```sh
git add .
git commit -m "fix merge conflict"
```
## 5. 最後 push 回 GitHub
```sh
git push
```
## 如果是反過來（要把 main 合併到 dev）
```sh
git checkout dev
git pull
git merge main
git push
```
