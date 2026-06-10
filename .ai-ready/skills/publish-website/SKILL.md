---
name: publish-website
description: 将当前会话中生成的 Web 项目发布为线上托管应用。负责环境信息获取、项目类型探测、前端构建（复用 deploy-website 的探测规则）、基于应用内容自动生成元数据、静态资源打包，并通过 multipart 表单 POST 到 showcase API 完成上传。支持纯静态、Node.js 前端、以及容器化后端项目。
arguments:
  - name: workspace
    description: 待发布项目的绝对路径，默认使用当前工作目录
    required: false
---

# 发布应用（Publish Website）

将纯前端项目或带后端的容器化项目发布为线上应用。本 Skill 以严格的流水线方式执行——**不得跳过步骤**，**不得在未完成时声称成功**。

## 流水线总览

1. 获取环境信息（client_id）
2. 询问是否包含后端服务（分流到 static / backend 子流水线）
3. 探测项目类型（复用 `deploy-website` 探测规则；后端覆盖 Node-Express、FastAPI、Django+gunicorn、Spring Boot jar、Go、Rust 等）
4. 准备产物：
   - static 分支：必要时构建前端，准备静态产物
   - backend 分支（步骤 3b）：生成 Dockerfile、build、run、healthcheck、save 镜像
5. 基于应用内容自动生成应用名称与描述，再向用户逐项确认；询问应用作者
6. 打包为 `/tmp/dist.zip`（static 分支）或确认 `/tmp/showcase-image.tar.gz`（backend 分支）
7. 确定 `ticket`（首次提交需询问用户是否复用既有应用）
8. 通过 multipart 表单 POST 到 showcase API，缓存返回的 `ticket`，向用户返回 `site_url`

服务端注册与管理员审核发生在 Skill 执行结束**之后**，不属于本 Skill 的职责范围。

### 关于 `ticket`（应用更新密钥）

- `ticket` 是 showcase 服务用来识别"同一个应用"的凭证：首次创建会下发一个 `ticket`，后续若想**更新**该应用而不是新建，需在请求体中带上同一个 `ticket`
- **会话级缓存**：当前会话中**首次**成功创建应用后拿到的 `ticket`，必须缓存于会话上下文（例如记在内存/笔记中），同一会话内后续每次提交都自动使用该 `ticket`，**不得**再向用户询问
- 仅当**本会话从未提交过应用**时，才需要询问用户是不是要复用其他任务中创建的应用（步骤 7a）
- **跨 kind 切换**：同一 `ticket` 可以从 `static` 切换为 `backend`（或反之），服务端会将原应用整体替换为新 kind 并重新进入待审核状态；在用户确认时必须明确告知「将把原应用从 X 切换为 Y，并重新进入待审核状态」

---

## 步骤 1 —— 获取环境信息

执行 `hostname` 命令获取当前主机名，作为后续上传请求中的 `client_id`：

```bash
hostname
```

将输出的字符串原样记录为 `client_id`。**不得**自行编造或使用其他值；若命令失败，终止并向用户报告。

---

## 步骤 2 —— 询问是否包含后端服务

使用 `question` 工具单独提问一次：

```
question: 该应用是否包含后端服务？
header: 后端服务
options:
  - 不包含（纯前端）
  - 包含（容器化部署）
```

- 用户选择 **不包含（纯前端）** → 进入 **static 子流水线**（步骤 3 → 4 → 5 → 6 → 7 → 8）
- 用户选择 **包含（容器化部署）** → 进入 **backend 子流水线**（步骤 3 → 3b → 5 → 7 → 8；步骤 4/6 由 3b 取代）

---

## 步骤 3 —— 探测项目类型

复用 `deploy-website` 中的探测逻辑。根据步骤 2 的选择分流：

### static 分支（不包含后端）

| 探测结果 | 走向 |
|---|---|
| 存在 `package.json`（Node 项目） | → **Node 构建分支**（步骤 4 分支 B/C） |
| 仅有 `index.html` / 静态 HTML 文件 | → **静态分支**（步骤 4 分支 A） |
| 探测到 PHP / Python / Go / Ruby / Java / Rust / Django / Rails | → 回复 `检测到后端项目（{lang}），如需发布请在步骤 2 选择"包含（容器化部署）"。` 并终止 |

