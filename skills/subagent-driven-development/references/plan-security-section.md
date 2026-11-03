# plan 里的安全约束段（`## Security Constraints`）

写进 plan 文件，位置在 `## Global Constraints` 之后、`## Review Focus` 之前。

## 放在哪、为什么

`writing-plans` 的 plan 头部长这样：

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: ...

**Goal:** ...
**Architecture:** ...
**Tech Stack:** ...
**Spec:** ...

## Global Constraints
[...]

## Review Focus
[...]

---
```

安全约束插在 `## Global Constraints` 和 `## Review Focus` 之间。理由：

- `## Global Constraints` 是**项目级**要求（版本下限、命名、依赖限制）——安全约束也是项目级的，
  但它多了一个属性：**每条要有测试归属**，所以需要独立的段来承载"约束 → 哪个任务"的映射。
- `## Review Focus` 是**"spec 没说但会咬人的输入"**——安全威胁是它的一个子集，但两者粒度不同：
  Review Focus 写的是"用户输入空列表时应该怎样"，安全约束写的是"这个接口必须鉴权"。
  不要把安全约束塞进 Review Focus：Review Focus 的每一行要配一个测试钉到任务里，
  而安全约束里有一部分是**架构级的**（比如"整个系统不存储明文密码"），它不属于任何一个任务。

## 段模板

```markdown
## Security Constraints

These constraints bind every task that touches the components named in them.
Each line is a control the implementation must satisfy, with the attack path
it closes. A task that touches a named component MUST satisfy the constraints
naming that component; the task reviewer verifies this as part of its gate.

[SEC-01] `src/auth/verify.ts` — reject tokens whose `exp` is more than 60s in
         the past; compare with a constant-time function
         (attack path: expired token → unvalidated expiry → session hijack)
         test: `test_rejects_expired_beyond_grace` in `tests/auth/verify.test.ts`

[SEC-02] `src/api/users.ts` — every handler must resolve the caller's tenant
         from the verified session token, never from a request body or query
         parameter
         (attack path: client-supplied tenant id → missing tenant check →
         cross-tenant read)
         test: `test_rejects_body_tenant_override` in `tests/api/users.test.ts`

[SEC-03] `src/log/` — never write a credential, session token, or full card
         number to any log sink, including error messages returned to the
         client
         (attack path: credential in log → log reader exfiltrates secret)
         test: `test_no_secret_in_logs` in `tests/log/redact.test.ts`
```

每条四行，缺一不可：

| 元素 | 作用 | 不能省的原因 |
|---|---|---|
| **组件路径** | 决定"这条约束绑定哪些任务" | 不知道绑谁，reviewer 就会对不相关的任务做安全审查 |
| **精确约束** | 实现者照着能直接写代码 | "要安全"不可实现；"exp 超过 60s 即拒绝"可以 |
| **攻击路径** | reviewer 判断严重级的依据 | 没有路径就退化成模式匹配噪音 |
| **测试归属** | 让约束可执行、可回归 | 没有测试的约束，实现者会当成建议，实现完就忘了 |

## 怎么把约束分派到任务

在 plan 的每个任务里，**加一个 Interfaces 块之外的约束引用**。不要复制约束全文——复制会漂移
（约束改了，任务里的副本没改，两边对不上，reviewer 判错）。

**复制禁令只管 plan 文件本身，不管派发。** 禁止的是 plan 里出现第二份**持久**副本；派发提示词
是临时产物，在派发那一刻从 `## Security Constraints` 这份唯一权威定义现拷现用，评审派发本来就
是这么做的。而实现者的派发**必须**原样带上绑定它的约束全文：任务块里只有引用，task-brief 只抽
任务块（抽不到 plan 头部的约束段），实现者又不读整份 plan——不随派发给它，它就要为一条它从没
见过的约束负责。持久副本一份（plan 的约束段），临时副本按需现拷，不存在漂移面。

引用格式：

```markdown
### Task 3: Token verification

**Files:**
- Create: `src/auth/verify.ts`
- Test: `tests/auth/verify.test.ts`

**Security constraints binding this task:**
- SEC-01 (see `## Security Constraints`) — implement and cover with
  `test_rejects_expired_beyond_grace`
```

一行引用 + 一句"实现它并用这个测试覆盖"。约束的**权威定义只在 `## Security Constraints` 那一处**。

**归属规则**：约束里的组件路径出现在这个任务的 `Files:` 里 → 这条约束绑定这个任务。
一个组件被多个任务改 → 多个任务都引用它。

## 与 writing-plans 自审的接线

`writing-plans` 的自审清单第 5 条就是安全约束检查（原第 5 条 Proportion 顺延为第 6 条）：

> **5. Security constraints:** Every constraint has a task that owns its
> component and a test in that task that exercises it; a trust boundary no
> constraint covers is a gap in the threat model, not in the plan.

第 5 条的后半句——"信任边界没有约束覆盖"——很重要：它抓的是**威胁模型本身的漏项**，
不是计划的漏项。

## 已有 plan 怎么补

1. 对现有 spec + plan 派一次 `threat-modeler-prompt.md`（只读子代理）
2. 读它的 `### Open Questions for the Human`，逐条裁决——**安全决策是你的**
3. 把 `### Security Constraints` 的 copy-pasteable 列表贴进 plan
4. 逐条找到拥有该组件的任务，加引用行
5. 找到该任务对应的测试步骤，插一个"写失败测试 → 跑失败 → 实现 → 跑通过"的步骤

第 5 步别省。没有测试步骤的约束，实现者写不写全凭自觉。

## 已有代码的项目：先补威胁模型，再补 plan

对已开工的项目，顺序是：

```
[threat-modeler 对现有 spec + 现有代码]  →  补 plan 的安全约束段  →  给后续任务加引用
                                                      ↑
                                     已完成的任务不回溯，除非约束指向的组件在那些任务里被改过
```

**不回溯是默认**，但有一个例外：**如果某条约束指向的组件在已完成的任务里被改过**，
那它现在就是一个已知的、没被修的缺陷。按 `receiving-code-review` 的纪律处理——
先核实，再决定是立刻修还是记进 ledger 留到 final review 裁决。
