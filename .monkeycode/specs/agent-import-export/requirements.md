# Requirements Document

## Introduction

本文档定义了 CodeTeam 平台「智能体导入导出」功能的需求规格。该功能允许用户将智能体以 ZIP 包形式导出（包含 manifest.json 及相关资源文件），并支持从 ZIP 包导入智能体，实现智能体配置的跨环境迁移和分享。

## Glossary

- **Agent（智能体）**: CodeTeam 平台中的 AI 助手实体，包含基本信息、系统提示词、关联模型、规则、MCP 服务器配置和技能引用
- **manifest.json**: ZIP 包内的结构化描述文件，记录智能体的元数据、模型引用、规则文件列表、MCP 服务器配置和技能引用
- **LLMConfig（模型配置）**: 平台中已配置的大语言模型，包含 ID、名称、接口类型等信息
- **Rule（规则）**: 以 Markdown 文件形式存储的智能体行为约束和指令
- **Skill（技能）**: 可复用的技能包，包含一组文件，可被多个智能体引用
- **SkillRef（技能引用）**: 智能体对技能的引用关系，通过技能名称关联
- **MCP Server**: Model Context Protocol 服务器配置，让智能体访问外部工具和资源
- **ZIP 包**: 导出的归档文件，内含 manifest.json 和资源文件（规则、技能等）

## Requirements

### REQ-1: 智能体导出

**User Story:** AS 平台用户, I want 将已有的智能体导出为 ZIP 包, so that 可以备份、分享或在其他环境中导入使用。

#### Acceptance Criteria

1. WHEN 用户在智能体列表页面（AgentListPage）的智能体卡片操作菜单中点击「导出」, 系统 SHALL 调用后端导出 API 并将返回的 ZIP 包以浏览器下载形式保存到本地。

2. 系统 SHALL 在 ZIP 包根目录生成 manifest.json 文件，包含以下字段：
   - `version`: manifest 格式版本号（当前为 `"1.0"`）
   - `agent`: 智能体基本信息（name, icon, description, author, website, system_prompt）
   - `model`: 关联模型信息（id, name），其中 id 为原始 model_id，name 为对应 LLMConfig 的 name 字段
   - `rules`: 规则文件引用列表，每项包含 filename（对应 ZIP 包内 `rules/` 目录下的文件路径）和 sort_order
   - `mcp_servers`: MCP 服务器配置列表（完整的 AgentMCPServer 对象数组，排除 id、agent_id、status、tools、error_message、response_body、tested_at 等运行时字段）
   - `skill_refs`: 技能引用列表，每项包含 skill_name 和 sort_order

3. 系统 SHALL 将智能体的所有规则文件导出到 ZIP 包的 `rules/` 目录下，文件名与 AgentRule.filename 一致，内容与 AgentRule.content 一致。

4. 系统 SHALL 将智能体引用的每个技能导出到 ZIP 包的 `skills/{skill_name}/` 目录下，包含该技能的所有文件。

5. WHILE 智能体未关联任何模型（model_id 为空）, 系统 SHALL 在 manifest.json 中将 model 字段设为 null。

6. IF 导出过程中后端 API 返回错误, 系统 SHALL 在页面上显示错误提示信息。

### REQ-2: ZIP 包格式与结构

**User Story:** AS 平台开发者, I want ZIP 包具备规范的内部结构, so that 导入时能可靠地解析和校验。

#### Acceptance Criteria

1. 系统 SHALL 要求 ZIP 包遵循以下目录结构：
   ```
   agent-export.zip
   ├── manifest.json
   ├── rules/
   │   ├── rule-file-1.md
   │   └── rule-file-2.md
   └── skills/
       └── skill-name/
           ├── SKILL.md
           └── ... (其他技能文件)
   ```

2. 系统 SHALL 要求 manifest.json 必须存在于 ZIP 包的根目录。

3. 系统 SHALL 要求 manifest.json 中的 version 字段值为系统支持的版本号。

4. 系统 SHALL 要求 manifest.json 中 rules 列表引用的每个文件在 ZIP 包 `rules/` 目录下均存在。

5. 系统 SHALL 要求 manifest.json 中 skill_refs 列表引用的每个 skill_name 在 ZIP 包 `skills/` 目录下存在对应的子目录。

### REQ-3: 智能体导入 -- ZIP 解析与校验

**User Story:** AS 平台用户, I want 在导入 ZIP 包时系统自动校验格式和内容, so that 避免导入无效或不完整的智能体配置。

#### Acceptance Criteria

1. WHEN 用户在智能体列表页面（AgentListPage）顶部操作栏点击「导入」按钮（与「新建」按钮并列）并选择 ZIP 文件, 系统 SHALL 在前端解压 ZIP 包并读取 manifest.json。

