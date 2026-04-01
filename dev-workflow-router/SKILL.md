---
name: dev-workflow-router
description: 开发任务总控路由技能，自动判定应调用 workflow-skill、doc-generator 或两者联动。用于"实现需求、修 bug、重构、补文档、文档缺失、代码后文档同步、需求追踪"等混合场景；当请求同时包含代码变更与文档更新意图时优先触发。

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

## 会话状态感知

### 状态追踪

Router 必须在每个输出中携带当前状态标记，并在下一轮输入时优先检查：

1. **首轮判定**：按原有关键词规则判定场景
2. **后续轮次判定**（优先级从高到低）：
   - **延续关键词**：`继续`、`下一步`、`然后`、`接着`、`好的`、`可以`、`确认`、`开始执行`、`改一下`、`还有`、`另外`、`再加上`、`没问题`、`行`、`嗯`
     → 读取上一轮状态，继续当前技能的当前阶段
   - **修正关键词**：`不对`、`不是这样`、`改一下需求`、`我重新说`、`换个方案`、`等等`、`先别`、`暂停`
     → 回退到 workflow-skill 阶段二（任务清单重新生成），或暂停当前阶段等待用户澄清
   - **完成关键词**：`好了`、`没问题了`、`就这样`、`结束`、`完成`、`可以了`
     → 如果在 workflow-skill 阶段三/四，触发阶段四审查闭环
   - **新任务关键词**：包含明确的代码/文档关键词（同原有规则）
     → 如果当前有未完成的技能流程，先询问：是继续当前任务还是开始新任务
   - **仍无法判定**：
     → 检查上一轮是否处于等待用户状态（`waiting: true`），如果是则视为对上一轮的回应
     → 否则输出澄清问题

### 状态标记格式

每轮输出末尾必须附加状态标记，供下一轮判定使用：

```text
<!-- WORKFLOW_STATE: {skill}:{stage}:{status}:{waiting} -->
```

**状态值说明**：

| 字段    | 可选值                                               | 含义               |
| ------- | ---------------------------------------------------- | ------------------ |
| skill   | `workflow-skill` / `doc-generator` / `both` / `idle` | 当前活跃技能       |
| stage   | `阶段零` ~ `阶段四` / `none`                         | 当前阶段           |
| status  | `in_progress` / `completed` / `blocked` / `none`     | 阶段状态           |
| waiting | `true` / `false`                                     | 是否在等待用户输入 |

**示例**：

```text
<!-- WORKFLOW_STATE: workflow-skill:阶段三:in_progress:false -->
<!-- WORKFLOW_STATE: doc-generator:阶段二:completed:false -->
<!-- WORKFLOW_STATE: idle:none:none:false -->
<!-- WORKFLOW_STATE: workflow-skill:阶段二:in_progress:true -->
```

### 上下文传递规则

- 如果上一轮状态不是 `idle`，本轮输入应优先视为对上一轮的延续
- 仅当用户明确表达"新任务"或出现新的强关键词时，才重置状态为 `idle` 后重新判定
- 跨技能切换时（如 workflow → doc-generator），状态应级联传递，记录来源技能以便返回
- 当 `waiting: true` 时，任何非空输入都应视为对等待项的回应

### 多轮对话路由流程

```text
用户输入
  │
  ├─ 首轮（无 WORKFLOW_STATE 或 state = idle）
  │    └─ 按关键词规则判定场景 → 初始化状态 → 执行对应技能
  │
  └─ 后续轮次（存在 WORKFLOW_STATE 且 state ≠ idle）
       ├─ 匹配延续关键词 → 继续当前技能当前阶段
       ├─ 匹配修正关键词 → 回退或暂停
       ├─ 匹配完成关键词 → 触发审查闭环
       ├─ 匹配新任务关键词 → 询问用户意图
       ├─ waiting = true → 视为对等待项的回应
       └─ 无法判定 → 输出澄清问题
```

## 关键词判定（优先级从高到低）

1. **联动关键词**：`实现并更新文档`、`改代码并同步文档`、`需求+文档`、`开发闭环`
   → 直接走"联动"
2. **代码关键词**：`修复`、`新增功能`、`重构`、`优化`、`测试`、`改配置`、`改逻辑`、`实现`、`开发`、`编码`、`写代码`、`改代码`、`加功能`、`修 bug`、`修问题`、`改接口`、`补测试`
   → 先走 `workflow-skill`
3. **文档关键词**：`补文档`、`更新 API 文档`、`architecture`、`conventions`、`changelog`、`requirements`、`写文档`、`更新文档`、`同步文档`、`维护文档`
   → 走 `doc-generator`
4. **无法判定**：先输出澄清问题，再选择技能链

## 执行顺序

1. 检查是否存在上一轮的 WORKFLOW_STATE。
2. 如果存在且非 idle，按会话状态感知规则处理。
3. 如果不存在或为 idle，按关键词规则判定场景。
4. 判断是否有代码变更。
5. 判断文档是否完整。
6. 根据路由规则执行技能链。
7. 输出当前处于哪个技能阶段、下一步动作与等待项。
8. 在输出末尾附加 WORKFLOW_STATE 状态标记。

## 输出模板

```text
🧭 路由结果
- 场景判定：{代码变更 / 文档维护 / 联动 / 延续上一轮}
- 已调用技能：{workflow-skill / doc-generator / 两者}
- 当前阶段：{阶段名}
- 下一步：{具体动作}
- 等待用户：{是/否，原因}

<!-- WORKFLOW_STATE: {skill}:{stage}:{status}:{waiting} -->
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

## 多轮对话示例

```text
# 第1轮
用户：帮我实现用户导出功能
路由判定：代码变更
→ 调用 workflow-skill，阶段零~二
输出末尾：<!-- WORKFLOW_STATE: workflow-skill:阶段二:in_progress:true -->

# 第2轮
用户：好的，开始吧
路由判定：延续（匹配"好的"）
→ 继续 workflow-skill 阶段三执行
输出末尾：<!-- WORKFLOW_STATE: workflow-skill:阶段三:in_progress:false -->

# 第3轮
用户：还有个问题，导出的格式需要支持 CSV
路由判定：延续+修正（匹配"还有"+"需要"）
→ 暂停阶段三，回到阶段二追加任务
输出末尾：<!-- WORKFLOW_STATE: workflow-skill:阶段二:in_progress:true -->

# 第4轮
用户：确认，继续
路由判定：延续（匹配"确认"+"继续"）
→ 继续 workflow-skill 阶段三
输出末尾：<!-- WORKFLOW_STATE: workflow-skill:阶段三:in_progress:false -->

# 第5轮
用户：好了
路由判定：完成（匹配"好了"）
→ 触发 workflow-skill 阶段四审查闭环
输出末尾：<!-- WORKFLOW_STATE: workflow-skill:阶段四:in_progress:false -->
```

## 约束

- 不得跳过 `workflow-skill` 的任务确认门禁。
- 不得在文档缺失且未确认策略时直接进入代码实现。
- 文档更新必须走 `doc-generator` 的增量更新规则。
- 每轮输出必须包含 WORKFLOW_STATE 状态标记。
- 后续轮次必须优先检查状态标记，不得忽略上下文直接重新判定。

---

## 版本信息

- **版本**：1.2.0
- **更新日期**：2026-04-01
- **变更记录**：新增会话状态感知机制，支持多轮对话延续触发；新增延续/修正/完成关键词识别；新增 WORKFLOW_STATE 状态标记协议
- **适用范围**：所有开发类对话，包括多轮对话场景