# 模块 4.4 · 把 React 版发布到公网（命令清单）

> 这份清单是课件「把它发布到公网」那一段的可执行版，加了两处课件里没写、但照做会踩的坑。
> 服务器路径按课件惯例：`~/zero-to-tech`，用户 `ubuntu`。

与 4.1（vanilla 版）的唯一实质差别：**Nginx 服务的不再是源码目录，而是服务器上 build 出来的 `dist/`**。
所以中间多了一个角色：服务器上要装 **Node**，先 `npm install` 备依赖、再 `npm run build` 产 `dist/`。

---

## 第 0 步：本地确认、提交、推送

```bash
cd ~/zero-to-tech            # 本地项目目录（Windows 用你自己的路径）
git status                   # 应当干净
```

`.gitignore` 里已经有 `dist/` 和 `node_modules/`（本仓库第 3 行），所以推上去的是源码，不含构建产物——这是 4.2 立的规矩，别破坏它。

```bash
git add -A
git commit -m "4.4 完成:数据与界面分离、state、URL"
git push
```

---

## 第 1 步：服务器装 Node（只需第一次）

SSH 登录服务器后：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 24
node -v      # 期望 v24.x
npm -v
```

> **坑 1：nvm 只对当前终端有效。**
> 关掉 SSH 再登进来，`node` / `npm` 会说找不到。每次新开终端先补一句：
> `\. "$HOME/.nvm/nvm.sh"`。
> 想一劳永逸，把它写进 `~/.bashrc` 末尾。

---

## 第 2 步：拉代码、装依赖、构建

```bash
cd ~/zero-to-tech
git pull
npm install          # 备好依赖（有 package-lock.json，用 npm ci 也行，更严格更快）
npm run build        # 源码 → dist/
ls dist              # 应看到 index.html 和 assets/
```

`dist/` 里就是浏览器能直接跑的静态文件，跟 4.1 那堆手写 `.html` 同性质。

---

## 第 3 步：让 Nginx 指向 `dist/`

```bash
sudo vim /etc/nginx/sites-enabled/default
```

只动 `root`，并且**多补一行 `location /`**：

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /home/ubuntu/zero-to-tech/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> **坑 2（本清单的重点）：课件只让你改 `root`，那样 `/text-lab` 刷新会 404。**
> 服务器磁盘上并没有叫 `text-lab` 的文件，Nginx 拿到 `/text-lab` 这个请求会去找 `/dist/text-lab`，找不到就 404。
> `try_files $uri $uri/ /index.html;` 的意思是：先找同名文件，没有就找同名目录，都没有就把请求交给 `index.html`——
> 页面外壳照发，剩下的由前端那个小路由照着地址栏决定渲染哪一页。
> 这也正是这一节「刷新不丢」这句承诺在线上成立的前提。

测试 + 生效：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 第 4 步：验收（三条，缺一条都不算完）

```bash
curl -I http://你的服务器IP/            # 200
curl -I http://你的服务器IP/text-lab    # 200（不是 301/404）
```

浏览器里：

1. 打开 `http://你的服务器IP/` —— 个人主页、卡片飞入、分数滚动，跟本地一样。
2. 点「文字实验室」—— 地址栏变成 `…/text-lab`。
3. 在这一页按 **F5** —— **还停在文字实验室**（这条就是 `try_files` 换来的）。
4. 点后退 —— 回首页。

---

## 如果撞上 403 Forbidden

Nginx 以 `www-data` 身份读文件，而 `/home/ubuntu` 默认权限常常是 750——**它走不进这个家目录**。查一下：

```bash
namei -l /home/ubuntu/zero-to-tech/dist
```

看到 `drwxr-x---` 就说明卡在这儿，两条路选一条：

```bash
# 路 A：给家目录补上"其他人可穿过"的执行位（最小改动）
sudo chmod o+x /home/ubuntu

# 路 B：更正经的做法——把产物挪到 /var/www
sudo mkdir -p /var/www/zero-to-tech
sudo rsync -a --delete ~/zero-to-tech/dist/ /var/www/zero-to-tech/
# 然后 root 改成 /var/www/zero-to-tech
```

---

## 以后每次改完代码

```bash
# 本地
git push

# 服务器
cd ~/zero-to-tech
git pull
npm install      # 只有依赖变了才需要；只改代码可以跳过
npm run build    # 每次都要
```

跟 4.1 比，工程化项目不再是「拉下来就能直接给 Nginx」，中间多了「备依赖 + 构建产物」这两步。
这就是工程化要付的成本，换来的是组件复用、构建优化那些好处。
