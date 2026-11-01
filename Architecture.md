# GroundGate architecture and flows

How the pieces fit, what runs where, and what happens on every path, good and bad. Every diagram below is Mermaid (renders on GitHub). Pre-rendered PNGs of the main ones are at the bottom for viewers that do not render Mermaid.

**Contents**
[1 Design principles](#1-design-principles) · [2 System context and trust boundaries](#2-system-context-and-trust-boundaries) · [3 Components](#3-components) · [4 Request lifecycle](#4-request-lifecycle) · [5 Typed actions and premises](#5-typed-actions-and-premises) · [6 Evidence cascade](#6-evidence-cascade) · [7 Verdict to decision](#7-verdict-to-decision) · [8 The action token](#8-the-action-token) · [9 Protected executor](#9-protected-executor) · [10 Failure flows](#10-failure-flows) · [11 Process isolation](#11-process-isolation) · [12 Agent integration (LangGraph)](#12-agent-integration-langgraph) · [13 Evaluation architecture](#13-evaluation-architecture) · [14 What the architecture does not do](#14-what-the-architecture-does-not-do) · [15 Rendered images](#15-rendered-images)

---

## 1 Design principles

| Principle | What it means in the code |
|---|---|
| **The agent is untrusted** | It sends JSON. It never holds the HMAC key, a token, the verifier connection or a write handle. Its free-text explanation is never evidence. |
| **Facts, not permissions** | The gate derives the *premises* of a call (order exists, belongs to this customer, status allows it, amount fits) from the tool schema, then checks them against a record. |
| **Absence is not contradiction** | A missing cache entry or an unreachable source is `UNVERIFIABLE`, never `CONTRADICTED`. Only evidence that disagrees contradicts. |
| **Pay for certainty by risk** | Cache first; at most one authoritative lookup, and only when the action's risk calls for it. High-risk actions need authoritative support. |
| **A verdict expires** | The executor re-derives and re-checks every premise inside the write transaction, so a change after the check cannot commit. |
| **Fail closed** | Verifier fault, audit failure, service timeout, malformed request, expired or replayed token: nothing executes. |

## 2 System context and trust boundaries

```mermaid
flowchart LR
  subgraph UNTRUSTED["Untrusted side"]
    LLM["Local LLM<br/>(Ollama, llama.cpp, vLLM)"]
    AG["Agent<br/>(LangGraph graph or script)"]
    CRM["Agent's CRM view<br/>may be stale or incomplete"]
    LLM <--> AG
    CRM --> AG
  end
  CH{{"JSON request / response<br/>max 16 KiB, no pickle"}}
  subgraph TRUSTED["Trusted process (spawned child)"]
    direction TB
    GATE["Gate<br/>build action, resolve evidence,<br/>decide, mint token"]
    EXEC["Protected executor<br/>only code that writes"]
    KEY["HMAC key, tokens"]
    GATE -->|"ALLOW + token"| EXEC
    KEY -.- GATE
    KEY -.- EXEC
  end
  CACHE[("Cache<br/>incomplete, may be stale")]
  SORR[("System of record<br/>SQLite, read-only for the verifier")]
  SORW[("System of record<br/>write path, executor only")]
  AUDIT[("Audit log<br/>append-only JSONL, fsync")]
  AG -- "tool call" --> CH --> GATE
  GATE -. "lookup" .-> CACHE
  GATE -- "one bounded read" --> SORR
  EXEC -- "BEGIN IMMEDIATE ... COMMIT" --> SORW
  GATE --> AUDIT
  EXEC --> AUDIT
  GATE -- "BLOCK / ESCALATE + reason" --> CH
  EXEC -- "status" --> CH
  CH --> AG
  classDef bad fill:#f8e3df,stroke:#b83a2b,color:#1c2430
  classDef ok fill:#e4f3ea,stroke:#1d7a4a,color:#1c2430
  classDef core fill:#e3ebf8,stroke:#1f3a68,color:#1c2430
  class LLM,AG,CRM bad
  class SORR,SORW ok
  class GATE,EXEC core
```

**Out of scope:** an attacker with the same OS user can ptrace the child or open the database file. Closing that needs OS-level separation (distinct uid or container). See [`THREAT_MODEL.md`](THREAT_MODEL.md).

## 3 Components

```mermaid
flowchart TB
  subgraph CORE["src/groundgate (standard library only)"]
    M["model.py<br/>typed actions, premises,<br/>validation, risk tier"]
    S["sources.py<br/>Cache, ReadOnlySource,<br/>budget, circuit breaker"]
    G["gate.py<br/>Gate: cascade, policy, tokens<br/>ProtectedExecutor, AuditLog"]
    V["service.py<br/>TrustedService:<br/>process-isolated JSON channel"]
    M --> G
    S --> G
    G --> V
  end
  subgraph AGENT["agent/ (optional extra: langgraph)"]
    GR["graph.py<br/>LangGraph agent"]
    AC["actuators.py<br/>GuardedActuator"]
    LC["llm.py<br/>OpenAI-compatible client"]
    RN["run_natural.py<br/>paired runner"]
    GR --> AC
    GR --> LC
    RN --> GR
  end
  AC -- "JSON only" --> V
  EV["scenarios/ benchmarks/ redteam/<br/>evaluation and attack suites"] --> G
  classDef core fill:#e3ebf8,stroke:#1f3a68,color:#1c2430
  class M,S,G,V core
```

| File | Responsibility |
|---|---|
| `src/groundgate/model.py` | tool table, premise predicates, `build_ir` (validation and risk), `rederive_ir` |
| `src/groundgate/sources.py` | `Cache` (freshness by TTL), `ReadOnlySource` (SQLite `mode=ro`, lookup budget, breaker) |
| `src/groundgate/gate.py` | `Gate.evaluate/run`, `ProtectedExecutor.run`, `AuditLog`, token mint and verify, `init_db` |
| `src/groundgate/service.py` | `TrustedService` child process and the agent-facing `submit()` |
| `agent/` | LangGraph agent, local-model client, 40 tickets, paired natural-error runner |

The core never imports `agent/`. The only way the agent reaches the gate is `TrustedService.submit()`.

## 4 Request lifecycle

```mermaid
sequenceDiagram
  autonumber
  participant A as Agent (untrusted)
  participant S as TrustedService (child)
  participant G as Gate
  participant C as Cache
  participant R as Record (read-only)
  participant X as Protected executor
  participant D as Record (write)
  participant L as Audit log
  A->>S: {tool, args, request_id}  (JSON)
  S->>G: ToolCall + actor from the authenticated session
  G->>G: validate, derive premises and risk tier
  alt invalid action or actor mismatch
    G->>L: BLOCK + reason
    G-->>A: BLOCK
  else valid
    G->>C: lookup (fresh? complete?)
    opt cache does not settle it, or action is high risk
      G->>R: one bounded read-only lookup
    end
    G->>G: verdict per premise, then aggregate
    G->>G: policy: ALLOW / BLOCK / ESCALATE
    G->>L: decision record (written BEFORE any token exists)
    alt ALLOW
      G->>X: action + single-use token
      X->>D: BEGIN IMMEDIATE, recheck, conditional UPDATE, COMMIT
      X->>L: execution record
      X-->>A: EXECUTED (or an abort status)
    else BLOCK or ESCALATE
      G-->>A: decision + reason, nothing executed
    end
  end
```

## 5 Typed actions and premises

The gate does not read the agent's explanation. It builds a typed action from `(tool, args, request_id)` and derives the premises itself.

| Tool | Reversible | Premises | Notes |
|---|---|---|---|
| `refund_order(order_id, customer_id, amount_cents)` | no | `exists`, `owner`, `refundable_status` (delivered), `amount_within_remaining` | risk is **high** when `amount_cents >= HIGH_AMOUNT_CENTS` (5000) |
| `cancel_order(order_id, customer_id)` | no | `exists`, `owner`, `cancellable_status` (placed or paid) | no amount, so always **high** |
| `update_address(order_id, customer_id, address)` | yes | `exists`, `owner`, `cancellable_status` | reversible, so low risk |

Premises that depend on `exists` inherit `CONTRADICTED` from it.

**Validation (any failure is an immediate BLOCK, before any evidence is read):** exact argument set and types (a bool is not an int); `order_id` matches `ORD-dddd`; amount in `(0, 10^9]`; `customer_id` format; address at most 200 characters with no control characters; `request_id` at most 128 characters; and, when the service has an authenticated actor, `customer_id` must equal it. A client-supplied actor field is ignored.

## 6 Evidence cascade

```mermaid
flowchart TD
  A["Tool call arrives"] --> B{"Schema and exact checks pass?"}
  B -- no --> X1["BLOCK: invalid action"]
  B -- yes --> C["Look up cache"]
  C --> D{"Fresh, fully supports,<br/>and action is low risk?"}
  D -- yes --> V1["Verdict from cache: SUPPORTED"]
  D -- no --> E{"Lookup budget left<br/>and circuit breaker closed?"}
  E -- no --> U["Premises stay UNVERIFIABLE"]
  E -- yes --> F["One read-only lookup<br/>of the system of record"]
  F -- "timeout or fault" --> U
  F -- "row, or confirmed absence" --> G["Authoritative evidence outranks cache<br/>conflicts are logged"]
  G --> H["Aggregate premises"]
  V1 --> H
  U --> H
  H --> I{"Any premise CONTRADICTED?"}
  I -- yes --> R1["Verdict CONTRADICTED"]
  I -- no --> J{"All SUPPORTED, and for a high-risk action<br/>all on authoritative evidence?"}
  J -- yes --> R2["Verdict SUPPORTED"]
  J -- no --> R3["Verdict UNVERIFIABLE"]
```

**Resolution rules**

* Authoritative evidence beats cache. A fresh cache entry is used only when no authoritative reading exists. A **stale** cache entry supports nothing.
* When both exist and disagree on a premise, authoritative wins and the premise is recorded under `conflicts` in the audit trace.
* A confirmed absence from the system of record is `CONTRADICTED` for `exists`. A cache *miss* is `UNVERIFIABLE`.
* A cache row missing a needed field (incomplete entry) leaves that premise unresolved.
* An optional LLM tier for natural-language policy claims may only **downgrade** a verdict; it never upgrades one and never overrides a contradiction. It is off by default.

**Premise verdict states**

```mermaid
stateDiagram-v2
  [*] --> UNVERIFIABLE: no usable evidence yet
  UNVERIFIABLE --> SUPPORTED: fresh or authoritative evidence agrees
  UNVERIFIABLE --> CONTRADICTED: authoritative evidence disagrees
  SUPPORTED --> CONTRADICTED: authoritative evidence disagrees
  SUPPORTED --> [*]
  CONTRADICTED --> [*]
```

**Lookup budget and circuit breaker** (inside `ReadOnlySource`)

```mermaid
stateDiagram-v2
  [*] --> Closed
  Closed --> Closed: lookup succeeds, fault counter reset
  Closed --> Open: 3 consecutive faults
  Open --> HalfOpen: cooldown elapsed (30 s default)
  Open --> Open: call inside cooldown is refused
  HalfOpen --> Closed: trial lookup succeeds
  HalfOpen --> Open: trial lookup fails
```
Refused or failed lookups leave premises `UNVERIFIABLE`; they are never read as "checked, fine". An optional per-action lookup budget raises `BudgetExhausted` the same way.

## 7 Verdict to decision

```mermaid
flowchart LR
  V{"Verdict"}
  V -- CONTRADICTED --> B1["BLOCK"]
  V -- SUPPORTED --> A1["ALLOW<br/>token issued"]
  V -- UNVERIFIABLE --> R{"Reversible action?"}
  R -- yes --> A2["ALLOW with log<br/>commit-time recheck still applies"]
  R -- no --> Q{"Escalation budget left?"}
  Q -- yes --> E1["ESCALATE<br/>nothing executes"]
  Q -- no --> B2["BLOCK"]
  classDef b fill:#f8e3df,stroke:#b83a2b,color:#1c2430
  classDef a fill:#e4f3ea,stroke:#1d7a4a,color:#1c2430
  classDef e fill:#fbf0d9,stroke:#a8650b,color:#1c2430
  class B1,B2 b
  class A1,A2 a
  class E1 e
```

| Verdict | Irreversible or high value | Reversible and low value |
|---|---|---|
| **CONTRADICTED** | BLOCK | BLOCK |
| **SUPPORTED** | ALLOW | ALLOW |
| **UNVERIFIABLE** | ESCALATE (BLOCK once the escalation budget is spent) | ALLOW and log |

The costs behind this table (harm 50, false block 1, escalation 5, lookup 0.1) are assumptions, not calibrated values. `ESCALATE` is a verdict today, not a workflow: there is no escalation consumer yet.

## 8 The action token

On ALLOW the gate writes its audit record first, then mints a token. No audit record, no token.

```mermaid
flowchart LR
  subgraph TOKEN["token = exp . nonce . version . tag"]
    E["exp<br/>now + 60 s"]
    N["nonce<br/>96 random bits"]
    VR["version<br/>row version seen at<br/>verification, or -1"]
    T["tag<br/>HMAC-SHA256"]
  end
  subgraph MSG["tag covers"]
    C["canonical action<br/>tool, args, request_id,<br/>risk tier, actor"]
    D["decision = ALLOW"]
    EN["exp, nonce, version"]
  end
  C --> T
  D --> T
  EN --> T
  K["key held only in the<br/>trusted process"] --> T
```

| Property | Enforced by |
|---|---|
| Bound to the exact action | the tag covers the canonical action, recomputed from live fields at verification; edited arguments break it |
| Expires | `exp`, 60 s |
| Single use | the nonce is inserted into `used_tokens` in the same transaction as the write |
| Not forgeable by the agent | the key never leaves the child process; comparison is constant-time |
| Scoped to the principal | the actor is part of the canonical action and of the idempotency key |

## 9 Protected executor

```mermaid
sequenceDiagram
  autonumber
  participant A as Agent
  participant G as Gate
  participant S as System of record (read-only)
  participant X as Protected executor
  participant D as Database (write)
  A->>G: refund_order(ORD-1001, C1, 12000)
  G->>G: build typed action, derive premises
  G->>S: one bounded read-only lookup
  S-->>G: row at version 4
  G->>G: premises SUPPORTED on authoritative evidence
  G->>G: write audit record, then mint token
  G->>X: action + token (HMAC, 60 s expiry, nonce, version)
  X->>X: verify token, re-derive premises itself
  X->>D: BEGIN IMMEDIATE
  X->>D: re-read row, re-check every premise
  X->>D: UPDATE ... WHERE version = 4
  X->>D: INSERT idempotency key and used-token nonce
  X->>D: COMMIT
  X-->>A: EXECUTED
```

**Check order inside the transaction**

```mermaid
flowchart TD
  S0["token valid and unexpired for THIS action?"] -- no --> GB["GateBypassError, nothing touched"]
  S0 -- yes --> T1["BEGIN IMMEDIATE (write lock)"]
  T1 --> I1{"idempotency key seen?"}
  I1 -- "same action" --> DUP["DUPLICATE"]
  I1 -- "different action" --> IC["IDEMPOTENCY_CONFLICT"]
  I1 -- no --> N1{"token nonce already used?"}
  N1 -- yes --> TR["TOKEN_REPLAYED"]
  N1 -- no --> P1{"every premise holds on the row<br/>as it is NOW?"}
  P1 -- no --> AP["ABORTED_PRECONDITION"]
  P1 -- yes --> V1{"strict_version on and<br/>version moved?"}
  V1 -- yes --> AV["ABORTED_STALE_VERSION"]
  V1 -- no --> U1["UPDATE ... WHERE version = seen"]
  U1 -- "0 rows" --> AP
  U1 -- "1 row" --> W["INSERT idempotency key and nonce, COMMIT"]
  W --> OK["EXECUTED"]
  classDef b fill:#f8e3df,stroke:#b83a2b,color:#1c2430
  classDef a fill:#e4f3ea,stroke:#1d7a4a,color:#1c2430
  class GB,DUP,IC,TR,AP,AV b
  class OK a
```

**Least privilege inside the executor.** Its SQLite connection installs an authorizer that allows only `SELECT`, `INSERT` into `idem` and `used_tokens`, and `UPDATE` of `orders.refunded_cents`, `status`, `address`, `version`. Any other statement is denied, so even a bug in the executor cannot, say, delete rows.

**Data model**

```mermaid
erDiagram
  orders {
    text order_id PK
    text customer_id
    text status
    int amount_cents
    int refunded_cents
    int version
    text address
    text notes
  }
  idem {
    text idem_key PK
    text result
    text action_hash
  }
  used_tokens {
    text nonce PK
  }
```
The three tables have no foreign keys. `idem` and `used_tokens` are written in the same transaction as the `orders` update, which is what ties them together.

| `ExecResult.status` | Meaning |
|---|---|
| `EXECUTED` | committed once |
| `DUPLICATE` | same idempotency key and same action: a retry, not a second write |
| `IDEMPOTENCY_CONFLICT` | `request_id` reused for a different action |
| `TOKEN_REPLAYED` | nonce already consumed |
| `ABORTED_PRECONDITION` | a premise failed on the live row, or the version-conditional update matched nothing |
| `ABORTED_STALE_VERSION` | (optional `strict_version`) the row moved since verification |
| `ERROR` | unexpected exception, rolled back |

## 10 Failure flows

| # | Situation | What happens | Result |
|---|---|---|---|
| 1 | Evidence is stale | a stale entry supports nothing; the gate falls through to the authoritative lookup; a high-risk action never rests on cache | allowed if the source agrees, else blocked |
| 2 | Authoritative source times out | premises stay `UNVERIFIABLE`; irreversible: ESCALATE (BLOCK after the budget); reversible: allow with a log | nothing irreversible runs on an unanswered question |
| 3 | Record changes after verification | executor re-reads inside `BEGIN IMMEDIATE`, re-checks, version-conditional UPDATE | `ABORTED_PRECONDITION`, nothing written |
| 4 | Token or request reused | nonce in `used_tokens`; idempotency key per actor | `TOKEN_REPLAYED` / `DUPLICATE` / `IDEMPOTENCY_CONFLICT` |
| 5 | Forged or stripped token, fake ALLOW | key lives only in the child; service ignores extra fields | BLOCK or `GateBypassError` |
| 6 | Audit write fails | no record, no token | BLOCK (fail closed) |
| 7 | Verifier raises | caught at `evaluate` | BLOCK (fail closed) |
| 8 | Service dead, slow or sent garbage | `submit()` times out or the child returns BLOCK | BLOCK (fail closed) |
| 9 | External API succeeds but the local commit fails | a local transaction cannot be atomic with an external call | **not solved here**; simulated mitigations in `benchmarks/external_effects.py` |

```mermaid
flowchart TD
  Q["Action approved earlier<br/>(token in hand)"] --> W{"What changed since?"}
  W -- "nothing" --> OK["premises hold -> EXECUTED"]
  W -- "record changed so a premise fails<br/>(status, owner, balance)" --> AB["recheck fails -> ABORTED_PRECONDITION"]
  W -- "unrelated field changed" --> OK2["premises still hold -> EXECUTED"]
  W -- "same request sent twice" --> DU["DUPLICATE, no second write"]
  W -- "token reused for a new request" --> RP["TOKEN_REPLAYED"]
  W -- "token edited or expired" --> BY["GateBypassError"]
  classDef b fill:#f8e3df,stroke:#b83a2b,color:#1c2430
  classDef a fill:#e4f3ea,stroke:#1d7a4a,color:#1c2430
  class AB,DU,RP,BY b
  class OK,OK2 a
```

## 11 Process isolation

| Lives in the trusted child | Lives on the agent side |
|---|---|
| HMAC key, every token | a JSON request and a JSON response |
| verifier (read-only) connection | the model, the prompt, the CRM view |
| executor (write) connection | no key, no token, no database handle |
| audit log handle | |

Only JSON crosses the pipe (never pickle). Requests over 16 KiB are refused by `submit()`; an oversized raw frame makes the pipe unreadable and the service exits, which fails closed (a denial of service, but nothing executes). Any key in a request other than `tool`, `args`, `request_id` (token, premises, decision, idempotency key) is ignored by construction.

## 12 Agent integration (LangGraph)

```mermaid
flowchart LR
  S([START]) --> L["crm_lookup<br/>stale or incomplete CRM view"]
  L --> P["propose<br/>local LLM -> one JSON tool call"]
  P --> J{"parse"}
  J -- "malformed: feedback, up to 2 retries" --> P
  J -- "valid tool call" --> A["act<br/>GuardedActuator -> TrustedService.submit"]
  J -- "none / gave up / LLM error" --> R["report"]
  A --> R --> E([END])
  classDef bad fill:#f8e3df,stroke:#b83a2b,color:#1c2430
  classDef core fill:#e3ebf8,stroke:#1f3a68,color:#1c2430
  class P bad
  class A core
```

The only path to a write is `parse -> act`. Parse failures, an empty proposal and an unreachable model all go straight to `report`, so nothing executes.

**Paired natural-error run** (`agent/run_natural.py`): one model proposal per (ticket, run) is judged four ways on identical state.

```mermaid
flowchart TB
  T["40 tickets<br/>intent fixed before any run"] --> G["LangGraph agent + local model"]
  G --> PR["recorded proposal"]
  PR --> LAB["label from the database truth<br/>(harmful_premise, correct, identity_mismatch,<br/>unintended, abstain, parse_failed)"]
  PR --> LIVE["GroundGate arm: LIVE<br/>graph act node -> TrustedService"]
  PR --> RP["replayed arms on identical state:<br/>no guard, closed-world cache, always query"]
  LIVE --> OUT["executed = order table changed in the DB"]
  RP --> OUT
  LAB --> TAB["tables: proposal classes, mistake families,<br/>arms, per-ticket view"]
  OUT --> TAB
  LIVE -. "integrity check: live vs replayed GroundGate" .-> RP
```

Status: harness built and tested with a scripted stand-in and a fake endpoint; **not yet run on a real model**. See [`../agent/README.md`](../agent/README.md).

## 13 Evaluation architecture

Four state views are kept apart so a decision made on one view and executed against another can be measured.

```mermaid
flowchart LR
  S0["S0<br/>what the agent saw"] --> R["R<br/>what the verifier could see<br/>(cache gaps, lag, outages)"]
  R --> S1["S1<br/>truth at verification"]
  S1 -- "time passes, drift" --> S2["S2<br/>truth at mutation"]
  S2 --> H["harm label<br/>(from S2 only)"]
  classDef lab fill:#fbf0d9,stroke:#a8650b,color:#1c2430
  class H lab
```

**Scenario runner pipeline (anti-circularity)**

```mermaid
flowchart TB
  F["scenario file<br/>label = valid | harmful<br/>written BEFORE any run"] --> ARMS
  subgraph ARMS["arms on a fresh DB per case"]
    A1["no guard"]
    A2["closed-world cache"]
    A3["always query"]
    A4["GroundGate"]
  end
  ARMS --> SNAP["snapshot AFTER scripted drift"]
  SNAP --> DIFF["executed = order row differs from snapshot<br/>(return values ignored)"]
  DIFF --> M["metrics with Wilson intervals"]
  A4 --> GP["harmful PASSED THE GATE (pre-commit)"]
  A4 --> HE["harmful EXECUTED (after recheck)"]
  GP -. "the gap is the executor's contribution,<br/>reported separately" .- HE
  F --> FZ["held-out: SHA-256 freeze,<br/>runner refuses if files changed"]
```

## 14 What the architecture does not do

* **Fresh-but-wrong cache on a low-value action passes the gate** (dev `S03`, held-out `H032`, `H041`, red-team X09). The executor's recheck catches it, not the gate.
* **A valid but unintended action by the right customer executes** (X38). No fact check can object to it.
* **Late-change protection comes from the executor recheck**, which any system with an atomic recheck can have. It is not credited to the gate's routing.
* **Not built:** escalation consumer, tamper-evident (hash-chained) audit log (it is append-only JSONL), key rotation, OS-level separation of the trusted process, outbox for external effects.
* **One domain:** three order tools on SQLite.

## 15 Rendered images

Pre-rendered from the same sources in [`diagrams/`](diagrams/). The roadmap image predates the latest plan; the Mermaid roadmap in the README is current.

<details><summary><b>Architecture</b></summary>

![Architecture](assets/architecture.png)
</details>

<details><summary><b>Verdict to decision</b></summary>

![Decision](assets/decision.png)
</details>

<details><summary><b>Commit sequence</b></summary>

![Commit sequence](assets/commit_sequence.png)
</details>

<details><summary><b>Evaluation framework (four state views)</b></summary>

![Evaluation framework](assets/evaluation_framework.png)
</details>

<details><summary><b>Evidence cascade (tall)</b></summary>

![Cascade](assets/cascade.png)
</details>

<details><summary><b>Original detailed architecture (all 11 nodes)</b></summary>

![Detailed architecture](assets/architecture_detailed.png)

Predates the external-effects model and does not draw it.
</details>