#### 包管理器探测（仅 Node 项目）

优先级顺序：
1. `pnpm-lock.yaml` → `pnpm`
2. `yarn.lock` → `yarn`
3. `package-lock.json` → `npm`
4. 都没有 → 默认 `npm`

#### 构建命令解析（仅 Node 项目）

按优先级：
1. `package.json` 的 `scripts.build` → `<pkgMgr> run build`
2. 已知框架默认产物目录——Vite/CRA/Astro → `dist`，Next.js 静态导出 → `out`，react-scripts → `build`
3. README 兜底：扫描 `README*` 中包含关键字 `build` / `compile` / `dist` 的命令并提取
4. 以上都失败 → **询问用户**指定构建命令，不要猜测

#### 预期产物目录

记录预期的输出目录（`dist` / `out` / `build`），供步骤 4 使用。

### backend 分支（包含后端）

探测项目语言/框架，至少覆盖以下场景：

| 探测特征 | 推断类型 |
|---|---|
| `package.json` 中出现 `express` / `fastify` / `koa` / `hapi` | Node-Express 系 |
| `requirements.txt` / `pyproject.toml` 中含 `fastapi` / `uvicorn` | FastAPI |
| `manage.py` + `requirements.txt` 含 `django` + `gunicorn` | Django + gunicorn |
| 顶层 `pom.xml` / `build.gradle` 且产物为 `*.jar`（Spring Boot） | Spring Boot jar |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| 其他 | 询问用户基础镜像与启动命令，不要猜测 |

记录推断结果，供步骤 3b 生成 Dockerfile 使用。

---

## 步骤 3b —— backend 子流水线

> 仅当步骤 2 选择「包含（容器化部署）」时执行。完成后跳过步骤 4/6 直接进入步骤 5、7、8。

### 3b.1 生成 Dockerfile

AI 基于步骤 3 的探测结果生成**多阶段 alpine** `Dockerfile`，写入 `/tmp/Dockerfile`。**不得**把 Dockerfile 写到用户工作目录。

硬性写法约束：

- 最终（runtime）stage 必须基于 alpine 或 alpine 风味的语言镜像（如 `eclipse-temurin:21-alpine-jdk`）
- runtime stage **禁止** `apt-get` / `dnf` / `yum`
- **禁止** `ADD <url>` —— 所有外部资源在 builder stage 用 `RUN curl/wget` 显式落盘
- 必须多阶段；runtime stage 只 `COPY --from=builder` 编译产物 / 运行时依赖
- `CMD` 必须使用 exec 形式（JSON 数组），例如 `CMD ["node","server.js"]`
- 必须 `EXPOSE <service_port>`，且与后续 multipart 字段 `service_port` 一致
- **所有 `FROM` 引用的 Docker Hub 镜像必须加 `docker.1ms.run/` 代理前缀**：
  - 无 namespace 的官方镜像（`alpine` / `node` / `python` / `golang` / `nginx` / `rust` / `caddy` 等）必须插入 `library/`：`FROM docker.1ms.run/library/alpine:3.20`
  - 已有 namespace 的镜像（如 `eclipse-temurin/...`）**不要**再插 `library/`：`FROM docker.1ms.run/eclipse-temurin:21-alpine-jdk`
  - `FROM scratch` **不走**代理，保留原样
  - 该前缀只在生成 Dockerfile 时注入；showcase 服务端 load 镜像后引用本地 image id，不再受代理影响

### 3b.2 确认容器运行时（必须）

**优先使用 `docker`；只有当 `docker` 不可用时，才回退到 `podman`。**

