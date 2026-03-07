# User Teaching Memory Rule

## Description
When a user teaches or instructs the AI assistant on how to perform a task or behave in a certain way during a chat conversation, these teachings should be recorded in a .monkeycode/MEMORY.md file located in the repository root.

## Implementation
- Monitor chat conversations for teaching-oriented instructions from users
- Before adding a new entry, scan .monkeycode/MEMORY.md for similar or identical instructions
- If a duplicate is found, either skip adding the new entry or merge it with the existing one
- When merging, update the existing entry with new context or date
- If no duplicate is found, append the instruction to .monkeycode/MEMORY.md in the repository root
- Format entries with date, user instruction, and context
- Preserve user privacy while capturing useful behavioral instructions

## Purpose
This creates a persistent memory of user preferences and instructions that can guide future interactions and customizations.

## MUST DO

- You MUST CREATE the [Memory File](.monkeycode/MEMORY.md) according to the **File Template** before your first reply IF IT'S NOT EXIST!
- You MUST READ the [Memory File](.monkeycode/MEMORY.md) as project level instruction before your first reply!
- EVERY user input should be validated by the following **Trigger Conditions**, Once MATCHED, UPDATE the [Memory File](.monkeycode/MEMORY.md) with user's instruction IMMEDIATLY BEFORE DOING ANYTHING

## Trigger Conditions

### 1. User provides EXPLICIT INSTRUCTIONS about how to behave

> Examples:
> - "回复我的时候请用中文"
> - "Always respond in bullet points"
> - "代码注释统一用英文写"
> - "不要在回复中使用 emoji"

### 2. User INSTRUCTS or CORRECTS the assistant's behavior

> Examples:
> - **Do something when some condition**: "每次改完代码后自动跑一下 lint"、"When I ask you to refactor, always write unit tests first"
> - **Don't do something when some condition**: "不要自动删除注释掉的代码"、"Don't modify files outside the src/ directory unless I explicitly say so"
> - **Use (or don't use) XXX to do something in XXX project/module**: "这个项目用 pnpm 不要用 npm"、"Use `pytest` instead of `unittest` in this repo"、"在 backend 模块里不要用 print，用 logger"

### 3. User gives advice on preferred approaches

> Examples:
> - "我更喜欢函数式写法，少用 class"
> - "Prefer composition over inheritance in this project"
> - "写 SQL 的时候用 CTE 而不是子查询"
> - "CSS 优先用 Tailwind 的 utility class，不要写自定义样式"

### 4. User explains how they'd like tasks to be performed

> Examples:
> - "改 bug 之前先帮我写个能复现问题的测试用例"
> - "每次提交代码前先跑 `make check`"
> - "重构的时候一次只改一个文件，改完我确认了再继续"
> - "When adding a new API endpoint, always update the OpenAPI spec first"

## File Template for MEMORY.md

```markdown
# Memory of User Instructions

This file contains a record of user instructions, preferences, and teachings that should be remembered for future interactions.

## Format
Entries should follow this format:

[User instruction summary]
- Date: [YYYY-MM-DD]
- Context: [Where or when this was mentioned]
- Instructions:
  - [What the user taught or instructed, line by line]

## Deduplication Policy
- Before adding a new entry, check for similar or identical instructions
- If a duplicate is found, either skip the new entry or merge with existing one
- When merging, update with new context or date information
- This helps prevent redundant entries and keeps the memory clean

## Entries

[some memory entries in the Format declared above]
```