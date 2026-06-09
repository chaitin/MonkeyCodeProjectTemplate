---
name: publish-website
description: 将当前会话中生成的 Web 项目发布为线上托管应用。负责环境信息获取、项目类型探测、前端构建（复用 deploy-website 的探测规则）、基于应用内容自动生成元数据、静态资源打包，并通过 multipart 表单 POST 到 showcase API 完成上传。当前仅支持纯静态与 Node.js 前端项目，暂不支持带后端的服务。
arguments:
  - name: workspace
    description: 待发布项目的绝对路径，默认使用当前工作目录
    required: false
---

# 发布应用（Publish Website）

将纯前端项目发布为线上应用。本 Skill 以严格的流水线方式执行——**不得跳过步骤**，**不得在未完成时声称成功**。

## 流水线总览

1. 获取环境信息（client_id）
2. 询问是否包含后端服务（不支持则提前终止）
3. 探测项目类型（复用 `deploy-website` 探测规则）
4. 准备静态产物（必要时构建）
5. 基于应用内容自动生成应用名称与描述，再向用户逐项确认；询问应用作者
6. 打包为 `/tmp/dist.zip`（排除开发相关文件）
7. 确定 `ticket`（首次提交需询问用户是否复用既有应用）
8. 通过 multipart 表单 POST 到 showcase API，缓存返回的 `ticket`，向用户返回 `site_url`

服务端注册与管理员审核发生在 Skill 执行结束**之后**，不属于本 Skill 的职责范围。

### 关于 `ticket`（应用更新密钥）

- `ticket` 是 showcase 服务用来识别"同一个应用"的凭证：首次创建会下发一个 `ticket`，后续若想**更新**该应用而不是新建，需在请求体中带上同一个 `ticket`
- **会话级缓存**：当前会话中**首次**成功创建应用后拿到的 `ticket`，必须缓存于会话上下文（例如记在内存/笔记中），同一会话内后续每次提交都自动使用该 `ticket`，**不得**再向用户询问
- 仅当**本会话从未提交过应用**时，才需要询问用户是不是要复用其他任务中创建的应用（步骤 7a）

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
  - 不包含
  - 包含（暂不支持）
```

- 若用户选择 **包含（暂不支持）**：回复 `当前暂不支持带后端的服务发布，本次发布已取消。` 并**立即终止 Skill**，不得进入后续任何步骤。
- 若用户选择 **不包含**：继续。

---

## 步骤 3 —— 探测项目类型

复用 `deploy-website` 中的探测逻辑，但分流判定不同：

| 探测结果 | 走向分支 |
|---|---|
| 存在 `package.json`（Node 项目） | → **Node 构建分支** |
| 仅有 `index.html` / 静态 HTML 文件 | → **静态分支** |
| PHP / Python / Go / Ruby / Java / Rust / Django / Rails / ... | → 视为不支持的后端，回复 `检测到后端项目（{lang}），当前暂不支持发布。` 并终止 |

### 包管理器探测（仅 Node 项目）

优先级顺序：
1. `pnpm-lock.yaml` → `pnpm`
2. `yarn.lock` → `yarn`
3. `package-lock.json` → `npm`
4. 都没有 → 默认 `npm`

### 构建命令解析（仅 Node 项目）

按优先级：
1. `package.json` 的 `scripts.build` → `<pkgMgr> run build`
2. 已知框架默认产物目录——Vite/CRA/Astro → `dist`，Next.js 静态导出 → `out`，react-scripts → `build`
3. README 兜底：扫描 `README*` 中包含关键字 `build` / `compile` / `dist` 的命令并提取
4. 以上都失败 → **询问用户**指定构建命令，不要猜测

### 预期产物目录

记录预期的输出目录（`dist` / `out` / `build`），供步骤 4 使用。

---

## 步骤 4 —— 准备静态产物

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

**先自动生成，再逐项询问用户**。每个字段单独发起一次 `question` 工具 调用，**不得**把多个字段合并到一个问题里。

### 5a. 基于应用内容自动生成 `site_name` 与 `site_description`

从 `<artifact_dir>` 读取以下信息综合生成：
- `<artifact_dir>/index.html` 的 `<title>` 标签
- `<artifact_dir>/index.html` 的 `<meta name="description">`
- 主要可见文本（如 `<h1>`、首屏正文）
- 若是 Node 项目：源项目根 `package.json` 的 `name` 与 `description`
- 关键页面 / 组件中的语义化文本

输出：
- 自动生成的 `site_name`（一句话短标题，<= 30 字）
- 自动生成的 `site_description`（一句话简介，<= 80 字）

> 若 `index.html` 无可解析内容且无 `package.json`，则在后续询问中**不要给出"满意"选项的默认值**，让用户必须自行输入。

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

## 步骤 6 —— 打包

必须先 `cd` 进入 `<artifact_dir>` 再打包，确保 zip 内没有包裹目录；同时**排除开发相关文件**，避免源码/依赖/版本控制目录被打入包：

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

---

## 步骤 8 —— 发布

通过 multipart 表单 POST 一次性提交所有字段与 zip 文件到 showcase API。

### 8a. 调用 API

不带 `ticket`（新建应用）：

```bash
curl -f -X POST \
  -F "client_id=<client_id>" \
  -F "site_name=<应用名称>" \
  -F "site_author=<应用作者>" \
  -F "site_description=<应用描述>" \
  -F "site_zip_file=@/tmp/dist.zip" \
  https://ugc-submit.showcase.monkeycode-ai.online/v1/create