```bash
if command -v docker >/dev/null 2>&1; then
  RUNTIME=docker
elif command -v podman >/dev/null 2>&1; then
  RUNTIME=podman
else
  # 都不在 PATH 中 → 通过包管理器安装 podman（不要尝试装 docker daemon）
  if command -v apt-get >/dev/null 2>&1; then
    sudo apt-get update && sudo apt-get install -y podman
  elif command -v dnf >/dev/null 2>&1; then
    sudo dnf install -y podman
  elif command -v yum >/dev/null 2>&1; then
    sudo yum install -y podman
  elif command -v apk >/dev/null 2>&1; then
    sudo apk add --no-cache podman
  elif command -v pacman >/dev/null 2>&1; then
    sudo pacman -S --noconfirm podman
  elif command -v brew >/dev/null 2>&1; then
    brew install podman && podman machine init && podman machine start
  else
    echo "无可用的包管理器，无法安装容器运行时" >&2
    exit 1
  fi
  RUNTIME=podman
fi
echo "using container runtime: $RUNTIME"
```

后续步骤一律使用 `"$RUNTIME"` 代替字面 `docker`，因 docker / podman CLI 在 build / run / save 路径上参数兼容（podman 是 rootless，第一次跑可能需要 `podman system migrate` 一次，遇到再处理）。

### 3b.3 本地 build

```bash
TAG="showcase-publish-$(openssl rand -hex 4):tmp"
"$RUNTIME" build -t "$TAG" -f /tmp/Dockerfile .
```

build 失败时：打印 stderr 末段（最多 200 行），**立即终止**，**不得**继续上传。

### 3b.4 本地 run + healthcheck

选择一个未占用的 host 端口（脚本探测，不要硬编码）；`$SVC` 为生成 Dockerfile 时确定的容器内业务端口（service_port）。

```bash
"$RUNTIME" run -d --rm \
  --cpus=1 --memory=1g --memory-swap=1g \
  -p $HOST:$SVC \
  --name "${TAG%:*}-run" \
  "$TAG"
```

**禁止** `--privileged`、`--network host`、`build context 之外的 bind mount`。

AI 根据应用类型选择 healthcheck path（如 `/`、`/healthz`、`/api/health`）与可接受状态码集合（默认 `{200,204,302,401}`）。在 30s 内每 2s 探测一次：

```bash
for i in $(seq 1 15); do
  code=$(curl -s -o /dev/null -w "%{http_code}" "http://127.0.0.1:$HOST<healthcheck_path>" || echo 000)
  case "$code" in
    200|204|302|401) ok=1; break;;
  esac
  sleep 2
done
```

任一失败（容器启动失败 / healthcheck 30s 内未命中可接受状态码）：

1. 打印 `"$RUNTIME" logs <container>` 末段（≤200 行）
2. `"$RUNTIME" stop <container>` + `"$RUNTIME" rmi $TAG` + `rm /tmp/Dockerfile`
3. **立即终止**，**不得**继续上传

### 3b.5 导出镜像

healthcheck 通过后：

```bash
"$RUNTIME" stop "${TAG%:*}-run"
# 服务端走 Docker daemon 加载，必须输出 docker-archive 格式；
# docker save 默认即此格式，podman save 必须显式指定 --format docker-archive。
if [ "$RUNTIME" = "podman" ]; then
  "$RUNTIME" save --format docker-archive "$TAG" | gzip -1 > /tmp/showcase-image.tar.gz
else
  "$RUNTIME" save "$TAG" | gzip -1 > /tmp/showcase-image.tar.gz
fi
```

**强制自检**：

```bash
size=$(stat -c%s /tmp/showcase-image.tar.gz)
test "$size" -le $((500*1024*1024)) || { echo "镜像超过 500MB"; exit 1; }
```

超过 500MB → 终止并提示用户精简产物（多阶段编译 + alpine + 仅拷贝必要文件）。

### 3b.6 准备 multipart 字段

记录后续步骤 8 需要的字段：

- `kind=backend`
- `site_image=@/tmp/showcase-image.tar.gz`
- `service_port=<容器内业务端口>`
- `healthcheck_path=<3b.4 中使用的 path>`

> 后端分支**不**生成 `/tmp/dist.zip`、**不**携带 `site_zip_file`。

### 3b.7 收尾（仅在步骤 8 成功 / 失败后均需执行）

```bash
"$RUNTIME" rmi "$TAG" 2>/dev/null || true
rm -f /tmp/Dockerfile
rm -f /tmp/showcase-image.tar.gz
```

