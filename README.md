# EduBoard 智教白板

面向基础教育的 AI 白板教学工具。教师粘贴或上传一道试题，AI 老师给出**分步解析**，并在白板上生成**可调参数、可控进程**的交互图形与实验动画。

- 需求 1 教师输入台：文本 + 图片（粘贴/拖拽/附件），发送 / 停止 / 清空输入 / 新对话
- 需求 2 解析面板：流式分步解析（KaTeX 公式），可一键折叠
- 需求 3 可视化白板：`<iframe srcdoc>` 承载自包含交互 HTML，滑块/播放控制进程
- 需求 4 关键词自动触发：输入实时检测学科/题型 → 路由 + 模型自主发 artifact 块 → 技能生成
- 需求 5 纯网页访问

## 架构

```
web (Vue3+Vite)  ──/api──►  server (Node/Express + AI SDK)  ──HTTP──►  skills-py (FastAPI + sympy)
   三区布局/流式              streamText + block-parser              kernel 精确计算 → 数据驱动模板
   KaTeX/iframe               router 关键词 + SQLite 持久化           → 自包含交互 HTML
```

灵感来源：ChatTutor（对话+白板动作驱动壳）与 edulab（sympy→数据驱动交互 HTML）。
**本项目完全自包含**——edulab 三技能已 vendored 到 `skills-py/edulab/`，不引用任何外部参考目录。

## 目录

```
eduboard/
├─ shared/     # ClientAction 契约 + 同构 resolver + 学科触发词表（前后端共用）
├─ web/        # Vue3 前端：PromptArea / AnalysisPanel / Board / SettingsDialog
├─ server/     # Express：gateway 多provider · block-parser · router · chat-service · db(SQLite)
└─ skills-py/  # FastAPI sidecar：edulab(vendored) 三技能 + physics 新增（抛体/简谐）
```

## 快速开始

### 方式一：一键启动（推荐）

双击项目根目录的 **`start.bat`**，脚本自动：
1. 检测 Node.js / Python 环境
2. 检查 `.env` 配置（不存在则从 `.env.example` 创建并打开编辑）
3. 自动创建 Python 虚拟环境并安装依赖（`skills-py/venv/`）
4. 安装 Node.js 依赖（`npm install --workspaces --no-audit --no-fund`）
5. 依次启动 sidecar → server → web，每个服务等待就绪后再启下一个
6. 自动打开浏览器 `http://127.0.0.1:8789`

停止所有服务：双击 **`stop.bat`**。

### 方式二：手动启动

### 前置准备
- Node.js >= 22（生产环境推荐 Node.js 24 LTS）· Python 3.11+（需已装 sympy、fastapi、uvicorn、pydantic）
- 一个大模型 API Key（OpenAI / DeepSeek / Anthropic），或兼容的 base_url

若缺少 Python 依赖：`pip install sympy fastapi uvicorn pydantic`

### 1) 配置
```bash
cp .env.example .env               # 然后编辑 .env 填写你的 API Key 等信息
```
至少填写以下字段：
- `MODEL_PROVIDER`=你的提供商（`openai` / `deepseek` / `anthropic`）
- `MODEL_API_KEY`=sk-...（必需）
- `MODEL_BASE_URL`=API 端点（OpenAI 兼容接口）
- `AGENT_MODEL`=模型名

> **如果 `.env` 中配置了 `MODEL_API_KEY`，所有用户都能用服务端 Key；否则必须在前端“设置”里填 BYOK（Key 仅存当前标签页会话的 sessionStorage，关闭标签页后清除）。**

sidecar 鉴权（必读）：
- `SIDECAR_TOKEN`=随机值 —— sidecar 与 server 的共享鉴权 token（两端必须一致）。
  留空时 sidecar **仅允许 127.0.0.1 回环访问**；若设置 `SIDECAR_HOST=0.0.0.0`
  且 token 为空，sidecar 会拒绝启动（fail-closed 保护，防止局域网未授权调用）。
  生成随机值：`python -c "import secrets; print(secrets.token_hex(24))"`
