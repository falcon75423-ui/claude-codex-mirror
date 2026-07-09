---
name: co-calibrate-plan
description: 'Cross-vendor plan calibration via Codex MCP. Use at the planning stage to leverage an external LLM as peer planner or critic. Auto-routes by argument: no arg = parallel-planning mode (Codex drafts an alternative plan in parallel; compare); plan-file path = review mode (Codex audits the existing plan; compare). Anti-anchoring: each side finishes its own version before sharing. Triggers when user types "/co-calibrate-plan", "/co-calibrate-plan <plan-file>", or says "calibrate this plan", "have codex review my plan", "second opinion on plan", "find holes in plan", "parallel plan", "他山看一下計畫", "plan 找漏洞".'
---

# /co-calibrate-plan — 計畫期跨 vendor 對撞

> **時機**：計畫期。對你的 plan 找漏洞、或讓 codex 並行做一個版本對比。
> **路徑**：MCP（codex MCP server）。反偏見模式（先各做各的再對比）。
> **共通骨架**：[`co-base-packet.md`](../../../../references/co-base-packet.md)。

---

## Step 0：判斷 mode（根據 $ARGUMENTS 自然分流）

| `$ARGUMENTS` | Mode | 行為 |
|--------------|------|------|
| 空 / 任務描述 | **Mode A：並行做 plan** | Codex 從零做一個 plan，Claude 同時也做，做完對撞 |
| 既有檔案路徑（如 `.claude/plans/foo.md`）| **Mode B：review plan** | Codex review 該 plan 找漏洞，Claude 同時也 review，做完對撞 |

🔴 判斷規則：
- 看 `$ARGUMENTS` 開頭——是檔案路徑（`./` 或絕對路徑或 `.claude/...`）→ Mode B
- 否則視為任務描述 → Mode A
- 若 Mode B 但檔案不存在 → 直接回報使用者「找不到 plan 檔案 `<path>`，是不是要走 Mode A？」

---

## Mode A：並行做 plan（無參數 / 任務描述）

### Step 1A：派 background subagent 跑 codex

🔴 用 Task tool + `run_in_background: true` 派 subagent 呼叫 `mcp__validate-plans-and-brainstorm-ideas__codex`：

#### Codex prompt 模板

```text
You are an external mirror, not a sub-agent. Your job is to surface angles I would miss on my own — not to do my work for me.

Constraints:
- You are a peer, not a junior assistant. Disagree freely.
- "You're absolutely right!", "That sounds reasonable", or any non-critical agreement is forbidden.
- Even if my framing seems clear, before answering, prepare at least 3 specific points where you would push back.
- Be direct and specific. Lead with what I most likely missed.
- Stay scoped. If asked about X, do not give general advice — answer X.

Create a detailed implementation plan for the following task. Think deeply about architecture, steps, edge cases, and trade-offs — but DO NOT share the plan yet. If you need to ask clarifying questions about the task before planning, ask them now. Otherwise, when your plan is fully formed, respond with EXACTLY: "PLAN_READY" and nothing else. Wait for my next message before sharing the plan.

Task: $ARGUMENTS
```

#### Codex 設定

| 參數 | 值 |
|------|---|
| `sandbox` | `"read-only"` |
| `cwd` | 目前的 working directory |

> approval 行為由本地 profile 檔 `~/.codex/co_mirror.config.toml`（codex ≥0.144 新格式：獨立檔、頂層鍵值）治理（`approval_policy = "never"`）。不要把它當 MCP 參數寫死——本地 codex 設定一處掌管所有 mirror 呼叫，codex CLI 升版時不用各處改。

### Step 2A：Claude 自己做 plan

🔴 **不可在 Step 3 前看 codex 結果**。同時自己做一個完整 plan：架構、步驟、edge cases、trade-offs。

### Step 3A：取回 codex plan 並對撞

確認 subagent 回報 `PLAN_READY` 後，用 `codex-reply`：

| 參數 | 值 |
|------|---|
| `threadId` | codex session 的 thread ID |
| `prompt` | `Now share your plan. Be direct and specific. Highlight where your approach diverges from what a default Claude plan would look like.` |

呈現格式：

````markdown
## Codex（他山）的 plan

```
{verbatim 原文}
```

## Claude 的 plan

{Claude Step 2A 自己做的 plan}

## 對撞結果