---

## 步骤 4 —— 准备静态产物（仅 static 分支）

> backend 分支跳过本节，直接进入步骤 5。

目标：得到一个**顶层直接包含 `index.html`** 的目录（记为 `<artifact_dir>`）。

打包前先清理旧产物：

```bash
rm -f /tmp/dist.zip
```

### 分支 A —— 纯静态 HTML 项目
直接使用项目根目录作为 `<artifact_dir>`。

### 分支 B —— 已有 dist 的 Node 项目
若预期产物目录存在且**包含 `index.html`**，则将其作为 `<artifact_dir>`。

### 分支 C —— 无 dist 的 Node 项目（需构建）

1. 若 `node_modules` 不存在，执行 `<pkgMgr> install`。失败时输出 stderr 尾部并**终止**。
2. 执行解析出的构建命令。失败时输出 stderr 尾部并**终止**，不得盲目重试。
3. 定位 `index.html`：
   - 先在预期产物目录内查找
   - 找不到则兜底：`find . -maxdepth 3 -name index.html -not -path './node_modules/*'`
   - 多个候选时取**路径最短**者
   - 仍未找到 → 终止并报告 `构建完成但未找到 index.html`

---

## 步骤 5 —— 生成并确认应用元数据

**先自动生成，再逐项询问用户**。每个字段单独发起一次 `question` 工具调用，**不得**把多个字段合并到一个问题里。

### 5a. 基于应用内容自动生成 `site_name` 与 `site_description`

- **static 分支**：从 `<artifact_dir>/index.html` 的 `<title>` / `<meta name="description">` / `<h1>` / 首屏正文综合生成；Node 项目可参考根 `package.json` 的 `name` 与 `description`
- **backend 分支**：从项目根 `package.json` / `pyproject.toml` / `pom.xml` / `Cargo.toml` / `go.mod` 中的 name + description 综合生成；若有 README，提取首段简介

输出：
- 自动生成的 `site_name`（一句话短标题，<= 30 字）
- 自动生成的 `site_description`（一句话简介，<= 80 字）

> 若无可解析内容，则在后续询问中**不要给出"满意"选项的默认值**，让用户必须自行输入。

### 5b. 询问应用名称（`question` 工具，独立一次调用）

```
question: 自动识别到的应用名称为「<生成的 site_name>」，是否使用？
header: 应用名称
options:
  - 满意，就用这个
```

- 用户选 **满意，就用这个** → 采用自动生成的值
- 用户走 **Other** 自行输入 → 采用其输入

### 5c. 询问应用描述（`question` 工具，独立一次调用）

```
question: 自动识别到的应用描述为「<生成的 site_description>」，是否使用？
header: 应用描述
options:
  - 满意，就用这个
```

- 处理逻辑同 5b。

### 5d. 询问应用作者（`question` 工具，独立一次调用）

```
question: 请输入应用作者的 ID（可在下方选项中选择或自行输入）
header: 应用作者
options:
  - 匿名作者
```

- 用户选 **匿名作者** → `site_author = "anonymous"`
- 用户走 **Other** 自行输入 → 采用其输入

---

## 步骤 6 —— 打包（仅 static 分支）

> backend 分支跳过本节；产物已在 3b.4 准备完毕。

必须先 `cd` 进入 `<artifact_dir>` 再打包，确保 zip 内没有包裹目录；同时**排除开发相关文件**：

```bash
cd <artifact_dir> && zip -r /tmp/dist.zip . \
  -x ".git/*" \
  -x ".git" \
  -x "node_modules/*" \
  -x "node_modules" \
  -x "src/*" \
  -x ".env*" \
  -x "*.log" \
  -x ".DS_Store" \
  -x ".vscode/*" \
  -x ".idea/*" \
  -x "package.json" \
  -x "package-lock.json" \
  -x "pnpm-lock.yaml" \
  -x "yarn.lock" \
  -x "tsconfig*.json" \
  -x "*.ts" \
  -x "*.tsx" \
  -x "vite.config.*" \
  -x "webpack.config.*" \
  -x "next.config.*"
```

