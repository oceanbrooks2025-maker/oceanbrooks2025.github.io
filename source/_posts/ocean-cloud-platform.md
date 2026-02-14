---
title: 私有云搭建：全栈开发记录
date: 2026-02-14 15:00
categories: '阶段总结'
tags:
  - 'cloudflare'
  - 'blog'
  - 'note'
  - 'ssh'
description: 关于个人建站的阶段记录
---


**前言**

过去两周，我继续折腾我的 Ubuntu 服务器，把它从一个只能在局域网访问的“孤岛”，改造成了一台拥有公网访问能力、托管了静态博客和动态全栈应用的私有云服务器。

这篇博文旨在完整记录整个技术选型和落地过程。我不打算只写技术路线，而是要把踩过的坑和关键的操作命令记下来，方便日后重装或迁移时查阅。

主要内容分为三部分：

1. **基础设施**：如何搞定内网穿透和远程访问（Cloudflare Tunnel）。
2. **静态网站**：利用 Hexo 和 GitHub 搭建个人博客。
3. **动态应用**：从零开发、Docker 化部署一个前后端分离的私有便签应用。

------

## 第一章 基础设施与远程访问

这是地基部分。核心诉求很简单：我人在外面（比如星巴克），要能访问家里服务器上的网页，或者通过终端控制它。

### 1.1 公网 IP 路线（已放弃）

最开始的想法是直接申请公网 IP。

- **IPv4**：直接给移动运营商打电话，理由是装监控。对方答复现在资源枯竭，家庭宽带是大内网 IP（CGNAT），给不了公网 IPv4。此路不通。
- **IPv6**：现在的光猫和路由器基本都支持 IPv6。我尝试开启了 IPv6，确实能获取到全球唯一的地址。但是测试后发现两个致命问题：
  1. **网络环境受限**：很多办公场所或公共 Wi-Fi 只支持 IPv4，根本连不上家里的 IPv6 地址。
  2. **端口封锁**：运营商对家庭宽带封锁了 80 和 443 端口。这意味着我以后部署网站，必须要在域名后面带个丑陋的端口号（比如 `:8080`），而且没法自动申请 SSL 证书。
  3. **防火墙限制**：家里的路由器无法关闭 IPv6 防火墙，这意味着外部流量进不来。

鉴于上述原因，我决定放弃直连方案，改用 **内网穿透（Tunnel）**。对比了 Frp（需要自己租 VPS 做中转）和 Cloudflare Tunnel，后者免费、不需要额外服务器、提供安全防护、且自带全球 CDN 加速，显然是首选。

### 1.2 域名准备

要用 Cloudflare，首先得有个域名。

我在注册商买了 `oceanlog.top`，费用很低。买好后的第一件事，就是修改域名的 DNS 服务器（Nameservers），把它托管给 Cloudflare。

- **操作**：在域名注册商后台，将 NS 记录修改为 Cloudflare 分配的那两个地址（例如 `xxx.ns.cloudflare.com`）。
- **验证**：去 Cloudflare 控制台添加站点，等到状态变成 "Active" 就算接管成功。

### 1.3 部署隧道

这是整个内网穿透的核心。它的原理是在我家服务器运行一个守护进程 `cloudflared`，主动向 Cloudflare 的边缘节点建立一条加密通道。外部流量通过 Cloudflare 进来，顺着通道流到我家，无需在路由器上做任何端口映射。

**操作步骤：**

1. 进入 **Cloudflare Zero Trust** 控制台，在左侧菜单找到 `Networks` -> `Tunnels`。
2. 点击 `Create a tunnel`，选择 `Cloudflared` 类型。
3. 给隧道起个名（比如 `HomeServer`），保存。
4. **安装守护进程**：在服务器（Ubuntu）上执行它给出的安装命令。

当时我使用的是 Debian/Ubuntu 的安装方式，命令大致如下（注意 `token` 是生成的长字符串）：

```bash
# 下载、安装、启动 cloudflared（注意替换 token）
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
sudo cloudflared service install <your-token> 
```

安装完成后，检查服务状态：

```bash
sudo systemctl status cloudflared
```

如果看到 `Active: active (running)`，且 Cloudflare 网页上的状态变成了 `Healthy`，说明隧道这就打通了。

### 1.4 SSH Web访问

