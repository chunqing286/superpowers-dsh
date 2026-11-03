# 安全审计员接入包（superpowers + DSH）

把"安全审计"作为**已有审查位的一个维度**注入 superpowers 流程，而不是新开一条平行流程。

本文是这套维度的**中文总览**：为什么这样设计、四个插入点、与 DSH `review` 工具的分工。
工作流实际遵守的规则在 [security-loop.md](security-loop.md)（英文，规范版本）。

原因：`subagent-driven-development` 已经有三个审查位（per-task review / scoped re-review / final
whole-branch review），它们之间靠固定的输出格式、fix loop、re-review 的 `ADDRESSED` 判定和
ledger 串起来。另起一个"安全审计员"角色会：与 task review 重复报同一批问题、findings 格式
对不上 re-review 模板、噪音翻倍把 fix loop 撑爆。

## 四个文件与四个插入点

（下文文件路径均相对于本技能目录 `skills/subagent-driven-development/`。）

| 文件 | 插入点 | 什么时候跑 |
|---|---|---|
| `references/threat-modeler-prompt.md` | writing-plans 写安全约束段之前（全新项目：brainstorming 之后） | 全新项目：产出安全约束 |
| `references/plan-security-section.md` | writing-plans 产出的 plan 里 | 已有 plan：补写安全约束段 |
| `references/security-review-part.md` | 派发时并入 `task-reviewer-prompt.md` | 每个任务审查（每次派发都并入） |
| `references/security-final-review.md` | 派发时并入终审 reviewer 的提示词 | 分支终审（每分支一次） |

```
全新项目：
  brainstorming → writing-plans → [threat-modeler] → (+安全约束段) → SDD(每任务安全审查) → final review → review(security lens) → merge

已有项目（已开工）：
  [threat-modeler 对现有 spec/plan 做一次] → 补写 plan 安全约束段 → 继续 SDD(从当前任务起注入)
```

## 安装

没有安装步骤——本 fork 已经把全部安全内容**内置在包里**。安装这个 npm 包
（`@chunqing286/superpowers-dsh`），就是全部安装；开发机上 clone 本仓库后本地 link 亦可
（**仅限本机开发**；部署到别的机器请用 npm 规格）。
不需要、也不存在任何打补丁脚本。

升级会**整体替换**已安装的插件目录，所以任何补丁都活不过一次升级，安全能力必须
活在包本身——这正是本 fork 存在的理由。曾经的 `install.ps1` + `pristine/` 备份 +
ASCII 锚点打补丁方案已废弃，由本 fork 取代；不要再对任何安装副本就地打补丁。

## 三条不可违反的规则

1. **输出格式一个字都不改**——唯一例外是报告多出一个 `### Security Coverage` 段，
   既有标题和分级一律不变。安全缺陷就是 Critical / Important / Minor 三级中的一种，不是新的一级。
   改格式 = re-review 判不了 = fix loop 断链。
2. **审计员只读。** 不许动工作树、index、HEAD、分支状态。要看别的版本就 `git worktree add` 到临时目录。
3. **按任务注入。** 安全约束是"这个任务必须做对的事"，不是"这个任务可能涉及的领域"。
   不给某个任务传安全约束，reviewer 就不会审那块的安全——这既是省钱的手段，也是防噪音的手段。

## 与 DSH `review` 工具的分工

| | SDD `references/security-review-part.md` | DSH `review` 工具 |
|---|---|---|
| 何时用 | SDD 循环**内**，每个任务 | SDD 循环**外**，合并前一次 |
| 假阳性控制 | 靠约束清单限定范围 | 靠 **verifier 逐条反驳** |
| 输出格式 | SDD 格式（fix loop 需要） | 自由格式 |

**不要把 `review` 工具塞进 SDD 循环内部**——它不产出符合 SDD 格式的 finding，会打乱 fix loop。

合并前那次（`review` + `lenses: ["security"]` + `depth: "full"`）看的是**整分支的组合效应**，
比如任务 A 放开了路径、任务 B 信任了那个路径——单任务视角看不出来。

## 可选加固

`dsh-experimental-auto-review`（默认关闭，需在插件页开启）会在**每次工具调用执行前**用 LLM 判一次
风险，medium（删既有对象、force push、生产操作、外部发送）需明确授权，high（跨信任边界泄露凭证）
始终拒绝。防的是审计员/实现者**越界执行危险操作**，不替代安全审计。