> **说明**：
> - 对于**分支 B/C**（产物目录在 `dist`/`out`/`build` 内），目录本身已是构建后的静态资源，大多数排除项不会命中，但保留排除规则作为防御性兜底
> - 对于**分支 A**（产物目录就是项目根），上述排除项可有效避免把源码、依赖、版本控制目录、配置文件打入包
> - 如果 `<artifact_dir>` 中确有需要的 `.ts`/`.tsx` 资源（极少见），需调整排除项；否则保持上述默认

**强制自检**：

```bash
unzip -l /tmp/dist.zip | head -30
```

必须满足：
- `index.html` 位于**顶层**（无任何路径前缀）
- 输出中**不应**出现 `.git/`、`node_modules/`、`src/`、`package.json` 等开发文件

任一不满足则立即终止——**不得**上传不合规的包。

---

## 步骤 7 —— 确定 `ticket` (即 **密钥**)

判断当前会话是否已有缓存的密钥 (ticket)：

- **已有缓存的密钥**（即本会话此前已成功提交过应用） → 直接复用，**跳过 7a**，进入步骤 8
- **没有缓存的密钥**（本会话首次提交） → 进入 7a 询问用户

### 7a. 询问是否复用既有应用（首次提交时执行一次）

使用 `question` 工具，**只提供一个显式备选项**；剩下的 Other 输入框本身就代表"有，输入密钥更新已有应用"——其 placeholder 即为该文案：

```
question: 之前在其他任务中提交过本应用吗？是需要更新已有应用，还是提交新应用？如果需要更新，请选择【其他】并填入之前任务提供的密钥。
header: 是否更新现有应用？
options:
  - 没有，提交新应用
  # Other: 输入框的 placeholder/语义为"有，输入密钥即可更新现有应用"，用户在此处直接填密钥
```

- 用户选择 **没有，提交新应用** → 密钥留空（不携带 `ticket` 字段）
- 用户在 **Other** 输入框中填入密钥 → 取用户输入的字符串作为 `ticket`
- 若用户输入的内容为空字符串或纯空格 → 视为未提供，按"提交新应用"处理

> **跨 kind 提示**：若用户提供了 ticket 且本次 kind 与他记忆中的原应用 kind 不一致（无法在 client 侧自动判定，按用户口述），必须在提交前提示「将把原应用从 X 切换为 Y，并重新进入待审核状态」并取得确认；服务端会在切换时整体替换原应用。

---

## 步骤 8 —— 发布

通过 multipart 表单 POST 一次性提交所有字段与产物到 showcase API。

### 8a. 调用 API

**static 分支**（不带 ticket / 带 ticket 二选一）：

```bash
curl -f -X POST \
  -F "client_id=<client_id>" \
  -F "kind=static" \
  -F "site_name=<应用名称>" \
  -F "site_author=<应用作者>" \
  -F "site_description=<应用描述>" \
  [ -F "ticket=<ticket>" ] \
  -F "site_zip_file=@/tmp/dist.zip" \
  https://ugc-submit.showcase.monkeycode-ai.online/v1/create
```

**backend 分支**：

```bash
curl -f -X POST \
  -F "client_id=<client_id>" \
  -F "kind=backend" \
  -F "site_name=<应用名称>" \
  -F "site_author=<应用作者>" \
  -F "site_description=<应用描述>" \
  [ -F "ticket=<ticket>" ] \
  -F "site_image=@/tmp/showcase-image.tar.gz" \
  -F "service_port=<容器内业务端口>" \
  -F "healthcheck_path=<healthcheck path>" \
  https://ugc-submit.showcase.monkeycode-ai.online/v1/create
```

字段说明：

