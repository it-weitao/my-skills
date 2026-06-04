---
name: plan-to-todo-spec
description: |
  基于代码上下文输出高质量 implementation plan，并进一步拆分为适合弱模型逐条执行的 todo spec。凡是用户提到 plan、拆任务、todo、checklist、implementation plan、执行计划、分步落地、弱模型协作、cc-generate-then-review，或希望先规划再逐条实施时，都应主动使用此 skill。尤其当任务跨模块、涉及依赖顺序、需要控制改动范围、需要后续恢复上下文时，必须使用此 skill。
---

# Plan To Todo Spec

这个 skill 用来把“一个较大的实现任务”拆成两层产物：

1. 强模型持有的 **implementation plan**
2. 强模型后续可直接消费的 **todo spec 列表**

这里的 todo 不是普通待办事项，而是给强模型后续生成弱模型 prompt 的中间规格。每条 todo 都必须足够清晰，便于后续配合 `cc-generate-then-review` 将单条 todo 转成高约束执行 prompt，再交给较弱模型逐条实现。

## 适用场景

在以下情况优先使用本 skill：

- 用户要求先出 plan 再实施
- 用户要求拆分 todo、checklist、task list
- 用户明确提到 `cc-generate-then-review`
- 用户要把复杂任务拆给较弱模型逐条完成
- 任务跨多个模块、层次或边界
- 任务需要长时间推进，且希望中途可恢复上下文
- 任务需要控制每次变更的文件范围和验证方式

如果任务非常小、只改一个点、且不需要进一步拆分，则不必强行生成很长的 todo spec。

## 核心目标

始终同时完成这几件事：

1. 产出一个决策完整的 plan
2. 基于模块边界和依赖顺序拆出 todo spec
3. 让每条 todo 都适合后续转交给弱模型执行
4. 让 plan 和 todo 在上下文丢失后仍可恢复
5. 降低弱模型误改无关文件、越权修改、一次改太多的风险

## 设计原则

### 1. 先按架构拆，再按执行拆

不要只按“想到什么做什么”列 todo。先识别：

- 领域层 / 应用层 / 基础设施层 / UI 层
- 公共类型与接口
- 上下游依赖关系
- 哪些变更必须先做，哪些可以并行
- 哪些文件必须一起改，哪些应拆开改

todo 必须建立在真实模块边界上，而不是拍脑袋平均切块。

### 2. Todo 粒度使用“文件组级”

默认每条 todo 应覆盖 `1 到 3 个强相关文件`，或同一模块内的一组紧密耦合修改。

优先这样拆：

- 一个应用层变更一条 todo
- 一个驱动适配变更一条 todo
- 一个公共类型定义加其直接消费者一条 todo
- 一个测试补全变更一条 todo

避免这样拆：

- 一条 todo 横跨多个不相关模块
- 一条 todo 同时改 application、infrastructure、test、docs 且没有强耦合
- 一条 todo 大到弱模型仍需自己继续做架构判断
- 一条 todo 小到只改一行但没有独立意义

### 3. Todo 必须服务于弱模型执行

每条 todo 都要假设后续执行者较弱，因此必须尽量消除它的自由裁量空间。要提前说明：

- 改哪些文件
- 只允许改到什么范围
- 哪些内容禁止改
- 前置依赖是什么
- 做完如何验证
- 预期结果是什么

### 4. 优先降低风险，而不是追求条目少

如果把多个改动塞进一条 todo 会提高误改概率，就拆开。宁可 todo 多一些，也不要让单条 todo 过于含混。

### 5. 保证可恢复

如果任务较大，输出必须能在上下文被清空后恢复。恢复信息至少应包含：

- 当前目标
- 总体 plan 摘要
- 模块划分
- 每条 todo 的状态
- 依赖关系
- 下一条建议执行项
- 恢复执行指令

## 工作流程

### 第一步：先理解代码和边界

在拆 plan 前，优先通过非破坏性方式理解：

- 相关入口文件
- 类型定义
- 服务/用例
- 驱动与适配器
- 当前测试覆盖
- 现有模块组织方式

如果某些关键事实可以从代码中确认，不要问用户。

只有在这些问题无法从环境中得到答案时才提问：

- 目标行为存在多种合理解释
- 架构方向存在多种高影响选择
- 用户对范围有强约束但未说明
- 是否需要持久化 plan/todo 状态

### 第二步：先出 implementation plan

