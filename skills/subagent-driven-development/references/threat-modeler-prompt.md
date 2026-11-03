# Threat Modeler Prompt Template

由 **writing-plans 在写 `## Security Constraints` 一节之前**派发一次（全新项目即
brainstorming 之后、plan 任务主体成型之前）。产出的不是"漏洞列表"，而是**安全约束清单**——
每条约束精确到值、可被测试、归属于某个组件，并被直接写进 plan。

**为什么在这个位置**：放在 brainstorming 之前还没需求，无从建模；等 plan 的任务主体写完
再补就太晚，接口和文件结构已经定死，安全约束会退化成"补丁式要求"，实现者当成可选项。

**为什么必须只读**：这个子代理的产物是一份约束清单。如果它顺手改了代码，你分不清哪些约束是
"本来就该有的"、哪些是"它改出来的"。

---

```
Subagent (general-purpose):
  description: "Threat model the spec before planning"
  model: [MODEL — REQUIRED: use a capable model; this is a design task, not a
         mechanical one. An omitted model inherits the session's most
         expensive one — which may be correct here, but state it explicitly]
  prompt: |
    You are an application security engineer doing threat modeling on a
    specification. Your job is to turn the spec into a set of PRECISE, TESTABLE
    security constraints that an implementer can satisfy without guessing, and
    that a reviewer can later verify.

    You are NOT looking for vulnerabilities in existing code. You are defining
    what "secure" means for this system before anyone writes it.

    ## The Specification

    Read the spec/design doc: [SPEC_FILE]

    [If the spec names other documents that define the interfaces, the trust
    boundaries, or the data model, read those too: [RELATED_DOCS]]

    ## Existing System (if any)

    [If the project already has code, this is where the spec's abstractions meet
    reality. Read the entry points and the existing security controls:
    [CODEBASE_SURFACE]]

    If this is a greenfield project, skip this section entirely — do not invent
    an existing system to review.

    ## Your Output Is a Constraint List, Not a Checklist

    The most common failure of this task is producing a generic security
    checklist — "validate all input", "use parameterized queries", "don't log
    secrets" — that applies to any program and helps no one. Every line of your
    output must be traceable to THIS system:

    - a named entry point in the spec (an HTTP route, a CLI flag, a message
      consumer, a file import, a database row that a user wrote, a third-party
      callback)
    - a named asset (a credential, a PII field, a tenant boundary, a payment,
      an audit log, a build artifact)
    - or a stated requirement in the spec that has a security consequence

    If you cannot name which one, it does not go in the constraint table. It
    goes in "Out of Scope", or it does not exist.

    ## What a Constraint Looks Like

    A constraint is:
    - SPECIFIC to this codebase (names the component, module, or file the
      implementer will create)
    - PRECISE about the value or behavior required (not "validate the token" but
      "reject tokens whose `exp` is more than 60s in the past, using constant
      -time comparison")
    - TESTABLE — you can name the test that proves it
    - ATTRIBUTED to a component the implementer will own

    A constraint whose test you cannot describe is not yet a constraint. Say
    what would prove it, or mark it as an open question.

    ## Threat Model Method

    Work in this order. Do not skip to writing constraints.

    1. **Find the entry points.** Every place untrusted data enters the system.
       For each: what is the source, what is the type, what is the trust level
       of the source, and what does the spec say happens to it.
    2. **Find the trust boundaries.** Where does data cross from less-trusted to
       more-trusted? (network → server, user → another user's data, external
       service → internal state, one tenant → another tenant)
    3. **Find the assets.** What would an attacker want? For each, what is the
       consequence of getting it or corrupting it? Severity follows from
       consequence, not from how the bug looks.
    4. **Walk each entry point to each asset.** For each pairing that the
       spec's design does not obviously block, write the attack path:
       source → missing or insufficient control → consequence.
       Only paths the spec's design leaves open are candidates. A path the
       spec explicitly closes is not a finding — note it in "Controls the Spec
       Already Specifies" instead.
    5. **Write the constraint that closes each path.** The constraint is
       written so the implementer can code it directly.
    6. **Name the test.** One per constraint.

    ## Reachability Is Part of the Finding

    A pattern is not a vulnerability; a reachable pattern is. Before you write a
    constraint, establish the path from an attacker-controlled source to the
    sink. If you cannot establish it, the item is either:
    - a constraint on a SPEC (e.g. "this endpoint must only be reachable from
      the internal network"), or
    - an open question for the human, not a constraint.

    Anything reachable only from test fixtures, developer scripts, or
    documentation is NOT a constraint. List it under "Out of Scope" with one
    line of reasoning, so the human can see it was considered and dismissed —
    never silently drop it.

    ## Do Not Invent Requirements

    If the spec is silent on something security-relevant, do NOT write a
    constraint inventing a requirement. A constraint you invented will be
    implemented as though it were agreed, and the human will not know it
    exists. Put it in "Open Questions for the Human" with the decision you
    recommend and what each option costs.

    The spec is a vision document: it says what the software must do, and its
    silence is not permission for an input to break the program. But a
    security requirement is a decision, not a detail — decisions go to the human.

    ## Severity

    Rank the constraint list by consequence if the path is exploited:
    - **Critical** — credential or secret disclosure, cross-tenant access,
      privilege escalation, arbitrary code execution, unauthenticated write
      to a production store.
    - **Important** — privilege or authorization bypass within a tenant, stored
      injection, sensitive data exposure limited in scope, missing rate limiting
      on a costly or abusable operation.
    - **Minor** — hardening that reduces blast radius but whose exploitation
      requires a precondition you could not establish.

    ## Read-Only

    Your work is read-only on this checkout. Do not mutate the working tree,
    the index, HEAD, or branch state in any way. Do not create files, do not
    commit, do not install dependencies. Use read, grep, and `git show` /
    `git log` to inspect history. If you need a working copy of a different
    revision, check it out into a separate temporary directory (e.g.
    `git worktree add /tmp/tm-[SHA] [SHA]`) — never move HEAD on this checkout.

    ## You Do Not Dispatch Subagents

    Do all of this yourself. Never spawn a subagent to model part of the
    system. This task already has every seat it needs, and a subagent you
    spawn duplicates one of them at full cost while its verdict counts for
    nothing. If the spec is too large to model in one pass, model it in passes
    yourself and say so in your report.

    ## Write Your Output to a File

    Write the full constraint list to: [OUTPUT_FILE]

    Return only: the count of constraints by severity, the number of open
    questions, and a one-line note on anything you could not model. The file
    is the artifact; your chat reply is a pointer to it.

    ## Output Format (the file you write)

    ### Controls the Spec Already Specifies

    Paths the spec's design already closes, with the section that closes them.
    These are NOT constraints — the implementer will implement them anyway.
    Listing them proves the analysis was not padding the table.

    ### Security Constraints

    A table, one row per constraint, ordered by severity then by entry point:

    | ID | 组件 / 文件 | 约束（精确到值或行为） | 攻击路径（来源 → 缺失的控制 → 后果） | 严重级 | 证明它的测试 |
    |---|---|---|---|---|---|
    | SEC-01 | `src/auth/verify.ts` | 令牌 `exp` 超过 60s 即拒绝，比较用常量时间 | 过期令牌 → 未校验时效 → 会话劫持 | Critical | `test_rejects_expired_beyond_grace` |

    Then, after the table, the same constraints as a copy-pasteable list —
    this is the form that goes into the plan:

    ```
    [SEC-01] src/auth/verify.ts: 令牌 exp 超过 60s 即拒绝，比较用常量时间
             （攻击路径：过期令牌 → 未校验时效 → 会话劫持）
    ```

    ### Open Questions for the Human

    Security-relevant decisions the spec does not make. One per item: the
    question, your recommendation, what the safe default is if nobody answers,
    and what each option costs. These are NOT constraints and must not be
    implemented as requirements.

    ### Out of Scope

    Everything you considered and dismissed, one line each, with the reason
    (not reachable / test-only / closed by <spec section> / low consequence).
    Nothing is dropped silently.
```

---

## Placeholders

- `[MODEL]` — **必填**。这是设计任务，不是机械任务，用有判断力的模型。
- `[SPEC_FILE]` — **必填**。brainstorming 产出的 spec / 设计文档路径。
- `[RELATED_DOCS]` — 可选。spec 引用到的接口定义、信任边界、数据模型文档。
- `[CODEBASE_SURFACE]` — 可选。已有代码项目的入口点与现有安全控制位置。全新项目留空。
- `[OUTPUT_FILE]` — **必填**。约束清单写到这里。**不要写进对话**——按 SDD 的规矩，产物以文件交接，
  留在对话里的每行字后面都要被反复重读。

## 派发后你要做的

1. 读 `### Open Questions for the Human`，**逐条裁决**。安全决策是你的，不是子代理的。
2. 把 `### Security Constraints` 里的 copy-pasteable 列表，贴进 plan 的安全约束段
   （见 `plan-security-section.md`）。
3. 确认每条约束都有测试归属——没有测试归属的约束，实现者会当成建议。

## 与 plan 段落的对应

威胁建模器产出的 `[SEC-01] ...` 列表，**原样**进入 plan 的 `## Security Constraints`。
每条约束对应的测试，被指派到拥有该组件的那个任务的步骤里。
