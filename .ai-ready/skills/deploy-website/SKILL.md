---
name: deploy-website
description: 部署并本地预览 Web 项目。自动检测项目类型（Node.js、PHP、Python/Django/Flask/Gunicorn、Go、Ruby/Rails、Java/Spring Boot、Rust 或静态 HTML），启动对应的开发服务器。
arguments:
  - name: workspace
    description: 要部署的工作区目录的绝对路径
    required: false
---

# 部署网站

自动检测项目类型，使用 `background_terminal_create` **在后台启动** 对应的开发服务器，确认启动成功后通过 MCP 工具 `request_preview` 获取预览地址。

## 项目类型检测逻辑

1. **Node.js Web 项目**
   - 在工作区根目录查找 `package.json`
   - 检查是否包含 `dev` 或 `start` 脚本
   - 执行 `npm run dev`（若无 `dev` 则执行 `npm start`）

2. **静态网站**
   - 在工作区查找 `.html` 文件（尤其是 `index.html`）
   - 执行 `python3 -m http.server 8000` 提供静态文件服务

3. **其他类型网站**

   根据项目特征文件检测类型并使用对应服务器：

   - **PHP 项目** - 查找 `.php` 文件或 `composer.json`
     - 执行 `php -S localhost:8000`
     - 若存在 `public/index.php`，则从 `public` 目录提供服务
   - **Python 项目**
     - **Django** - 查找 `manage.py` 和 `settings.py`
       - 执行 `python manage.py runserver 8000`
     - **Flask** - 查找 `app.py` 或 `wsgi.py`
       - 执行 `python app.py` 或 `flask run --port=8000`
     - **Gunicorn/uWSGI** - 查找 `gunicorn.conf.py` 或 `uwsgi.ini`
       - 执行 `gunicorn -b 0.0.0.0:8000 app:app`（根据实际模块名调整）
     - **通用 Python** - 查找 `requirements.txt` 或 `pyproject.toml`
       - 执行 `python -m http.server 8000`
   - **Go 项目** - 查找 `go.mod`
     - 执行 `go run main.go` 或 `go run .`
   - **Ruby/Rails** - 查找包含 `rails` 的 `Gemfile`
     - 执行 `rails server -p 8000` 或 `bundle exec rails server -p 8000`
   - **Java/Spring Boot** - 查找 `pom.xml`（Maven）或 `build.gradle`（Gradle）
     - 执行 `mvn spring-boot:run` 或 `./gradlew bootRun`
   - **Rust/Axum/Actix** - 查找 `Cargo.toml`
     - 执行 `cargo run`
   - **基于 README 检测** - 若以上方式均未匹配：
     - 搜索 README 文件（`README.md`、`README.rst`、`README.txt`、`doc/README.md`）
     - 查找包含 `run`、`start`、`serve`、`dev`、`preview` 等关键词的命令
     - 提取并执行对应命令

## 工作流程

### 1. 检查是否已有服务器进程

启动新服务器之前，先进行以下检查：

- 调用 `background_terminal_list` 检查是否已有运行中的后台终端
- 检查是否有进程已在监听端口：

```bash
ss -tlnp 2>/dev/null || netstat -tlnp 2>/dev/null || lsof -iTCP -sTCP:LISTEN -P -n
```

若检测到已有服务器进程：
- 从输出中提取端口号
- 跳过启动步骤，直接使用该端口调用 `request_preview`
- 通知用户检测到已有服务器

### 2. 查阅项目文档确定启动命令（优先级最高）

在使用默认检测逻辑之前，**必须先查阅项目文档**，确认项目是否有明确指定的启动方式。按以下顺序读取文档：

1. `.monkeycode/MEMORY.md` - 项目记忆文件，可能包含之前记录的启动方式
2. `README.md` - 项目说明文档，通常包含开发服务器启动命令
3. `AGENTS.md` - Agent 指令文件，可能包含针对 AI 助手的启动指引

**查找目标**：在上述文档中搜索与 Web 服务启动相关的内容，例如：
- 明确的启动命令（如 `npm run dev`、`make serve`、`docker compose up` 等）
- 启动前的前置步骤（如环境变量设置、配置文件生成等）
- 指定的端口号或特殊参数

**判断逻辑**：
- 若文档中有**明确的启动指示**，则以文档描述为准，使用文档中指定的命令启动服务器
- 这个启动命令需要与识别到的项目类型（如开发语言）相匹配
- 向用户明确告知：启动 Web 服务的方式遵循了哪份文档（例如："按照 README.md 中的说明启动开发服务器"）
- 若文档中没有相关描述，或描述不明确，则回退到下方的默认检测逻辑

### 3. 检测项目类型并使用 `background_terminal_create` 启动服务器

仅在未检测到已有服务器、且项目文档中未找到明确启动指示时，执行此步骤的默认检测逻辑。

> **重要**：必须使用 MCP 工具 `background_terminal_create` 启动服务器。该工具在后台终端中运行命令，不会阻塞当前会话，并提供输出日志追踪。

根据项目类型调用 `background_terminal_create` 执行对应命令：

- **Node.js 项目**（存在 `package.json`）：
  - 先在前台执行 `npm install` 确保依赖已安装
  - 再使用 `background_terminal_create` 执行 `npm run dev`（或 `npm start`）