- **Claude 漏掉的步驟**：...
- **Codex 沒考慮到的 edge case**：...
- **架構選擇分歧**：兩邊各推哪條路、各自理由
- **更簡單的選項**：（如有）
- **整合後的最終 plan 雛形**：...

## ⚖️  退場閘門

下一輪會挖到新層次的盲點嗎？同層次更細缺口 = 該停。
````

---

## Mode B：review plan（有 plan-file 參數）

### Step 0B：讀 plan 檔 + 找 original request

1. Read 該 plan 檔的完整內容
2. 從對話歷史找到「最初讓使用者要做這個 plan 的 user prompt」（context 不夠時，向使用者問一句確認）

### Step 1B：派 background subagent 跑 codex

🔴 用 Task tool + `run_in_background: true` 派 subagent：

#### Codex prompt 模板（替換大括號內容）

```text
You are an external mirror, not a sub-agent. Your job is to surface angles I would miss on my own — not to do my work for me.

Constraints:
- You are a peer, not a junior assistant. Disagree freely.
- "You're absolutely right!", "That sounds reasonable", or any non-critical agreement is forbidden.
- Even if my framing seems clear, before answering, prepare at least 3 specific points where you would push back.
- Be direct and specific. Lead with what I most likely missed.
- Stay scoped. If asked about X, do not give general advice — answer X.

You are a staff engineer reviewing this plan. Analyze it for critical issues, big simplifications, or a completely different better approach — but DO NOT share your review yet. If you need to ask clarifying questions about the plan or original request before reviewing, ask them now. Otherwise, when your review is fully formed, respond with EXACTLY: "REVIEW_READY" and nothing else. Wait for my next message before sharing your review.

Original request from the user:
<original_request>
{paste the user's original prompt/request that triggered the plan}
</original_request>

Plan:
<plan>
{paste the FULL contents of the plan file, verbatim}
</plan>
```

#### Codex 設定

同 Mode A（`sandbox: read-only`、`cwd: 目前 working directory`；approval 由本地 co_mirror profile 檔治理）。

### Step 2B：Claude 自己 review 同個 plan

🔴 **不可在 Step 3 前看 codex 結果**。獨立 review：
- 致命缺陷或方向錯誤
- 簡化機會
- 漏掉的 edge cases / risks
- 是不是有完全不同更好的 approach？

### Step 3B：取回 codex review 並對撞

確認 subagent 回報 `REVIEW_READY` 後，用 `codex-reply`：

| 參數 | 值 |
|------|---|
| `threadId` | codex session 的 thread ID |
| `prompt` | `Go ahead, share your review. Be direct and concise. Do not repeat the plan back. Focus only on critical issues, big simplifications, or a completely different better approach. Remember: list at least 3 push-back points before any agreement.` |

呈現格式：

````markdown
## Codex（他山）的 review

```
{verbatim 原文}
```

## Claude 的 review

{Claude Step 2B 自己的 review}

## 對撞結果（逐 issue 處理）

對 codex 提的每個 issue + Claude 自己提的每個 issue，逐條：
- ✅ 採納並更新 plan
- ❌ override 並寫明為什麼當前 approach 比較好

🔴 採納 / override 由**使用者**拍板，不由 Claude 自動決定。

## ⚖️  退場閘門

下一輪會挖到新層次的盲點嗎？同層次更細缺口 = 該停。
````

---

## 繼續對話（兩 mode 通用）

要 push back codex 某個點 → 用 `codex-reply` + 同個 threadId 接續：

| 參數 | 值 |
|------|---|
| `prompt` | 你的回應（addressing points / explaining overrides / asking clarification）|

> 如果你要 override codex 的點，**寫明為什麼**——這樣 codex 才能根據你的理由 push back（mirror 對撞繼續往下挖）。

---

## 紅線（取自 [base packet](../../../../references/co-base-packet.md)）

| # | 紅線 |
|---|------|
| 1 | Codex 端永遠 `sandbox: read-only`；approval 行為由本地 co_mirror profile 檔（`~/.codex/co_mirror.config.toml`）治理（禁止寫成 MCP 參數）|
| 2 | Codex 結論 verbatim，禁止 Claude 摘要 / 翻譯 / 自動採納 |
| 3 | Codex 是資深同儕對手戲不是權威——逐條驗證 |
| 4 | 退場閘門軟提醒每次必印 |

🔴 **mirror 不是 sub-agent**——這個 Skill 不是叫 codex 替 Claude 做 plan，是讓 codex 提供**獨立平行視角**，最終由人對撞拍板。