- `SIDECAR_URL` 默认 `http://127.0.0.1:8791`，如果 sidecar 换了端口记得同步改。

并发调优（可选）：
- `MAX_CONCURRENT_CHATS=6` —— 同时进行的大模型对话上限（2 核机器 4-8）
- `RATE_LIMIT_PER_MINUTE=15` —— 单 IP 每分钟对话/建会话上限（防单人多会话拖垮全局）
- `MANIM_MAX_RENDERERS=2` —— manim 并发渲染数（超低配机器可改 1）

### 2) 安装依赖（根目录一次安装所有 workspace）
```bash
npm install --workspaces   # 同时装 shared + server + web
```

> **关于 node_modules 的备份与重装**
> - `node_modules/`（约 400MB）属于**可再生成依赖**，备份项目时可以排除；
> - 若开发目录按此方式清理后（或换机器/重新克隆），运行前需重新执行上面的
>   `npm install --workspaces`——node_modules 缺失或为空目录时，`npm run dev` /
>   `npm run build` 等命令会直接报错（`Cannot find module`），装完即恢复；
> - **安装版（EduBoard-V1.0.0-Setup.exe）用户无需手动操作**：安装包不含
>   node_modules，安装程序会在安装过程中自动执行 `npm install`（npmmirror
>   加速，失败回退官方源），目标机只需预装 Node.js 22+ 并联网。

### 3) 启动 Python 技能 sidecar
```bash
cd skills-py
python server.py            # 默认监听 http://127.0.0.1:8791
# 也可指定端口： SIDECAR_PORT=18791 python server.py
# 仅容器/受控网络需要：SIDECAR_HOST=0.0.0.0 python server.py
```
验证：`curl http://127.0.0.1:8791/health` → `{"ok":true}`

### 4) 启动后端
```bash
cd server
npx tsx src/index.ts        # 默认监听 SERVER_PORT（.env 中配置，默认 17790）
# 若 17790 被占，编辑 .env 改成其它端口即可
```
验证：`curl http://127.0.0.1:17790/health` → `{"ok":true}`

### 5) 启动前端
```bash
cd web
npx vite --port 8789        # 或 npm run dev
```
访问 `http://127.0.0.1:8789`。

前端 /api 请求自动由 Vite 代理到后端（代理目标通过 `VITE_API_BASE_URL` 或默认 `http://127.0.0.1:17790`）。

### 6) 一键启动（开发用）
根目录安装 `concurrently` 后：
```bash
npm run dev                  # 并行启动 sidecar + server + web
```
（此命令假定 sidecar 端口 8791、server 端口 17790、web 端口 8789）

### 端口速查

| 服务 | 端口 | 配置方式 |
|---|---|---|
| sidecar（Python） | `SIDECAR_PORT` 或默认 8791 | `.env` 中 `SIDECAR_HOST` / `SIDECAR_PORT` |
| server（Express） | `SERVER_PORT` 或默认 17790 | `.env` 中 `SERVER_PORT` |
| web（Vite） | 8789 | `--port` 参数 |

安全说明：开发默认（.env.example）服务端只监听 `127.0.0.1`。如确需局域网访问，
可显式设置 `SERVER_HOST=0.0.0.0`，但项目本身是单用户会话模型，公网或多人部署前
必须在反向代理层增加身份认证与 HTTPS。已内置的纵深防御：
- sidecar 与 server 之间 SIDECAR_TOKEN 鉴权（hmac 常量时间比较，/health 豁免）；
- 后端 /api 的 Origin/CORS 守卫、SSRF 防护（云元数据/DNS 重绑定域名/链路本地）；
- 对话与建会话按来源 IP 限速（`RATE_LIMIT_PER_MINUTE`），会话 30 天自动清理；
- API Key 仅存前端 sessionStorage，不落盘。
Docker Compose 默认将 Web 入口发布到所有网卡的 `8789` 端口，后端和 sidecar 不直接暴露给局域网。

### 局域网 HTTP 部署

使用 Docker Compose 时执行：