plan 必须是“决策完整”的，而不是笼统建议。至少覆盖：

- 当前状态与目标状态
- 模块边界调整
- 关键接口或类型变化
- 依赖顺序
- 风险点
- 验证策略

如果处于 Plan mode，最终 plan 必须放在 `<proposed_plan>` 块内。

### 第三步：再从 plan 派生 todo spec

todo 不是 plan 的简单复述，而是可执行拆分。拆分时按以下顺序思考：

1. 哪些前置定义要先落
2. 哪些调用方会因此受影响
3. 哪些适配层需要分别跟进
4. 哪些测试应跟随哪条 todo 一起做
5. 哪些事项可延后到最后集中验证

### 第四步：如有需要，生成恢复文件内容

如果用户要求持久化，或任务明显很大，生成一个可直接保存的恢复文件内容。

默认路径建议为：

` .codex/plans/<date>-<topic>.md `

注意：

- 在 Plan mode 下，不要真的写文件
- 在非 Plan mode 下，只有用户明确要求落盘时才写入
- 若用户未要求真正写文件，则只输出文件内容

## 输出要求

最终输出必须同时包含两部分：

1. `implementation plan`
2. `todo spec list`

当任务较大或用户明确要求持久化时，再额外包含：

3. `recovery file` 内容

## Plan 输出结构

如果处于 Plan mode，使用这个结构：

