# Skills 集合说明

本目录存放可复用的 Cursor Agent Skills，用于规范化开发流程与文档维护。

## 目录结构

```text
skills/
├── dev-workflow-router/
│   └── SKILL.md
├── workflow-skill/
│   └── SKILL.md
└── doc-generator/
    └── SKILL.md
```

## Skill 职责

- `dev-workflow-router`：开发任务总控路由，负责识别请求类型并分发到合适技能（代码流程 / 文档流程 / 联动流程）。
- `workflow-skill`：代码任务标准流程，覆盖文档检查、任务确认、实现、验证与审查闭环。
- `doc-generator`：项目文档生成与增量同步，维护架构、规范、API、开发指南与需求变更记录。

## 推荐使用方式

1. 用户请求到来后，先由 `dev-workflow-router` 判定场景。
2. 仅代码变更：进入 `workflow-skill`。
3. 仅文档维护：进入 `doc-generator`。
4. 代码 + 文档同步需求：先执行 `workflow-skill`，在合适阶段联动 `doc-generator`。

## 适用场景示例

- 修复 Bug、新增功能、重构、补测试
- 补文档、更新 API 文档、维护需求记录与变更日志
- 要求“实现并同步文档”的开发闭环任务

## 维护建议

- 新增 skill 时，保持目录名与 `SKILL.md` 中 `name` 一致。
- 更新技能规则时，优先增量修改，避免破坏既有工作流约定。
- 对外说明能力范围时，以各 `SKILL.md` 的 `description` 为准。