| 字段 | static | backend | 来源 |
|---|---|---|---|
| `client_id` | 必填 | 必填 | 步骤 1 中 `hostname` 命令的输出 |
| `kind` | 必填（`static`） | 必填（`backend`） | 步骤 2 的选择 |
| `site_name` | 必填 | 必填 | 步骤 5b |
| `site_author` | 必填 | 必填 | 步骤 5d |
| `site_description` | 必填 | 必填 | 步骤 5c |
| `site_zip_file` | 必填 | 不得出现 | 步骤 6 |
| `site_image` | 不得出现 | 必填 | 步骤 3b.4 |
| `service_port` | 不得出现 | 必填 | 步骤 3b |
| `healthcheck_path` | 不得出现 | 选填（默认 `/`） | 步骤 3b.3 |
| `ticket` | 选填 | 选填 | 步骤 7 |

要点：

- `-f` 让 HTTP 非 2xx 状态返回非零退出码
- 失败最多**重试 1 次**（应对网络抖动）
- 所有字段都需经过 shell 转义
- **不得**额外传入 `user_id` / `task_id` 等字段
- **不得**混用 kind 与产物字段（如 `kind=static` 同时携带 `site_image`）

### 8b. 解析响应

服务端响应结构：

```json
{
  "status": 200,
  "data": {
    "message": "success or error detail",
    "site_url": "https://xxxxx.showcase.monkeycode-ai.online",
    "ticket": "<会话内复用此 ticket>"
  }
}
```

处理规则：
- `status` 为 2xx 且 `data.site_url` 非空 → 视为成功，提取 `site_url`
- **若 `data.ticket` 非空**：将其**缓存到当前会话上下文**，本会话后续每次提交都自动使用该 `ticket`，不得再次询问用户
- **若 `data.ticket` 与请求中携带的 `ticket` 不一致**：必须在最终反馈中**明确告知用户新的 `ticket` 值**
- 其他情况 → 视为失败，将 `data.message` 作为错误原因向用户报告，**不得**伪装成功

### 8c. 向用户反馈

成功时（`site_url` 必须**单独一行**作为可点击链接渲染，**不得**包在代码块里；**不得**提示绑定微信或公众号）：

```
应用已提交发布，预览地址：

<site_url>
```

**若服务端返回的 `data.ticket` 与请求中携带的 `ticket` 不同**：

```
应用已提交发布，预览地址：

<site_url>

本应用的更新凭证 ticket 为：`<new_ticket>`
后续如需继续更新本应用，请在新会话中向我提供此 ticket。
```

失败时报告 HTTP 状态码与 `data.message`，**不得**编造应用地址。

---

## 硬性规则（必须遵守）

### 通用

- **不得自行编造 `client_id`**：必须来自 `hostname` 命令的真实输出
- **`ticket` 仅在本会话首次提交时询问用户**；首次提交成功拿到的 `ticket` 必须缓存到会话上下文，后续提交自动复用，**不得**反复询问
- **不得自行编造 `ticket`**：要么来自用户输入，要么来自服务端返回
- **应用名称/描述的自动生成必须基于真实应用内容**，不得凭空捏造；用户提供的输入优先级最高
- **应用元数据三项必须分三次 `question` 工具询问**，不得合并
- **不得在请求中传 `user_id` / `task_id`**
- **不得伪装成功**：任一步失败必须如实报告
- **不得轮询审核状态**：Skill 在上传后即结束
- **不得在最终反馈中提示绑定微信或公众号**
- **不得在最终反馈中把 `site_url` 包入代码块**
- **服务端返回的 `data.ticket` 与请求携带的 `ticket` 不一致时，必须在最终反馈中显式告知用户新的 `ticket`**
- **必须按 `status` 与 `data.site_url` 同时判定成功**
- **跨 kind 切换不得在 client 侧报错**；必须提示「将把原应用从 X 切换为 Y，并重新进入待审核状态」

### static 分支专用

- **打包前必须 `rm -f /tmp/dist.zip`**
- **zip 根目录必须是 `index.html`** 且**不得**包含 `.git`、`node_modules`、`src`、`package.json` 等开发文件，通过 `unzip -l` 验证

### backend 分支专用（Dockerfile + 容器化）