```md
<proposed_plan>
# <标题>

## Summary
- ...

## Implementation Changes
- ...

## Public Interfaces / Types
- ...

## Test Plan
- ...

## Assumptions
- ...
</proposed_plan>
如果不在 Plan mode，也尽量保持相同结构，但不强制使用标签。

Todo Spec 输出结构
ALWAYS 使用如下格式输出 todo 列表。不要自行删字段。

## Todo Specs

### TODO-01 <标题>
- Status: `pending`
- Module: `<模块名>`
- Goal: `<这一条要完成的目标>`
- Target Files:
  - `<file-a>`
  - `<file-b>`
- Allowed Scope:
  - `<允许修改的具体范围>`
- Out Of Scope:
  - `<禁止修改的内容>`
- Dependencies:
  - `<依赖的前置 todo；若无则写 none>`
- Implementation Notes:
  - `<强模型给弱模型的关键实现提示，只写必要事实，不写泛泛建议>`
- Validation Commands:
  - `<command-1>`
  - `<command-2>`
- Expected Result:
  - `<完成后可观察到的结果>`
- Weak-Model Prompt Readiness:
  - `<说明这条 todo 是否已足够直接喂给 cc-generate-then-review；默认写 ready 或 explain why not>`

### TODO-02 <标题>
- Status: `pending`
- Module: `<模块名>`
- Goal: `<...>`
- Target Files:
  - `<...>`
- Allowed Scope:
  - `<...>`
- Out Of Scope:
  - `<...>`
- Dependencies:
  - `<...>`
- Implementation Notes:
  - `<...>`
- Validation Commands:
  - `<...>`
- Expected Result:
  - `<...>`
- Weak-Model Prompt Readiness:
  - `<...>`
字段填写规则
Status
只允许以下值：

pending
in_progress
done
blocked
初始生成时默认使用 pending。如果用户提供了已有进度，再按事实填写。

Module
填写真实模块名或层名，不要写模糊词，例如：

application/events
application/services
infrastructure/drivers/agora
infrastructure/drivers/tencent
shared types
避免使用：

misc
other
refactor
common changes
Goal
Goal 必须描述完成态，不要写成动作口号。

好例子：

让 application 层只依赖统一事件抽象，不再感知具体 driver 事件形状
坏例子：

重构一下事件
优化代码
处理兼容性
Target Files
必须尽可能具体。优先列出确定文件，而不是目录。

如果当前只能确定大致位置，要明确说明“待强模型二次确认”，不要伪装成已经确定。

Allowed Scope
这里写“这条 todo 允许动什么”，例如：

只允许新增/调整 whiteboard 事件领域类型及其在 service 中的消费方式
只允许修改 agora driver 的事件映射逻辑，不改 room service 对外接口
Out Of Scope
这里写“明确禁止动什么”，例如：

不修改导出 API
不修改无关 driver
不改字符串文案
不顺手修复其他 lint 问题
不变更测试框架配置
Dependencies
必须写前置依赖。没有就写 none。

如果一条 todo 依赖另一条，就必须显式写出 todo id，而不是笼统写“前面的任务”。

Implementation Notes
只保留后续弱模型真正需要的实现事实，例如：

哪个类型应成为单一事实来源
哪个 service 应保持接口不变
哪个 driver 只做适配，不应上浮业务逻辑
哪个字段命名必须与现有约定一致
不要写空话，例如：

注意代码质量
做最小修改
保持架构清晰
这些属于执行 prompt 的通用约束，不属于实现事实。

Validation Commands
至少给出最小必要校验。优先精确到受影响包或测试范围，避免默认全仓大校验，除非确实需要。

Expected Result
必须写成可验收结果，而不是抽象愿景。

好例子：

service 侧改为只消费统一事件结构，agora 和 tencent driver 均能编译通过
坏例子：

架构更优雅
可维护性更高
Weak-Model Prompt Readiness
只有当以下信息都足够清楚时才写 ready：

文件范围清晰
修改边界清晰
前置依赖清晰
验证命令清晰
否则写出原因，例如：

not ready: target files still need confirmation from current event export graph
恢复文件输出结构
当任务较大或用户要求持久化时，额外输出一个恢复文件内容，使用如下模板：

# <任务标题>

## Goal
- <总体目标>

## Current Status
- Overall Status: `planning|in_progress|blocked|done`
- Recommended Next Todo: `TODO-XX`

## Context Summary
- <当前代码状态摘要>
- <关键模块边界>
- <当前约束或风险>

## Implementation Plan
- <这里放 plan 的摘要版>

## Todo Specs
- <完整 todo 列表，沿用上面的固定格式>

## Execution Rules
- 后续执行时一次只处理一条 todo
- 先确认依赖项已完成，再生成弱模型 prompt
- 使用 `cc-generate-then-review` 基于单条 todo 生成执行 prompt
- 若执行结果偏离 Allowed Scope，必须回退并重生成 prompt
- 不得跨 todo 混改

## Recovery Instructions
1. 先阅读本文件，不要先重拆计划。
2. 根据 `Recommended Next Todo` 选择下一条未完成 todo。
3. 检查该 todo 的 `Dependencies` 是否全部完成。
4. 将该 todo 交给强模型，使用 `cc-generate-then-review` 生成弱模型执行 prompt。
5. 弱模型执行后，再按当前仓库状态更新本文件中的状态。
与 cc-generate-then-review 的协作规则
本 skill 的 todo spec 必须天然兼容 cc-generate-then-review。

这意味着每条 todo 至少要能映射到以下槽位：

GOAL ← Goal
TARGET_FILES ← Target Files
ALLOWED_SCOPE ← Allowed Scope
VALIDATION_COMMAND_* ← Validation Commands
同时，Out Of Scope 和 Implementation Notes 应为强模型后续补充 prompt 约束提供材料。

如果某条 todo 还不能稳定映射到这些槽位，就不能标记为 ready。

质量门槛
生成 todo spec 时，主动做这些检查：

是否有 todo 横跨不相关模块
是否有 todo 缺少清晰文件范围
是否有 todo 依赖顺序不明确
是否有 todo 无法独立验证
是否有 todo 会迫使弱模型自行做架构决策
是否有 todo 只是把 plan 改写了一遍，没有真正可执行化
若存在上述问题，先重拆，再输出。

风险控制
优先避免以下错误：

把公共类型变更和多个下游适配一起塞进一条 todo
把“实现逻辑 + 测试补全 + 文档整理”无脑绑定为一条 todo
没有明确禁止范围，导致弱模型顺手改无关代码
没有显式依赖，导致弱模型按错误顺序执行
验证命令过粗，导致每条 todo 成本过高
恢复文件只保存 checklist，没有上下文摘要
默认策略
当用户没有明确指定时，使用以下默认值：

todo 粒度：文件组级
每条 todo 目标文件数：优先 1 到 3 个
输出内容：plan + todo spec
大任务额外输出：recovery file
recovery 路径建议：.codex/plans/<date>-<topic>.md
持久化行为：
Plan mode：只输出文件内容，不写文件
非 Plan mode：只有用户明确要求时才写文件
状态初始值：pending
简短自检
输出前快速检查：

plan 是否已经决策完整
todo 是否按模块和依赖顺序拆分
每条 todo 是否适合后续生成弱模型 prompt
是否给出了明确文件范围和禁止范围
是否具备恢复上下文所需信息
如果任一项答案是否定，先修正再输出。