- **PHP 项目**（存在 `composer.json` 或 `.php` 文件）：
  - 使用 `background_terminal_create` 执行 `php -S localhost:8000`
  - 若存在 `public/index.php`，则执行 `php -S localhost:8000 -t public`

- **Python/Django**（存在 `manage.py`）：
  - 使用 `background_terminal_create` 执行 `python manage.py runserver 8000`

- **Python/Flask**（存在 `app.py` 或 `wsgi.py`）：
  - 使用 `background_terminal_create` 执行 `flask run --port=8000` 或 `python app.py`

- **Python/Gunicorn**（存在 `gunicorn.conf.py` 或 `uwsgi.ini`）：
  - 使用 `background_terminal_create` 执行 `gunicorn -b 0.0.0.0:8000 app:app`

- **Python 通用**（存在 `requirements.txt` 或 `pyproject.toml`）：
  - 使用 `background_terminal_create` 执行 `python -m http.server 8000`

- **Go 项目**（存在 `go.mod`）：
  - 使用 `background_terminal_create` 执行 `go run .` 或 `go run main.go`

- **Ruby/Rails**（`Gemfile` 中包含 `rails`）：
  - 使用 `background_terminal_create` 执行 `bundle exec rails server -p 8000`

- **Java/Spring Boot**（存在 `pom.xml` 或 `build.gradle`）：
  - 使用 `background_terminal_create` 执行 `./mvnw spring-boot:run` 或 `./gradlew bootRun`

- **Rust 项目**（存在 `Cargo.toml`）：
  - 使用 `background_terminal_create` 执行 `cargo run`

- **静态 HTML 文件**（存在 `index.html` 或其他 `.html` 文件）：
  - 使用 `background_terminal_create` 执行 `python3 -m http.server 8000`

- **基于 README 检测**（以上均未匹配时）：
  - 从 README 文件中提取启动命令，使用 `background_terminal_create` 执行

### 4. 读取输出日志确认服务器启动成功

`background_terminal_create` 返回的结果中包含子进程的 ID（PID），记录该 PID 以便后续管理。

然后通过以下步骤确认服务器已成功启动：

```
# 第 1 步：调用 background_terminal_output_path 获取日志文件路径
# 第 2 步：使用 Read 工具读取日志文件
# 第 3 步：查找启动成功的标志，例如：
#   - "Listening on port ..."
#   - "Server running at ..."
#   - "ready in ... ms"
#   - "Local: http://localhost:..."
# 第 4 步：若启动失败，读取日志中的错误信息并排查问题
```

若日志显示服务器尚未启动完成，等待几秒后重新读取日志文件，重复此过程直到确认服务器已运行或检测到错误。

### 5. 获取预览地址

调用 MCP 工具 `request_preview`，传入服务器监听的端口号，获取预览 URL。

### 6. 向用户展示预览地址

- 输出可点击的超链接，指向 `request_preview` 返回的预览地址
- 告知用户后台终端 ID，方便后续管理

### 7. 处理 IP 白名单问题

- 若用户反馈因 IP 未加白名单而无法访问，请用户提供拒绝页面上显示的 IP 地址
- 再次调用 MCP 工具 `request_preview`，使用 `additional_ip_whitelist` 字段添加用户的 IP
- `additional_ip_whitelist` 字段接受多个 IPv4 地址，以逗号分隔（例如 `"192.168.1.100,10.0.0.50"`）

## 进程管理

### 查看后台终端

使用 `background_terminal_list` 查看所有运行中的后台终端及其状态。

### 查看服务器输出

使用 `background_terminal_output_path` 获取后台终端的日志文件路径，然后读取日志文件查看服务器输出、错误或运行状态。

### 停止服务器

当用户请求停止服务器时，使用 `background_terminal_kill` 并传入终端 ID 来停止。

**严禁**使用 `pkill` 或 `killall` 加进程名的方式停止服务器（例如 `pkill node`、`killall python`），这可能导致：
- 误杀用户手动启动的进程
- 误杀其他无关的开发服务器
- 误杀同名系统进程

始终使用 `background_terminal_kill` 停止由本 skill 启动的服务器。

## 注意事项

- 必须使用 `background_terminal_create` 启动服务器，确保不阻塞当前会话
- 启动后必须通过 `background_terminal_output_path` 获取日志路径并读取日志，确认服务器已运行后再获取预览地址
- 启动前务必检查是否已有服务器进程，避免端口冲突
- 静态服务器默认端口为 8000
- Node.js 项目的端口取决于项目配置
- 端口冲突时可动态更换端口号
- 使用 `background_terminal_list` 追踪运行中的终端，使用 `background_terminal_kill` 停止终端
- 严禁使用 `pkill` 或 `killall` 停止服务器，必须使用 `background_terminal_kill` 加终端 ID
- 运行服务器前确保依赖已安装：
  - Node.js：`npm install`
  - Python：`pip install -r requirements.txt`（若存在）
  - PHP：`composer install`（若存在 `composer.json`）
  - Ruby/Rails：`bundle install`（若存在 `Gemfile`）
  - Go：`go mod download`（`go run` 会自动处理）
  - Java：Maven/Gradle wrapper 自动处理依赖
  - Rust：`cargo fetch`（`cargo run` 会自动处理）
