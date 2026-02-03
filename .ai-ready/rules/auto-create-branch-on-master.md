# Auto Create Branch on Master/Main Rule

当 agent 启动时，如果检测到当前分支是 master 或 main，需要先获取当前日期，根据用户的任务描述生成一个符合规范的分支名并切换到该分支。

## 触发条件

当 agent 启动并检测到以下条件时：
1. 当前目录是一个 git 仓库（存在 `.git` 目录）
2. 当前分支是 `master` 或 `main`
3. 用户的初始输入包含**有意义的任务**（非闲聊）

### 闲聊识别

以下类型的输入被视为**闲聊**，不触发分支创建：

| 类型 | 示例 |
|------|------|
| 问候 | "Hello", "Hi", "Hey there" |
| 天气询问 | "How is the weather today" |
| 寒暄 | "How are you", "What's up" |
| 无意义的输入 | "Test", "123", "???" |

**处理逻辑**：
1. 如果初始输入是闲聊，**不要**创建分支，保持当前 `master` 或 `main` 分支
2. 继续与用户对话，等待用户首次提出**有意义的任务**
3. 当用户提出有意义的任务时，立即执行分支创建流程

### 有意义的任务示例

| 任务描述 | 类型判断 |
|----------|----------|
| "帮我添加一个登录页面" | feat |
| "修复用户注册的 bug" | fix |
| "重构首页代码" | refactor |
| "更新依赖包" | chore |
| "给我解释一下这段代码" | 有意义但不需要切分支 |

## 执行步骤

### 1. 获取当前日期

```bash
date +%y%m%d
```

例如：260130（表示 2026 年 1 月 30 日）

### 2. 分析任务类型

根据用户提供的任务描述，判断任务类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能开发 | 新增登录页面、添加用户管理功能 |
| `fix` | Bug 修复 | 修复登录失败问题、解决页面加载慢 |
| `chore` | 杂项/工具链 | 更新依赖、修改配置文件、添加文档 |
| `refactor` | 代码重构 | 重构组件、优化代码结构 |

### 3. 生成任务摘要

从用户的任务描述中提取关键词，生成简短的英文摘要（使用小写字母，单词之间用连字符分隔）：

- "帮我添加一个登录页面" → `add-login-page`
- "修复用户注册时的验证错误" → `fix-user-registration-validation`
- "重构用户管理的代码" → `refactor-user-management`

### 4. 组合分支名

按照以下格式组合分支名：

```
YYMMDD-(feat|fix|chore|refactor)-xxxxx-xxxx-xxxx
```

示例：
- `260130-feat-add-login-page`
- `260130-fix-user-registration-validation`
- `260130-refactor-user-management`
- `260130-chore-update-dependencies`

### 5. 创建并切换分支

```bash
git checkout -b <branch-name>
```

例如：
```bash
git checkout -b 260130-feat-add-login-page
```

## 完整示例

假设今天是 2026 年 2 月 3 日，用户的任务描述是："帮我添加一个用户登录页面"

1. 获取日期：`260203`
2. 判断类型：这是一个新功能开发，使用 `feat`
3. 生成摘要：`add-login-page`
4. 组合分支名：`260203-feat-add-login-page`
5. 执行命令：`git checkout -b 260203-feat-add-login-page`

## 注意事项

- 分支名生成后，需要向用户确认是否使用该分支名，或是否需要调整
- 如果用户明确要求在 master/main 分支上操作，则不需要创建新分支
- 如果当前目录不是 git 仓库，或当前分支不是 master/main，则不需要执行此规则
- 任务摘要的长度应控制在合理范围内，通常不超过 50 个字符
