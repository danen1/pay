# 微信/QQ场景内域名遮罩跳转服务

- 微信/QQ内置浏览器打开：保持当前域名（例如 `baidu.com`），但实际内容来自 `TARGET_URL`；
- 外部浏览器（如Safari/Chrome）打开：直接跳转到 `TARGET_URL`（保留路径与查询参数）。

## 方案 A：纯静态 HTML（仅有静态托管即可）

- 将 `index.html` 上传到你的静态空间（例如只支持 HTML 的服务器）。
- 打开 `index.html`，将其中的 `TARGET_URL` 设置为你的目标域，例如：
  ```html
  <script>
    const TARGET_URL = 'https://your-target.com';
  </script>
  ```
- 行为说明：
  - 微信/QQ 内置浏览器：在你的域名下，通过 `iframe` 展示目标域内容（若目标域禁止嵌入，则显示提示与“在浏览器中打开”按钮）。
  - 外部浏览器（Safari/Chrome 等）：自动跳转到目标域，保留路径与查询参数。
- 局限：
  - 若目标站点设置了 `X-Frame-Options: DENY/SAMEORIGIN` 或严格 CSP 将阻止 `iframe`，只能引导用户“在浏览器中打开”。
  - 纯前端无法进行真正的跨域代理与 Cookie/头部重写，仅能展示或跳转。

## 方案 B：Node 反向代理（需要可运行 Node 的服务器）

1. 安装依赖：
   - 服务器安装 Node.js 16+。
   - 在项目目录执行：
     ```bash
     npm install
     ```
2. 配置环境：
   - 复制 `.env.example` 为 `.env`，设置：
     - `TARGET_URL`：需要镜像/跳转的目标域，例如 `https://your-target.com`
     - `PORT`：服务端口（默认 3000）
3. 启动服务：
   ```bash
   npm run start
   ```
   访问 `http://localhost:3000/_health` 检查服务状态。

## 部署到你的域名（Node方案）

常见做法是在你的域名（如 `baidu.com`）的 Nginx 上反向代理到此 Node 服务。

示例 Nginx 配置（需按你的环境微调）：

```nginx
map $http_user_agent $is_webview {
    default                0;
    ~*MicroMessenger        1; # 微信
    ~*\bQQ\/               1; # QQ App WebView
    ~*QQBrowser            1; # QQ 浏览器内核标识
    ~*MQQBrowser           1;
}

server {
    listen 80;
    server_name baidu.com; # 替换为你的域名

    location /_health {
        proxy_pass http://127.0.0.1:3000/_health;
    }

    # 非微信/QQ：跳转到目标域（保留路径与查询）
    location / {
        if ($is_webview) {
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:3000;
            break;
        }
        return 302 https://your-target.com$request_uri; # 替换为你的目标域
    }
}
```

> 提醒：如果你的 Node 服务直接对外（不经 Nginx），请确保 HTTPS 与反向代理头已正确配置，以让 UA 识别与协议判断准确。

## 工作原理与注意点

- UA 识别：通过 `User-Agent` 判断微信 (`MicroMessenger`) 或 QQ (`QQ/`, `QQBrowser`, `MQQBrowser`) WebView。
- 微信/QQ内：使用反向代理（`http-proxy-middleware`）拉取 `TARGET_URL` 的内容，并在传输层重写 Cookie 域（确保功能可用）。对文本类响应（HTML/CSS/JS）做基础的绝对链接替换，将 `https://TARGET_HOST` 改写为当前域，以尽量保持“域名遮罩”。
- 外部浏览器：对 GET/HEAD 用 `302`，其它方法用 `307`，跳转到 `TARGET_URL` 并保留路径与查询参数。
- 兼容性：如果目标站点设置了严格的 CSP、绝对跳转或依赖复杂的前端路由/跨域策略，可能需额外调整（例如更完整的内容重写、Nginx `sub_filter`、或特定 Header 定制）。

## 法规与合规

请确保你拥有目标内容的合法授权，并遵守平台（微信/QQ）及目标网站的服务条款。某些站点禁止镜像/遮罩或内容重写，部署前请审慎评估。