```

带 `ticket`（更新已有应用）：

```bash
curl -f -X POST \
  -F "client_id=<client_id>" \
  -F "site_name=<应用名称>" \
  -F "site_author=<应用作者>" \
  -F "site_description=<应用描述>" \
  -F "ticket=<ticket>" \
  -F "site_zip_file=@/tmp/dist.zip" \
  https://ugc-submit.showcase.monkeycode-ai.online/v1/create
```

字段说明：

| 字段 | 是否必填 | 来源 |
|---|---|---|
| `client_id` | 必填 | 步骤 1 中 `hostname` 命令的输出 |
| `site_name` | 必填 | 步骤 5b 确认后的应用名称 |
| `site_author` | 必填 | 步骤 5d 确认后的应用作者 |
| `site_description` | 必填 | 步骤 5c 确认后的应用描述 |
| `site_zip_file` | 必填 | 步骤 6 生成的 `/tmp/dist.zip` 文件 |
| `ticket` | 选填 | 步骤 7 得到的 `ticket`；为空则不传该字段 |

要点：
- `-f` 让 HTTP 非 2xx 状态返回非零退出码
- 失败最多**重试 1 次**（应对网络抖动）
- 所有字段都需经过 shell 转义，含空格/特殊字符时务必加双引号
- **不得**额外传入 `user_id` / `task_id` 等字段

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
- **若 `data.ticket` 与请求中携带的 `ticket` 不一致**（包括首次提交时服务端下发的新 ticket、或更新场景下服务端返回了与请求不同的 ticket）：必须在最终反馈中**明确告知用户新的 `ticket` 值**，并提示"后续若要继续更新本应用，请使用此 ticket"
- 其他情况 → 视为失败，将 `data.message` 作为错误原因向用户报告，**不得**伪装成功

### 8c. 向用户反馈

成功时按以下结构回复（注意 `site_url` 必须**单独一行**作为可点击链接渲染，**不得**包在代码块里；**不得**提示绑定微信或公众号）：

```
应用已提交发布，预览地址：

<site_url>
```

**若服务端返回的 `data.ticket` 与请求中携带的 `ticket` 不同**（或首次提交时下发了新的 ticket），必须在上述预览地址下方追加一段提示，明确告知用户新的 ticket 值，便于其在新会话中继续更新本应用：

```
应用已提交发布，预览地址：

<site_url>

本应用的更新凭证 ticket 为：`<new_ticket>`
后续如需继续更新本应用，请在新会话中向我提供此 ticket。
```

失败时报告 HTTP 状态码与 `data.message`，**不得**编造应用地址。

---

## 硬性规则（必须遵守）

- **不得跳过后端服务确认**：包含后端的项目必须在任何构建动作前终止
- **不得自行编造 `client_id`**：必须来自 `hostname` 命令的真实输出
- **`ticket` 仅在本会话首次提交时询问用户**；首次提交成功拿到的 `ticket` 必须缓存到会话上下文，后续提交自动复用，**不得**反复询问
- **不得自行编造 `ticket`**：要么来自用户输入，要么来自服务端返回
- **应用名称/描述的自动生成必须基于真实应用内容**，不得凭空捏造；用户提供的输入优先级最高
- **应用元数据三项必须分三次 `question` 工具 询问**，不得合并
- **不得在请求中传 `user_id` / `task_id`**：API 表单字段只包含规定的 5 个
- **不得伪装成功**：任一步失败必须如实报告，不得声称应用已发布
- **不得轮询审核状态**：Skill 在上传后即结束
- **不得在最终反馈中提示绑定微信或公众号**
- **不得在最终反馈中把 `site_url` 包入代码块**：必须单独一行作为链接渲染
- **服务端返回的 `data.ticket` 与请求携带的 `ticket` 不一致时，必须在最终反馈中显式告知用户新的 `ticket`**，并说明后续更新本应用需使用此 ticket
- **打包前必须 `rm -f /tmp/dist.zip`**，避免旧产物残留
- **zip 根目录必须是 `index.html`** 且**不得**包含 `.git`、`node_modules`、`src`、`package.json` 等开发文件，通过 `unzip -l` 验证
- **必须按 `status` 与 `data.site_url` 同时判定成功**，二者缺一不可

---

## 错误处理速查

| 失败点 | 处理动作 |
|---|---|
| `hostname` 命令失败 | 报告错误并终止 |
| 找不到项目根 | 询问用户路径，不得猜测 |
| 用户选择"包含（暂不支持）" | 回复不支持消息并终止 |
| 探测到后端语言 | 回复不支持消息并终止 |
| 构建命令无法解析 | 询问用户指定命令 |
| `install` 失败 | 输出 stderr 尾部并终止 |
| `build` 失败 | 输出 stderr 尾部并终止 |
| 构建后找不到 `index.html` | 输出目录结构并终止 |
| 应用内容无任何可用元数据 | 自动生成留空，由用户在 `question` 工具 的 Other 中输入 |
| zip 自检发现开发文件混入 | 调整排除项重新打包；仍存在则终止 |
| API 请求非 2xx | 重试 1 次；仍失败则报告 `status` 与 `data.message` 后终止 |
| `data.site_url` 为空 | 视为失败，报告 `data.message` 后终止 |
| 用户输入的 `ticket` 实际无效（API 返回错误） | 报告 `data.message` 后终止；不得自动转为新建应用 |
| `data.ticket` 缺失（旧版服务端） | 仍按成功处理，但本会话后续提交无法走更新流程 |