- 最终（runtime）stage 必须基于 **alpine**（或 alpine 风味的语言镜像，如 `eclipse-temurin:21-alpine-jdk`）
- runtime stage 禁止 `apt-get` / `dnf` / `yum`
- 禁止 `ADD <url>`
- 必须**多阶段**；runtime stage **只 COPY 产物**，不得编译
- `CMD` 必须为 **exec 形式**（JSON 数组）
- 必须 `EXPOSE` 与 multipart 字段 `service_port` 一致的端口
- **Skill 不得把 Dockerfile 写到用户工作目录**；必须写到 `/tmp/Dockerfile`
- **优先使用 `docker`，缺失时必须用包管理器安装 `podman` 后继续**，绝不可手动安装 docker engine；所有 build / run / save 命令统一以 `"$RUNTIME"` 引用
- 本地 `"$RUNTIME" run` **禁止** `--privileged`、`--network host`、build context 之外的 bind mount
- **build / run / healthcheck 任一失败必须 abort 并打印 stderr 末段，禁止继续上传**
- 镜像 tar.gz 必须 ≤ 500MB
- 收尾必须执行 `"$RUNTIME" rmi <tag>`、`rm /tmp/Dockerfile`、`rm /tmp/showcase-image.tar.gz`（无论成功失败）
- **所有 `FROM` 引用的 Docker Hub 镜像必须加 `docker.1ms.run/` 代理前缀**：
  - 无 namespace 的官方镜像必须插入 `library/`（如 `docker.1ms.run/library/alpine:3.20`、`docker.1ms.run/library/node:20-alpine`）
  - 已有 namespace 的镜像**不要**再插 `library/`（如 `docker.1ms.run/eclipse-temurin:21-alpine-jdk`）
  - `FROM scratch` **不走**代理

---

## 错误处理速查

| 失败点 | 处理动作 |
|---|---|
| `hostname` 命令失败 | 报告错误并终止 |
| 找不到项目根 | 询问用户路径，不得猜测 |
| static 分支探测到后端语言 | 提示用户改回步骤 2 选「包含」并终止本次 |
| 构建命令无法解析 | 询问用户指定命令 |
| `install` 失败 | 输出 stderr 尾部并终止 |
| `build` 失败（前端 / `"$RUNTIME" build`） | 输出 stderr 尾部并终止 |
| 既无 `docker` 又无 `podman`，且包管理器不可用 | 报告"无可用容器运行时"并终止 |
| 构建后找不到 `index.html` | 输出目录结构并终止 |
| 应用内容无任何可用元数据 | 自动生成留空，由用户在 `question` 工具的 Other 中输入 |
| zip 自检发现开发文件混入 | 调整排除项重新打包；仍存在则终止 |
| 镜像 tar.gz > 500MB | 提示精简产物（多阶段编译 + alpine + 仅拷贝必要文件）并终止 |
| `"$RUNTIME" run` 启动失败 | 打印 `"$RUNTIME" logs` 末段，清理容器/镜像/Dockerfile，终止 |
| healthcheck 30s 内未命中可接受状态码 | 打印 `"$RUNTIME" logs` 末段，清理，终止 |
| API 请求非 2xx | 重试 1 次；仍失败则报告 `status` 与 `data.message` 后终止 |
| `data.site_url` 为空 | 视为失败，报告 `data.message` 后终止 |
| 用户输入的 `ticket` 实际无效（API 返回错误） | 报告 `data.message` 后终止；不得自动转为新建应用 |
| `data.ticket` 缺失（旧版服务端） | 仍按成功处理，但本会话后续提交无法走更新流程 |
| API 返回 `kind_mismatch` | 提示「上传字段与所选类型不符，本次发布已取消」并终止 |
| API 返回 `image_too_large` | 提示「镜像超过 500MB，请精简产物（多阶段编译 + alpine + 仅拷贝必要文件）」并终止 |
| API 返回 `image_invalid` | 提示「镜像 tar 校验失败，请确认 `"$RUNTIME" save` 流程未中断；如使用 podman 须显式 `--format docker-archive`」并终止 |
| API 返回 `container_start_failed` | 透传 `data.detail`（≤200 字）并终止 |
| API 返回 `healthcheck_failed` | 提示「服务端启动后 healthcheck 失败：<detail>，请本地复跑 `"$RUNTIME" run` + curl 自查」并终止 |