隧道通了之后，第一件事是解决远程管理问题。我暂时只需要一个能随时随地用的 Web 终端，Cloudflare Zero Trust 自带的 Browser Rendering 功能完美解决，连 SSH 客户端都不用装。

**配置流程：**

1.  **设置公网映射 (Public Hostname) ** ：

   在 Tunnel 的配置页面，添加一个 Public Hostname。

   - **Subdomain**: `ssh` (即 `ssh.oceanlog.top`)
   - **Service**: `ssh://localhost:22`
   - 这一步告诉 Cloudflare，把访问 `ssh.oceanlog.top` 的流量，转发给我服务器的 22 端口。

2. ** 配置应用策略 (Access Application) **：

   光映射不行，还得有界面。

   - 在 Zero Trust 左侧菜单 `Access` -> `Applications` -> `Add an application`。
   - 选择 `Self-hosted`。
   - **Application name**: 填个名字比如 "SSH Web"。
   - **Session Duration**: 设置登录一次管多久，比如 "24 hours"。
   - **Domain**: 选择刚才配置的 `ssh.oceanlog.top`。
   - **Settings 页面**: 关键点在最下面的 `Browser rendering`，选择 `SSH`。

**效果**：

保存后，我在任何浏览器的地址栏输入 `ssh.oceanlog.top`，就会弹出一个 Cloudflare 的身份验证页（验证邮箱验证码），验证通过后直接在浏览器里渲染出一个终端窗口，输入服务器账号密码即可登录。

至此，无论我有无公网 IP，只要有网，我就能完全控制家里的服务器。基础设施搭建完毕。



------



## 第二章 静态博客

基础设施搞定后，先搭建一个简单的静态网站，用以存放个人博客。我不希望在这个环节投入任何服务器运维成本（比如去维护一个 Linux 上的 Nginx 或者 PHP 环境），所以我选择了 **Jamstack** 架构：**Hexo + GitHub + Cloudflare Pages**。

这套方案不仅完全免费，而且利用了 Cloudflare 的全球 CDN，速度快，自带 SSL 证书，是目前最省心的静态站方案。

### 2.1 架构思路与选型

传统的博客是动态的，用户访问时服务器现算页面。而 Hexo 是静态生成器，它把 Markdown 文章在本地编译成 HTML。

我的部署流程如下：

1. **本地**：写 Markdown，运行 Hexo 本地预览。
2. **GitHub**：只托管“源代码”（即生成 HTML 之前的原始文件）。
3. **Cloudflare Pages**：监听 GitHub 仓库。一旦有新代码提交，自动拉取 -> 构建 -> 发布。

### 2.2 本地环境初始化

前提是电脑上得装好 Node.js 和 Git。

安装 Hexo CLI 并初始化博客目录非常简单：

```bash
npm install -g hexo-cli
hexo init ocean-blog
cd ocean-blog
npm install
```

初始化完成后，目录结构里最重要的几个部分是：

- `source/`：存放我的 Markdown 文章和图片。
- `themes/`：存放博客主题。
- `_config.yml`：全局配置文件。

此时运行 `hexo server`，就可以在 `localhost:4000` 看到雏形了。

### 2.3 GitHub 仓库策略

这里有一个容易混淆的点：**我们要传到 GitHub 上的，不是 `public` 文件夹，而是除了 `public` 和 `node_modules` 之外的所有源文件。**

很多教程的路线是 `hexo-deployer-git` 把生成的 HTML 推送到 GitHub Pages，那种方式不仅由于 GitHub 网络问题导致访问慢，而且你就丢失了源代码的版本控制。

我的做法是：GitHub 存放源文件，CloudFlare 负责构建

1. 在 GitHub 上新建一个**私有仓库**（Private Repo），命名为 `ocean-blog-source`。私有是为了保护草稿和配置隐私。
2. 在本地博客根目录初始化 Git：

```bash
git init
git branch -M main
git remote add origin git@github.com:yourname/ocean-blog-source.git
```

1. **配置 `.gitignore`**：这是最重要的一步。确保文件里包含以下内容，避免把垃圾文件传上去：

```text
node_modules/
public/
db.json
*.log
```

1. 提交源代码：

```bash
git add .
git commit -m "feat: blog init"
git push -u origin main
```

### 2.4 Cloudflare 自动化部署

代码上了 GitHub，接下来就是让 Cloudflare 接管构建。

