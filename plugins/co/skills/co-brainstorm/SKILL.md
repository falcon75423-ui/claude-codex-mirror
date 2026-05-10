---
name: co-brainstorm
description: Cross-vendor mirror brainstorming via Codex. Use when the user has no clear direction yet and wants to bounce ideas off an external LLM (different training distribution = different blind spots). Spawns Codex via MCP in parallel while Claude independently brainstorms, then compares both. Triggers when user types "/co-brainstorm <topic>", or when user says "bounce ideas off codex", "want a second opinion before deciding", "explore alternatives", "what angles am I missing", "brainstorm with codex", "external mirror", "他山", or wants to use a different model as a peer thinker before forming their own conclusion.
---

# /co-brainstorm — 跨 vendor 對撞想法

> **時機**：沒方向時用。對某個主題想對撞想法、找 Claude 自己想不到的角度。
> **路徑**：MCP（codex MCP server）。反偏見模式（先各做各的再對比）。
> **共通骨架**：[`co-base-packet.md`](../../../../references/co-base-packet.md)（mirror 哲學、紅線、IPI 防禦）。

---

## Arguments

`$ARGUMENTS` = 要 brainstorm 的主題（一句到一段話）。

範例：
- `/co-brainstorm 怎麼設計 OAuth 同時支援 5 個 IdP 的後端架構`
- `/co-brainstorm 文字編輯器要不要做 vim mode、技術成本 vs 使用者價值`

---

## Step 1：派 background subagent 跑 codex（先 ready 不取結果）

🔴 **必須**用 Task tool + `run_in_background: true` 派 subagent。Subagent 呼叫 `mcp__validate-plans-and-brainstorm-ideas__codex`：

### Codex prompt 模板（複製貼上，替換 $ARGUMENTS）

```text
You are an external mirror, not a sub-agent. Your job is to surface angles I would miss on my own — not to do my work for me.

Constraints:
- You are a peer, not a junior assistant. Disagree freely.
- "You're absolutely right!", "That sounds reasonable", or any non-critical agreement is forbidden.
- Even if my framing seems clear, before answering, prepare at least 3 specific points where you would push back.
- Be direct and specific. Lead with what I most likely missed.
- Stay scoped. If asked about X, do not give general advice — answer X.

Brainstorm on the following topic. Think deeply, explore multiple angles, and prepare your ideas — but DO NOT share them yet. If you need to ask clarifying questions about the topic before brainstorming, ask them now. Otherwise, when your brainstorm is fully formed, respond with EXACTLY: "BRAINSTORM_READY" and nothing else. Wait for my next message before sharing your ideas.

Topic: $ARGUMENTS
```

### Codex 設定

| 參數 | 值 |
|------|---|
| `prompt` | 上面的 prompt（替換 `$ARGUMENTS`）|
| `sandbox` | `"read-only"` |
| `approval-policy` | `"never"` |
| `cwd` | 目前的 working directory |

Subagent 處理 codex 來回通信。如果 codex 問澄清問題（不回 `BRAINSTORM_READY`），subagent 用自己判斷 + codebase context 回答，直到 codex 完成思考並回 `BRAINSTORM_READY`。

---

## Step 2：Claude 同時自己 brainstorm

🔴 **不可在 Step 3 前看 codex 結果**。整個機制的價值就是兩個獨立版本對撞——提前看 codex 答案 = anchor bias 失敗。

獨立想：
- 多種 approaches 與 alternatives
- Trade-offs 與 risks
- Edge cases 與 constraints
- 創意或反常規角度

把自己的想法**明確寫下來**（用 markdown / code block，後面 Step 3 要對比）。

---

## Step 3：取回 codex 結果並對撞

確認 subagent 已回報 codex `BRAINSTORM_READY` 後，用 `mcp__validate-plans-and-brainstorm-ideas__codex-reply`：

| 參數 | 值 |
|------|---|
| `threadId` | codex session 的 thread ID（subagent 提供）|
| `prompt` | `Now share your brainstorm. Be direct and specific. Lead with the angles you think I most likely missed. Remember: list at least 3 push-back points before any agreement.` |

收到 codex 結果後，按以下順序呈現給使用者：

````markdown
## Codex（他山）的 brainstorm

```
{verbatim 原文，禁止改寫摘要}
```

## Claude 的 brainstorm

{Claude 自己 Step 2 寫的內容}

## 對撞結果

- **Claude 漏掉的角度**：...
- **更簡單的選項**：...
- **沒看到的風險**：...
- **兩邊都同意的核心**：...

## ⚖️  退場閘門

下一輪會挖到新層次的盲點嗎？同層次更細缺口 = 該停。
````

🔴 採納 / 不採納 / 部分採納由**使用者**拍板，不由 Claude 自動決定。

---

## 繼續對話（可選）

要深挖某個點 → 再用 `codex-reply` + threadId 接續：

| 參數 | 值 |
|------|---|
| `threadId` | 同一個 thread ID |
| `prompt` | 你的 follow-up 問題（挑戰 codex 的假設、探索另一種 alternative、測 edge case）|

---

## 紅線（取自 [base packet](../../../../references/co-base-packet.md)）

| # | 紅線 |
|---|------|
| 1 | Codex 端永遠 `sandbox: read-only` + `approval-policy: never` |
| 2 | Codex 結論 verbatim，禁止 Claude 摘要 / 翻譯 / 自動採納 |
| 3 | Codex 是資深同儕對手戲不是權威——逐條驗證 |
| 4 | 退場閘門軟提醒每次必印 |

🔴 **mirror 不是 sub-agent**——把 codex 當執行工人 = 失去 mirror 價值。
