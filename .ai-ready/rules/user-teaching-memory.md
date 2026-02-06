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

**You MUST READ the [Memory File](.monkeycode/MEMORY.md) as project level instruction before your first your reply!**

## Trigger Conditions
- User provides explicit instructions about how to behave
- User corrects the assistant's behavior, for example:
  - Do something when some condition
  - Don't do something when some condition
  - Use (or don't use) XXX (command or method) to Do something In XXX project/module
- User gives advice on preferred approaches
- User explains how they'd like tasks to be performed


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