1. 登录 **Cloudflare Dashboard**，进入 `Workers & Pages` -> `Create application` -> `Pages` -> `Connect to Git`。

2. 授权访问我的 GitHub 账号，并选择刚才创建的 `ocean-blog-source` 仓库。

3. **配置构建环境（Build settings）**：

   Cloudflare 非常智能，预设了 Hexo 模板，通常只需要确认以下信息：

   - **Framework preset**: 选 `Hexo`。
   - **Build command**: `hexo generate`（或者 `npm run build`）。
   - **Build output directory**: `public`。

4. 点击 `Save and Deploy`。

Cloudflare 会立刻拉取我的代码，下载 Node.js 依赖，执行 `hexo generate`，然后把生成的 `public` 目录推送到它的 CDN 上。

整个过程大概耗时 1-2 分钟。以后我写完文章，只需要在本地 VS Code 里执行：

Bash

```
git add .
git commit -m "new post: hello world"
git push
```

Cloudflare 就会自动触发构建，我的博客就更新了。

### 2.5 域名绑定

最后，给博客一个体面的入口。

在 Cloudflare Pages 的项目设置里，点击 `Custom domains`，输入 `blog.oceanlog.top`（后改变主意，直接用主域名 `oceanlog.top`）。

由于我的域名 DNS 本来就是 Cloudflare 托管的，它会自动添加一条 CNAME 记录，并自动申请和部署 SSL 证书。稍等片刻，HTTPS 的锁头就出现了。

至此，一个高可用、零成本、自动化的静态博客系统搭建完成。



------



## 第三章 动态应用

静态博客是对外展示，动态应用是对内服务。这一部分涵盖了全栈开发、容器化部署、网络隔离的完整过程，是技术含量最高、逻辑最复杂的部分。

这一章的目标是：从零开发一个**前后端分离**的私有便签应用（Ocean Notes），并将其容器化部署到服务器上。它必须满足三个核心需求：

1. **数据私有**：内容存在我家服务器的硬盘上，而非第三方云端。
2. **访问控制**：通过长 Token 进行“门神”式验证。
3. **架构安全**：后端服务隐藏在防火墙和 Nginx 之后，绝不直接暴露给公网。

### 3.1 技术栈选型

作为一个轻量级的个人工具，我没有选择厚重的 Java/Spring 或复杂的 K8s，而是选择了**“小而美”的 Python 云原生栈**：

- **前端**：Vue 3 (Composition API) + Tailwind CSS (CDN 引入，保持极简)。
- **后端**：FastAPI (Python 高性能异步框架) + SQLite (单文件数据库)。
- **网关**：Nginx (反向代理 + 静态资源托管)。
- **部署**：Docker + Docker Compose (容器编排)。

### 3.2 后端开发：FastAPI

后端的职责很简单：验证 Token、读写数据库。

#### 核心代码实现

为了保持项目整洁，我采用了**单文件架构** (`main.py`)。核心逻辑包括：

1. **数据模型 (Pydantic)**：定义了前端传来的数据格式（`content`）和返回的数据格式（`id`, `timestamp`）。

2. **中间件鉴权**：编写了一个依赖注入函数 `verify_token`。它会拦截每一个请求，检查 HTTP Header 中的 `X-Ocean-Token`。

3. **环境变量隔离**：这是安全开发的第一课。我没有将密码硬编码在代码里，而是通过 `os.getenv("SECRET_TOKEN")` 从系统环境变量中读取。

   需要注意的是，要在 .gitignore 添加忽略 .env 文件

**核心代码片段：**

```python
# main.py (部分核心逻辑)
app = FastAPI()

# 从环境变量读取密码，若无则使用默认值（开发环境）
EXPECTED_TOKEN = os.getenv("SECRET_TOKEN", "ocean_dev_token")

# 依赖注入：门神验证
async def verify_token(x_token: str = Header(..., alias="X-Ocean-Token")):
    if x_token != EXPECTED_TOKEN:
        raise HTTPException(status_code=401, detail="Invalid Token")
    return x_token

# 数据库路径：适配 Docker 挂载
DB_PATH = "storage/data.db"
```

### 3.3 前端开发：Vue 3

前端是一个单页应用 (SPA)，包含两个视图状态：

1. **验证视图**：一个简单的密码框。
2. **工作视图**：便签列表和输入框。

