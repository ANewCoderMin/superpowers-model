---
name: subagent-driven-development
description: Use when executing implementation plans with independent tasks in the current session
---

# Subagent-Driven Development

在同一会话中由单一执行者按任务顺序执行实施计划，不再派发子代理，不做自动审查。

**Core principle:** 单一执行者顺序执行任务，保持上下文一致，不做自动审查。

## When to Use

```dot
digraph when_to_use {
    "Have implementation plan?" [shape=diamond];
    "Tasks mostly independent?" [shape=diamond];
    "Stay in this session?" [shape=diamond];
    "subagent-driven-development" [shape=box];
    "executing-plans" [shape=box];
    "Manual execution or brainstorm first" [shape=box];

    "Have implementation plan?" -> "Tasks mostly independent?" [label="yes"];
    "Have implementation plan?" -> "Manual execution or brainstorm first" [label="no"];
    "Tasks mostly independent?" -> "Stay in this session?" [label="yes"];
    "Tasks mostly independent?" -> "Manual execution or brainstorm first" [label="no - tightly coupled"];
    "Stay in this session?" -> "subagent-driven-development" [label="yes"];
    "Stay in this session?" -> "executing-plans" [label="no - parallel session"];
}
```

**vs. Executing Plans (parallel session):**
- Same session (no context switch)
- 单一执行者连续推进（不派发子代理）
- 不进行自动审查
- 更少流程开销

## The Process

```dot
digraph process {
    rankdir=TB;

    subgraph cluster_per_task {
        label="Per Task";
        "执行当前任务（实现/按需验证）" [shape=box];
        "记录结果并标记完成" [shape=box];
    }

    "Read plan, extract all tasks with full text, note context, create TodoWrite" [shape=box];
    "More tasks remain?" [shape=diamond];
    "Use superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];

    "Read plan, extract all tasks with full text, note context, create TodoWrite" -> "执行当前任务（实现/按需验证）";
    "执行当前任务（实现/按需验证）" -> "记录结果并标记完成";
    "记录结果并标记完成" -> "More tasks remain?";
    "More tasks remain?" -> "执行当前任务（实现/按需验证）" [label="yes"];
    "More tasks remain?" -> "Use superpowers:finishing-a-development-branch" [label="no"];
}
```

## Prompt Templates

本模式不派发子代理，因此不使用子代理模板。

## Example Workflow

```
You: I'm using Subagent-Driven Development to execute this plan.

[Read plan file once: docs/plans/feature-plan.md]
[Extract all 5 tasks with full text and context]
[Create TodoWrite with all tasks]

Task 1: Hook installation script

[执行 Task 1，按计划实现]
[记录结果并标记 Task 1 完成]

Task 2: Recovery modes

[执行 Task 2，按计划实现]
[记录结果并标记 Task 2 完成]

...

[After all tasks]
[Use finishing-a-development-branch]
```

## Advantages

**vs. Manual execution:**
- 流程化执行但仍由单一执行者完成
- 计划驱动，避免遗漏步骤

**vs. Executing Plans:**
- 同一会话完成（无交接）
- 连续推进，无需批次等待

**Efficiency gains:**
- 无子代理调度成本
- 上下文连续，切换成本低

**Quality gates:**
- 质量把关由开发者人工审查

**Cost:**
- 更少流程成本，但质量依赖开发者人工把关

## Red Flags

**Never:**
- 脱离计划随意改动范围
- 遇到阻塞不说明原因直接推进

**If blocked:**
- 立刻停下并向开发者说明阻塞点

## Integration

**Required workflow skills:**
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks

**Subagents should use:**
- 不使用 `superpowers:test-driven-development`（本仓库禁用 TDD），按计划编写与验证测试
- 涉及 Java 读侧开发时，遵循 `skills/modelView/SKILL.md` 规范

**Alternative workflow:**
- **superpowers:executing-plans** - Use for parallel session instead of same-session execution