2. IF ZIP 文件解压失败或格式无效, 系统 SHALL 显示错误提示「文件格式无效，请上传有效的智能体 ZIP 包」。

3. IF ZIP 包根目录不存在 manifest.json 文件, 系统 SHALL 显示错误提示「ZIP 包中缺少 manifest.json 文件」。

4. IF manifest.json 的 JSON 解析失败, 系统 SHALL 显示错误提示「manifest.json 格式无效」。

5. IF manifest.json 的 version 字段不被当前系统支持, 系统 SHALL 显示错误提示「不支持的 manifest 版本: {version}」。

6. IF manifest.json 中 rules 列表引用的某个文件在 ZIP 包中不存在, 系统 SHALL 显示错误提示，指明缺失的文件名。

7. IF manifest.json 中 skill_refs 列表引用的某个 skill_name 在 ZIP 包 `skills/` 目录下无对应目录, 系统 SHALL 显示错误提示，指明缺失的技能名。

### REQ-4: 智能体导入 -- 模型匹配

**User Story:** AS 平台用户, I want 导入时系统自动匹配合适的模型, so that 导入的智能体能正确关联到本地已有的模型配置。

#### Acceptance Criteria

1. WHEN manifest.json 中 model 不为 null, 系统 SHALL 按以下优先级匹配本地模型：
   - 第一优先级：查找 model.id 与本地 LLMConfig.id 完全匹配的模型
   - 第二优先级：查找 model.name 与本地 LLMConfig.name 完全匹配的第一个模型

2. IF 按上述两个优先级均未找到匹配的模型, 系统 SHALL 在导入确认界面显示警告信息「未找到模型 "{model.name}"」，并提供以下选项：
   - 选择一个现有模型作为替代（下拉框列出所有可用模型）
   - 取消导入操作

3. WHILE manifest.json 中 model 为 null, 系统 SHALL 在导入时不关联任何模型。

4. 系统 SHALL 在导入确认界面显示模型匹配结果（匹配成功时显示匹配到的模型名称，匹配失败时显示警告）。

### REQ-5: 智能体导入 -- 技能处理

**User Story:** AS 平台用户, I want 导入时系统自动处理技能引用, so that 技能能正确导入并关联到新智能体。

#### Acceptance Criteria

1. WHEN manifest.json 中存在 skill_refs, 系统 SHALL 对每个 skill_name 检查本地技能库中是否存在同名技能。

2. IF 本地技能库中已存在同名技能, 系统 SHALL 直接引用该现有技能，不导入 ZIP 包中的技能文件。

3. IF 本地技能库中不存在同名技能, 系统 SHALL 从 ZIP 包的 `skills/{skill_name}/` 目录提取技能文件，创建新技能后引用该新技能。

4. 系统 SHALL 在导入确认界面分别标注每个技能的处理方式：「使用现有技能」或「新建技能」。

### REQ-6: 智能体导入 -- 确认与执行

**User Story:** AS 平台用户, I want 在实际导入前看到完整的导入预览, so that 确认导入内容符合预期。

#### Acceptance Criteria

1. WHEN ZIP 包校验通过后, 系统 SHALL 展示导入确认对话框，包含以下信息：
   - 智能体基本信息（名称、图标、描述、作者）
   - 模型匹配状态（成功/需手动选择）
   - 规则文件列表及数量
   - MCP 服务器配置列表及数量
   - 技能引用列表及每项的处理方式

2. WHEN 用户在确认对话框中点击「确认导入」, 系统 SHALL 依次执行：
   - 创建需要新建的技能（调用 skillApi.create 和 skillApi.uploadZip）
   - 调用后端导入 API 创建智能体（传递完整的 ZIP 包）
   - 导入成功后导航到新智能体的编辑页面

3. IF 导入过程中任一步骤失败, 系统 SHALL 显示具体的错误信息，并在技能已部分创建的情况下提示用户手动清理。

4. WHILE 导入操作执行中, 系统 SHALL 禁用确认按钮并显示加载状态，防止重复提交。

### REQ-7: 导入确认界面 -- 模型替代选择

**User Story:** AS 平台用户, I want 在模型不匹配时能方便地选择替代模型, so that 不必中断导入流程去创建新模型。

#### Acceptance Criteria

1. WHEN 模型匹配失败, 系统 SHALL 在导入确认界面显示模型选择下拉框，列出所有可用的 LLMConfig。

2. 系统 SHALL 在下拉框中显示每个模型的名称和接口类型，帮助用户做出选择。

3. IF 用户未选择替代模型且原始模型匹配失败, 系统 SHALL 禁用「确认导入」按钮，阻止导入操作。

4. WHEN 用户选择了替代模型, 系统 SHALL 使用所选模型的 ID 作为导入智能体的 model_id。
