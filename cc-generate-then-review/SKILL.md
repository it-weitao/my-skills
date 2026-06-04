---
name: cc-generate-then-review
description: 输出给 Claude Code 手动执行的高约束 Prompt，并在用户执行完成后对代码变更进行审查。适用于“让 Claude Code 直接改文件，Codex 仅负责审查与验收”的协作流程，尤其适合较弱模型场景。
---

# CC Generate Then Review

## Workflow

1. 确认目标
- 明确目标文件路径。
- 明确修改范围（例如：只允许注释与空行）。

2. 仅输出一段给 Claude Code 的 Prompt
- 不替用户执行 Claude Code。
- Prompt 必须包含：目标、范围、硬约束、执行步骤、自检命令、固定输出格式。
- Prompt 必须作为一个整体输出，放在单个代码块或单个纯文本块中，方便用户一次性复制。
- 输出 Prompt 时，禁止在代码块或文本块外添加解释、前言、总结或额外说明。

3. 等待用户贴回执行结果
- 期望用户返回结构化哨兵行（成功/失败统一格式）：
`STATUS: SUCCESS|FAIL`
`FILES: <comma-separated paths>`
`ESLINT: PASS|FAIL|SKIP`
`TYPECHECK: PASS|FAIL|SKIP`
`TEST: PASS|FAIL|SKIP`
`ERROR_STAGE: NONE|ANALYZE|EDIT|VALIDATE`
`ERROR_CMD: <failed command or NONE>`
`ERROR_SUMMARY: <one-line summary or NONE>`
- 当 `STATUS: FAIL` 时，`ERROR_STAGE`、`ERROR_CMD`、`ERROR_SUMMARY` 必须为非 `NONE`。

4. 审查与验收
- 运行 `git diff -- <target files>`。
- 对受影响文件执行最小必要校验（eslint/tsc/测试）。
- 按严重级别输出 review 结论（阻断 -> 高 -> 中 -> 低）。

## Weak-Model Prompt Rules

1. 使用短句和编号步骤，不使用含糊表达。
2. 用“必须/禁止”而不是“建议/尽量”。
3. 明确“只改哪些文件，禁止改哪些内容”。
4. 指定“先做什么，再做什么”，减少模型跳步。
5. 指定失败处理：校验失败必须继续修复直到通过；若最终仍失败，必须输出结构化错误信息。
6. 指定固定收尾格式，便于机器和人工验收。

## Prompt Template (Weak Model)

将下列模板中的占位符替换后输出给用户：

你现在在仓库 `<REPO_PATH>` 中工作。

请直接编辑以下文件（仅这些）：
`<TARGET_FILES>`

任务目标：
`<GOAL>`

修改范围与硬约束（必须全部满足）：
1) 只允许修改：`<ALLOWED_SCOPE>`。
2) 严禁修改：导出接口、函数签名、常量值、字符串字面量、逻辑分支、事件绑定、import 顺序（除非明确允许）。
3) 严禁修改无关文件。
4) 若发现当前文件存在与本任务无关的问题，不要顺手修复，保持最小变更。

执行步骤（必须按顺序）：
1) 先阅读目标文件并理解上下文。
2) 仅按任务目标做最小修改。
3) 执行校验命令：
   - `<VALIDATION_COMMAND_1>`
   - `<VALIDATION_COMMAND_2>`
4) 若任一命令失败，继续修复并重复执行，直到全部通过。

输出要求（最终仅输出以下键值行，不要附加解释）：
STATUS: SUCCESS|FAIL
FILES: `<TARGET_FILES>`
ESLINT: PASS|FAIL|SKIP
TYPECHECK: PASS|FAIL|SKIP
TEST: PASS|FAIL|SKIP
ERROR_STAGE: NONE|ANALYZE|EDIT|VALIDATE
ERROR_CMD: <failed command or NONE>
ERROR_SUMMARY: <one-line summary or NONE>

失败输出约束：
- 若 `STATUS: FAIL`，则 `ERROR_STAGE`、`ERROR_CMD`、`ERROR_SUMMARY` 不得为 `NONE`。
- 若 `STATUS: SUCCESS`，则 `ERROR_STAGE: NONE`、`ERROR_CMD: NONE`、`ERROR_SUMMARY: NONE`。

## Review Output Format

1. Findings
- 按严重级别列出问题，每条包含文件路径和行号。

2. Open Questions / Assumptions
- 仅在确实存在不确定性时给出。

3. Summary
- 若无问题，明确写：未发现功能性问题。
- 补充剩余风险或测试缺口（如有）。
