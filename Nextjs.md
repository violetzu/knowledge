## 建立 Next.js 專案
```sh
npx create-next-app@latest my-app
```
## use docker
```sh
mkdir my-next-app
cd my-next-app
    
docker run --rm -it \
  -v "%cd%":/app \
  -w /app \
  node:20-alpine \
  sh -lc "npx create-next-app@latest ."
```