#### 遇到的坑：异常吞没

在调试时，我发现即便输入了错误的 Token，前端有时也会错误地进入“登录状态”。

**原因分析**：

我在 `fetch` 请求外层包裹了太宽泛的 `try...catch`，导致后端返回的 `401 Unauthorized` 错误被前端“吃掉”了，逻辑继续向下执行，错误地将无效 Token 存入了 `localStorage`。

**修复方案**：

重构了 `request` 通用函数，显式拦截 `401` 状态码，并抛出特定错误供上层 UI 处理。

```javascript
const request = async (endpoint, options = {}) => {
    // ... 组装 Headers ...
    const res = await fetch(`${API_BASE}${endpoint}`, { ...options, headers });
    
    // 严查 401，遇到直接抛错或踢出
    if (res.status === 401) {
        throw new Error("Token 错误: 拒绝访问");
    }
    return await res.json();
};
```

### 3.4 容器化：Dockerfile

应用开发完成后，下一步是“装箱”。我为后端编写了 `Dockerfile`。

**一点优化**：在 Dockerfile 中指定清华源镜像，提高构建速度。

```dockerfile
# 使用轻量级 Python 基础镜像
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .

# 使用清华源加速依赖安装
RUN pip install --no-cache-dir -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

COPY . .

# 启动命令
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 3.5  Docker 容器构建 [重点]

这是全篇最复杂、也是踩坑最多的部分。我们需要把 Nginx、Python 后端和数据存储串联起来。

#### 1. 初始架构与 "502 Bad Gateway"

最开始，我试图让 Nginx 监听 `127.0.0.1:8888`，目的是只允许本机访问。

结果：Cloudflare Tunnel 报错 502。

**原因排查**：

- **网络隔离**：Docker 容器（Tunnel）运行在 `172.17.x.x` 网段，而 Nginx 监听的是宿主机的回环地址 `127.0.0.1`。
- **不可达**：容器内的进程无法直接访问宿主机的 `localhost`。

**解决方案**：

将 Nginx 的端口绑定修改为 **Docker 网桥 IP** (`172.17.0.1`)。这样既利用了 Docker 网络的互通性，又避免了向局域网暴露端口。

#### 2. 后端隐身

为了更高的安全性，我对 Python 后端容器做了“隐身处理”。

**操作**：在 `docker-compose.yml` 中，彻底删除后端的 `ports` 映射。

```yaml
  simple-note-backend:
    # ports: 
    #   - "8000:8000"
```

**意义**：这意味着 Python 后端完全断开了与宿主机外部的直接联系。它只能被处于同一个 Docker 网络下的 Nginx 访问。外部访客即使扫到了我的 IP，也无法绕过 Nginx 直接攻击 API。

#### 3. 数据持久化

初次部署时，我重启容器后发现数据丢了。

**原因**：Docker 卷挂载路径错误。我原本写的是 `- ./data/data.db:/app/data.db`（文件对文件）。

**机制**：当宿主机文件不存在时，Docker 会自动创建一个名为 `data.db` 的文件夹，导致 SQLite 无法写入。

**修复**：改为目录对目录挂载 `- ./data:/app/storage`，让数据库文件在目录中自动生成。

### 3.6 最终配置

这是经过多次迭代后，稳定运行的 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  # 服务1：Python 后端 (核心业务)
  simple-note-backend:
    build: ./simple-note/backend
    restart: always
    # 安全：不暴露任何端口到宿主机
    volumes:
      - ./data:/app/storage  # 持久化存储
    environment:
      - SECRET_TOKEN=${SECRET_TOKEN} # 从 .env 读取密码

  # 服务2：Nginx 网关 (流量入口)
  nginx-gateway:
    image: nginx:alpine
    restart: always
    ports:
      # 绑定到 Docker 网桥 IP，仅允许 Tunnel 访问，局域网不可见
      - "172.17.0.1:8888:80"
    volumes:
      - ./nginx/dev.conf:/etc/nginx/conf.d/default.conf
      - ./simple-note/frontend:/usr/share/nginx/html/simple-note
    depends_on:
      - simple-note-backend
```

### 3.7 总结

至此，我的私有云便签应用 "Ocean Notes" 已经稳稳地运行在旧服务器上了。