```bash
docker compose up -d --build
```

同一局域网设备访问 `http://<部署机器局域网IP>:8789`。如需更换端口，在
`.env` 中设置 `WEB_PORT=8080`，然后访问 `http://<局域网IP>:8080`。

系统已识别 `X-Forwarded-Host` 和 `X-Forwarded-Proto`。后期增加 HTTPS 时，
可在当前 Web 服务前放置 Caddy、Nginx 或其他反向代理终止 TLS，无需修改前端 API 地址。

Windows 安装包用户可查看 `pack/使用与部署说明.txt`。其中提供了通过
`sites-enabled`、`conf.d` 或面板独立站点新增代理配置的示例，不要求替换用户现有的
`nginx.conf`。安装包为联网引导安装（npm/PyPI 依赖安装时下载）；MiKTeX、ffmpeg、
pandoc 三项可选系统依赖不打包，安装完成后安装程序会自动检测缺失项并引导下载。
sidecar 服务注册时从 `.env` 读取 `SIDECAR_HOST` / `SIDECAR_TOKEN`（修改后需重跑
安装程序或 `install-service.bat` 重注册）。

### 可能遇到的问题

**server 报错 `EADDRINUSE`（端口被占用）**：编辑 `.env` 换 `SERVER_PORT` 即可——server 会从项目根 `.env` 读取配置，无需在 cwd 有 `.env`。

**sidecar 200 但 server 连不上 sidecar**：确认 `.env` 中 `SIDECAR_URL` 与 sidecar 实际监听端口一致。

**`生成交互图失败`**：确认 sidecar 在运行且能 `curl /health` 通过；确认 `.env` 的 `SIDECAR_URL` 正确。

## 模型配置

默认走 OpenAI 兼容接口。可在网页右上角「设置」里选择 OpenAI / DeepSeek / Anthropic 并填 baseURL + Key + 模型名；非敏感设置保存在 localStorage，Key 仅保存在当前标签页会话的 sessionStorage。也可在 `.env` 里配服务端默认值。

## 验证

- 技能自检：`python skills-py/physics/lib/physics_kernel.py`、`python skills-py/edulab/edu-analytic-geometry/lib/analytic_kernel.py`
- 直生成：`python skills-py/edulab/edu-analytic-geometry/scripts/generate.py ellipse_dot_range ./demo.html`
- 端到端：粘贴一道解析几何题 → 解析面板出解 + 白板出现交互图，拖滑块看数量积随倾角变化

## Manim 数学精讲动画

项目已内置 `math-function` Manim 技能，首批支持：

- `function_graph`：按原题表达式生成函数图像动画；
- `derivative_tangent`：由 SymPy 精确计算导数、切点、斜率和切线，并演示割线趋近；
- `integral_area`：由 SymPy 精确计算定积分并展示积分区间面积。

动画输入仍使用 artifact 结构化 spec，模型只抽取函数表达式和题目参数，不能提交任意 Python。MP4 按 spec 哈希缓存在 `MANIM_MEDIA_DIR`（默认 `./data/manim`），会话数据库仅保存引用它的 HTML，不内嵌视频。Manim 默认单任务渲染，可通过 `MANIM_MAX_RENDERERS` 调整并发。

本地运行除 Python 包外还需要系统可执行文件 `ffmpeg`（视频合成）与
`MiKTeX`（pdflatex，数学公式渲染）。程序会自动检测 MiKTeX 常见安装路径并注入
PATH（安装后重启服务即可）。推荐使用 `docker compose up --build`，容器镜像会自动
安装 FFmpeg、Cairo、Pango 与 TeX。示例 spec：

```json
{"body":"derivative_tangent","params":{"expression":"x^2","x0":2,"x_min":-3,"x_max":5,"y_min":-3,"y_max":10}}
```

## 许可

Copyright (c) 2026 YINJUN. 仅供个人学习使用，商用请联系权利人（188878477@qq.com）。

- `skills-py/edulab/` 为 edulab（Apache-2.0）vendored 副本，随附其 LICENSE / NOTICE。
