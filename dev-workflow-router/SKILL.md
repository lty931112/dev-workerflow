---
name: dev-workflow-router
description: 开发任务总控路由技能，自动判定应调用 workflow-skill、doc-generator 或两者联动。用于“实现需求、修 bug、重构、补文档、文档缺失、代码后文档同步、需求追踪”等混合场景；当请求同时包含代码变更与文档更新意图时优先触发。
---

# Dev Workflow Router（开发流程总控）

## 目的

在开发类对话中作为入口技能，先判断任务类型，再路由到合适的技能：
- `workflow-skill`：代码实现流程管控
- `doc-generator`：文档初始化/同步/需求记录

## 路由规则

1. **仅文档类请求**（创建/补全文档、更新接口文档、记录需求）  
   → 直接调用 `doc-generator`

2. **涉及代码变更**（新增、修改、重构、修复、测试）  
   → 先调用 `workflow-skill`

3. **代码变更且文档缺失**  
   → `workflow-skill` 阶段零发现缺失后，切换到 `doc-generator` 初始化，再返回 `workflow-skill`

4. **代码变更完成后需要同步文档**  
   → `workflow-skill` 阶段四结束后调用 `doc-generator` 阶段二/三

## 关键词判定（优先级从高到低）

1. **联动关键词**：`实现并更新文档`、`改代码并同步文档`、`需求+文档`、`开发闭环`  
   → 直接走“联动”
2. **代码关键词**：`修复`、`新增功能`、`重构`、`优化`、`测试`、`改配置`、`改逻辑`  
   → 先走 `workflow-skill`
3. **文档关键词**：`补文档`、`更新 API 文档`、`architecture`、`conventions`、`changelog`、`requirements`  
   → 走 `doc-generator`
4. **无法判定**：先输出澄清问题，再选择技能链

## 执行顺序

1. 判断是否有代码变更。
2. 判断文档是否完整。
3. 根据路由规则执行技能链。
4. 输出当前处于哪个技能阶段、下一步动作与等待项。

## 输出模板

```text
🧭 路由结果
- 场景判定：{代码变更 / 文档维护 / 联动}
- 已调用技能：{workflow-skill / doc-generator / 两者}
- 当前阶段：{阶段名}
- 下一步：{具体动作}
- 等待用户：{是/否，原因}
```

## 联动示例（最小可复用）

```text
用户请求：实现用户导出功能并补充相关文档
路由判定：联动
执行链路：
1) workflow-skill：阶段零~三完成代码与测试
2) workflow-skill：阶段四完成审查
3) doc-generator：阶段二同步 architecture/api/development-guide
4) doc-generator：阶段三记录 requirements/changelog
```

## 约束

- 不得跳过 `workflow-skill` 的任务确认门禁。
- 不得在文档缺失且未确认策略时直接进入代码实现。
- 文档更新必须走 `doc-generator` 的增量更新规则。