回顾这部分工作，最大的收获不在于代码本身，而在于对 **网络接口绑定 (Interface Binding)**、**Docker 网络隔离** 以及 **数据持久化** 的深刻理解。这些在本地开发环境（Localhost）很难遇到的问题，在服务器部署实战中给我上了宝贵的一课。



------



## 第四章 总结与未来路线图

### 4.1 技术路线回顾

经过两周的折腾，这台服务器现在已经变成了一个标准化的私有云平台。回顾整个过程，我并没有捣鼓什么高深的技术，而是将业界成熟的方案进行了组合：

1. **网络层**：
   - 放弃了不稳定的公网 IP 和端口映射，选择了 **Cloudflare Tunnel**。
   - 实现了 **Zero Trust** 访问控制，安全地通过浏览器访问 SSH 和 Web 服务。
   - 解决了家庭宽带无 80/443 端口的问题，实现了标准的 HTTPS 访问。
2. **静态服务**：
   - 采用了 **Jamstack** 架构（Hexo + GitHub + Cloudflare Pages）。
   - 实现了 **GitOps** 工作流：本地只管写 Markdown，推送到 GitHub 后，构建和发布全自动完成。
   - 成本为零，且拥有企业级的 CDN 加速。
3. **动态服务：
   - 采用了微服务容器化架构。
   - **Nginx** 作为网关，负责路由分发和静态资源。
   - **FastAPI** 作为后端，负责业务逻辑，且通过环境变量隔离了敏感配置。
   - **网络隔离**：后端容器彻底断网，只通过 Docker 内部网络与网关通信，最大程度减少了攻击面。

这套架构最大的优势在于**可复制性**和**扩展性**。有了这个地基，以后部署任何新服务（比如私有网盘、媒体中心），只需要复制一份 `docker-compose.yml`，改改端口和配置即可。

### 4.2 升级路线：Notes v2.0

目前的便签应用只是一个 MVP（最小可行性产品），功能还很简陋。接下来的开发计划包括：

- **Markdown 渲染支持**：目前输入框只支持纯文本。既然是程序员的笔记，支持 Markdown 语法（代码高亮、列表、粗体）是刚需。前端需要引入 `marked.js` 或类似库。
- **全文搜索**：随着笔记增多，光靠瀑布流展示是不够的。需要在后端实现一个简单的关键词过滤接口，甚至引入简单的搜索引擎机制。
- **用户体验优化**：
  - **夜间模式**：适配 Tailwind CSS 的 Dark Mode。
  - **移动端适配**：目前虽然能用，但在手机上的交互体验（特别是输入框高度）还有优化空间。

### 4.3 扩展路线：引入更多云应用

既然 Docker 环境已经搭好，服务器性能还有富余，可以考虑部署更多实用的自托管服务：

- **个人导航页（Dashboard）**：现在我有 `blog.oceanlog.top` 和 `dev.oceanlog.top`，以后服务多了记不住域名。可以部署一个 **Homer** 或 **Dashy**，作为浏览器的起始页，统一管理入口。
- **服务监控（Monitoring）**：如何知道博客挂了没？部署一个 **Uptime Kuma**。它可以监控我的所有服务状态，一旦挂了通过 Telegram 或邮件发报警，而且它还能生成好看的状态页。
- **文件同步（Private Cloud）**：目前的便签只能存文字。如果需要存文件、照片，可以部署轻量级的 **File Browser** 或者功能强大的 **Nextcloud**。

### 4.4 运维路线：数据备份

这是目前架构中唯一的短板。虽然我用了 Docker Volume 持久化了数据，但数据依然存储在服务器的物理硬盘上。**“数据不做异地备份，就等于没有备份”。**

下一步的计划是引入 **Rclone** 工具：

- 编写 Shell 脚本，每天凌晨定时打包压缩 `data/` 目录。
- 通过 Rclone 自动加密上传到对象存储（如 Cloudflare R2 或 AWS S3）。
- 利用 Cloudflare R2 的免费额度（10GB），实现零成本的异地容灾备份。

------

**结语**

折腾服务器的过程，本质上是一个**“发现问题 -> 寻找方案 -> 踩坑填坑 -> 最终落地”**的闭环。

从最开始连不上 IPv6 的沮丧，到后来看着 Docker 日志跑起来的兴奋，再到最后能在手机上随时随地记下一条笔记的满足感。这大概就是技术折腾的乐趣所在吧。