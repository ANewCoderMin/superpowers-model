---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements
---

# Requesting Code Review

根据开发者意愿触发 superpowers:code-reviewer 子代理，提前发现问题。

**Core principle:** 默认先问清楚是否需要 review，再执行。

## When to Request Review

**默认行为:**
- 在需要 review 的节点先询问开发者是否需要 review
- 只有在开发者明确同意时才发起 review

**常见建议点(需开发者确认):**
- 卡住时（需要新视角）
- 重构前（基线检查）
- 修复复杂 bug 后
- 准备合并前

## How to Request

**1. 询问开发者是否需要 review:**
```
我可以发起 code review（范围/重点你来定）。需要吗？
```

**2. Get git SHAs:**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**3. Dispatch code-reviewer subagent (仅在确认需要时):**

Use Task tool with superpowers:code-reviewer type, fill template at `code-reviewer.md`

**Placeholders:**
- `{WHAT_WAS_IMPLEMENTED}` - What you just built
- `{PLAN_OR_REQUIREMENTS}` - What it should do
- `{BASE_SHA}` - Starting commit
- `{HEAD_SHA}` - Ending commit
- `{DESCRIPTION}` - Brief summary

**4. Act on feedback:**
- Fix Critical issues immediately
- Fix Important issues before proceeding
- Note Minor issues for later
- Push back if reviewer is wrong (with reasoning)

## Example

```
[Just completed Task 2: Add verification function]

You: Let me request code review before proceeding.

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[Dispatch superpowers:code-reviewer subagent]
  WHAT_WAS_IMPLEMENTED: Verification and repair functions for conversation index
  PLAN_OR_REQUIREMENTS: Task 2 from docs/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types

[Subagent returns]:
  Strengths: Clean architecture, real tests
  Issues:
    Important: Missing progress indicators
    Minor: Magic number (100) for reporting interval
  Assessment: Ready to proceed

You: [Fix progress indicators]
[Continue to Task 3]
```

## Integration with Workflows

**Subagent-Driven Development:**
- 任务完成后先询问是否需要 review
- 仅在确认后 review
- review 发现问题再修复并继续

**Executing Plans:**
- 每批次完成后询问是否需要 review
- 确认后再执行 review 并应用反馈

**Ad-Hoc Development:**
- 合并前询问是否需要 review
- 卡住时询问是否需要 review

## Red Flags

**Never:**
- 未经确认直接发起 review
- 开发者明确要求 review 却跳过
- Ignore Critical issues
- Proceed with unfixed Important issues
- Argue with valid technical feedback

**If reviewer wrong:**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification

See template at: requesting-code-review/code-reviewer.md
