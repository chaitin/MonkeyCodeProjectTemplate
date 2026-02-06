# Auto Create Branch on Master/Main Rule

当 agent 准备对 workspace 中的文件进行写操作时，如果检测到当前分支是 master 或 main，需要先获取当前日期，根据任务上下文生成一个符合规范的分支名并切换到该分支。

## 触发条件

仅当满足以下**所有条件**时触发分支创建：
1. 当前目录是一个 git 仓库（存在 `.git` 目录）
2. 当前分支是 `master` 或 `main`
3. Agent 准备执行会对 workspace 文件进行**写操作**（如 Edit、Write、NotebookEdit）

## 执行步骤

### 1. 获取当前日期

```bash
date +%y%m%d
```

例如：260130（表示 2026 年 1 月 30 日）

### 2. 分析任务类型

根据当前任务上下文，判断任务类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能开发 | 新增登录页面、添加用户管理功能 |
| `fix` | Bug 修复 | 修复登录失败问题、解决页面加载慢 |
| `chore` | 杂项/工具链 | 更新依赖、修改配置文件、添加文档 |
| `refactor` | 代码重构 | 重构组件、优化代码结构 |

### 3. 生成任务摘要

从当前任务上下文中提取关键词，生成简短的英文摘要（使用小写字母，单词之间用连字符分隔）：

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

假设今天是 2026 年 2 月 3 日，任务上下文是："帮我添加一个用户登录页面"

当 Agent 准备执行 Edit/Write 操作修改文件时：

1. 获取日期：`260203`
2. 判断类型：这是一个新功能开发，使用 `feat`
3. 生成摘要：`add-login-page`
4. 组合分支名：`260203-feat-add-login-page`
5. 执行命令：`git checkout -b 260203-feat-add-login-page`
6. 然后继续执行原有的写操作

## 注意事项

- **只在准备修改文件时才触发**：纯读操作（Read、Glob、Grep 等）不会触发分支创建
- 分支名生成后，应直接执行分支创建，无需向用户确认（用户已通过任务描述表达了意图）
- 如果用户明确要求在 master/main 分支上操作，则不需要创建新分支
- 如果当前目录不是 git 仓库，或当前分支不是 master/main，则不需要执行此规则
- 任务摘要的长度应控制在合理范围内，通常不超过 50 个字符
