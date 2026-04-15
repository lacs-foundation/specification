# LACS — Linux Agent Control Standard

**Version:** 2026-04-15 (Draft)  
**Status:** Draft  
**Repository:** https://github.com/lacs-foundation/lacs

---

## Abstract

LACS (Linux Agent Control Standard) is an open protocol for AI agents that operate at the Linux system level. It defines the contracts between three privilege-separated components — a *Planner*, a *Shell*, and a *Daemon* — and mandates typed actions, formal risk classification, explicit approval gates, and an immutable audit trail.

LACS does not specify how planning is done, which AI model is used, or which Linux distribution is targeted. It specifies what must happen *between* the moment a user states an intent and the moment a privileged action runs on the system.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Terminology](#2-terminology)
3. [Architecture](#3-architecture)
4. [Typed Action Model](#4-typed-action-model)
5. [Risk Classification](#5-risk-classification)
6. [Plan Model](#6-plan-model)
7. [Approval Protocol](#7-approval-protocol)
8. [Execution Protocol](#8-execution-protocol)
9. [Audit Requirements](#9-audit-requirements)
10. [Wire Protocol](#10-wire-protocol)
11. [Security Considerations](#11-security-considerations)
12. [Conformance](#12-conformance)
13. [Appendix A — Action Definition Format](#appendix-a--action-definition-format)
14. [Appendix B — Approval UX Recommendations (non-normative)](#appendix-b--approval-ux-recommendations-non-normative)

---

## 1. Introduction

### 1.1 Motivation

Existing AI system agents operate on raw shell access. The agent sends text to a shell process. The operating system receives bytes. Nothing in this pipeline knows what the action *is*, whether it is reversible, or whether the user understood what was about to happen.

The consequences are predictable:

- `rm -rf ~/important_dir` and `ls ~/` are the same type to a shell.
- A confirmation dialog showing "Delete temporary files?" can hide `rm -rf` underneath it.
- After the fact, there is no record of what the agent proposed versus what the user approved versus what ran.

LACS addresses this by replacing shell text with a typed action model, imposing a formal risk-based approval gate, and requiring an immutable audit trail.

### 1.2 Design Goals

1. **The planner proposes; it never executes.** Planning and execution run in separate processes with no shared memory and no privilege escalation path.
2. **Actions are typed, not textual.** The system knows what an action *is*, not just which bytes it produced.
3. **Risk is formal, not advisory.** Risk levels are enforced by the executor, not suggested to the user.
4. **Approval is action-gated, not dialog-gated.** The user approves a named, typed action — not a natural-language description of what the agent thinks it will do.
5. **Every execution is auditable.** What was proposed, what was approved, and what ran are permanently recorded.
6. **Reversibility is planned, not hoped for.** High-risk actions must declare their rollback procedure at definition time.

### 1.3 Relationship to Existing Work

LACS is inspired by the [Model Context Protocol](https://modelcontextprotocol.io) (MCP) for AI tool integration and the [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) (LSP) for IDE standardization. Like both, LACS standardizes a protocol that would otherwise be re-invented — incorrectly — by each implementation.

LACS is not a replacement for MCP. A LACS Planner MAY itself be exposed as an MCP tool, allowing any MCP-compatible LLM client to use LACS as a safe system management backend.

---

## 2. Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

| Term | Definition |
|---|---|
| **Action** | A named, typed system operation with a defined parameter schema and a declared risk level. |
| **Action Catalogue** | The complete set of Actions that a conforming Daemon implementation supports. |
| **Plan** | An ordered list of one or more Plan Steps that together fulfill a user's stated intent. |
| **Plan Step** | A single Action invocation within a Plan, including its parameters and a human-readable summary. |
| **Planner** | The component that receives user intent, reasons about it, and proposes a Plan. The Planner has no ability to execute Actions. |
| **Shell** | The component that presents a Plan to the user, collects Approval, and forwards the approved Plan to the Daemon. |
| **Daemon** | The privileged component that validates, previews, executes, and audits Actions. |
| **Approval** | An explicit, user-originated authorization for a specific Plan Step to execute. |
| **Named Confirmation** | An Approval mode in which the user must type the exact Action name to proceed. |
| **Risk Level** | A formal classification (`Low`, `Medium`, or `High`) that determines the Approval mode required for a Plan Step. |
| **Audit Log** | An append-only record of every execution event, written by the Daemon. |
| **Rollback** | An automatic reversal of a failed High-risk Action, executed by the Daemon without further Approval. |

---

## 3. Architecture

### 3.1 Components

A LACS-conforming system consists of exactly three components:

```
User Intent
    │
    ▼
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Planner    │──Plan──►│    Shell     │──Plan+──►│    Daemon    │
│              │         │              │ Approval │              │
│ (untrusted   │         │ (presents +  │         │ (privileged  │
│  process)    │         │  approves)   │         │  process)    │
└──────────────┘         └──────────────┘         └──────────────┘
                                                         │
                                                         ▼
                                                    Audit Log
```

The three components MUST run as separate processes. The Planner MUST NOT share memory with the Daemon. The Planner MUST NOT communicate with the Daemon directly — all communication flows through the Shell.

### 3.2 Trust Levels

| Component | Trust Level | What it may do |
|---|---|---|
| Planner | Untrusted | Propose Plans. Nothing else. |
| Shell | User-level | Present Plans, collect Approval, forward to Daemon. |
| Daemon | Privileged | Validate, preview, execute, audit. |

The Daemon MUST NOT execute any Plan Step that has not been approved via the protocol-defined Approval flow. It MUST NOT accept Plan Steps directly from the Planner.

### 3.3 Communication Contracts

- **Planner → Shell**: The Planner emits a `Plan` object. The transport MAY be a Unix socket, stdin/stdout, or any IPC mechanism. The Planner MUST NOT be able to initiate execution.
- **Shell → Daemon**: The Shell sends an `ApprovedPlan` message containing the Plan and its Approval attestations. The transport SHOULD be a Unix domain socket with peer credential verification.
- **Daemon → Shell**: The Daemon streams execution progress events and a final `JobResult`. It also responds to preview requests before execution begins.

---

## 4. Typed Action Model

### 4.1 Definition

An **Action** is a named, formally described system operation. An Action is not a shell command. Two Actions that invoke the same underlying command are still distinct Actions if they represent different semantic operations.

Every Action MUST have:

- A unique `action_name` in `PascalCase`
- A declared `risk_level` (one of `Low`, `Medium`, `High`)
- A `params` schema (MAY be empty `{}` for zero-parameter actions)
- A human-readable `description`

Every `High`-risk Action MUST additionally declare a `rollback_procedure` that the Daemon can execute automatically on failure.

### 4.2 Action Names

Action names MUST:

- Use `PascalCase` (e.g., `AddLayeredPackage`, `GetDiskUsage`)
- Be stable: once published in a version of the Action Catalogue, a name MUST NOT change meaning
- Follow the semantic convention:
  - `Get*` — retrieve a single piece of state (read-only)
  - `List*` — enumerate a collection (read-only)
  - All others — mutating or mixed-effect operations

### 4.3 Action Parameters

Action parameters MUST be a JSON-serializable object. Parameters MUST be validated by the Daemon before execution. The Daemon MUST reject any Plan Step whose parameters fail schema validation.

### 4.4 Action Catalogue

An implementation's Action Catalogue MUST be versioned alongside the implementation. The Catalogue MUST:

- List every supported Action with its full definition
- Declare the risk level for each Action
- Declare rollback procedures for all `High`-risk Actions
- Be machine-readable (JSON Schema or equivalent)

The Planner MUST only propose Actions present in the Catalogue it was configured against. The Daemon MUST reject Plan Steps whose `action_name` is not in its Catalogue.

---

## 5. Risk Classification

Risk levels are formal properties of Actions, not advisory suggestions. The Daemon enforces them unconditionally.

### 5.1 Low Risk

An Action is `Low` risk if and only if it:

- Produces no persistent side effects on the system
- Does not write to any file, database, service, or network endpoint
- Can be repeated arbitrarily without consequence

*Examples: reading disk usage, listing installed packages, querying service status.*

Low-risk Actions do NOT require user Approval. The Daemon MAY execute them immediately upon receipt of a valid Plan.

### 5.2 Medium Risk

An Action is `Medium` risk if it:

- Produces a persistent side effect, AND
- The effect is reversible (manually or automatically)

*Examples: installing a package, enabling a service, creating a container.*

Medium-risk Actions REQUIRE affirmative user Approval before execution. The Daemon MUST NOT execute a Medium-risk Action without a valid Approval attestation from the Shell.

### 5.3 High Risk

An Action is `High` risk if it:

- Produces an irreversible side effect, OR
- Modifies access-control state (users, groups, SSH keys, sudo rules), OR
- Could compromise system availability (reboots, kernel parameter changes)

*Examples: removing layered packages, deleting SSH authorized keys, removing a user account.*

High-risk Actions REQUIRE Named Confirmation before execution (see [Section 7.2](#72-named-confirmation)). The Daemon MUST NOT execute a High-risk Action without a valid Named Confirmation attestation.

### 5.4 Risk Assignment Rules

Risk levels are assigned by the Action Catalogue author. Implementations MUST NOT downgrade a risk level at runtime based on user flags, configuration, or Planner-supplied values. The Daemon MUST derive risk level from its own Catalogue, not from the Plan Step.

---

## 6. Plan Model

### 6.1 Plan Structure

A Plan is an ordered list of Plan Steps produced by the Planner in response to a user's stated intent.

```json
{
  "intent": "install vim and enable the ssh service",
  "steps": [
    {
      "action_name": "AddLayeredPackage",
      "summary": "Install vim as a layered package",
      "risk_level": "Medium",
      "params": { "package": "vim" }
    },
    {
      "action_name": "EnableService",
      "summary": "Enable and start sshd",
      "risk_level": "Medium",
      "params": { "service": "sshd", "now": true }
    }
  ]
}
```

### 6.2 Plan Steps

Each Plan Step MUST include:

- `action_name`: exact Action name from the Catalogue
- `summary`: one-sentence human-readable description of what this step will do
- `risk_level`: SHOULD match the Catalogue (the Daemon verifies independently)
- `params`: parameter values conforming to the Action's parameter schema

The Shell MUST display the `action_name`, `summary`, and `risk_level` to the user for every Plan Step before requesting Approval.

### 6.3 Execution Ordering and Failure Semantics

Plan Steps execute in order. If any step fails:

- Execution of subsequent steps MUST halt
- If the failed step is `High` risk, the Daemon MUST attempt rollback (see [Section 8.3](#83-rollback))
- The Daemon MUST record the failure in the Audit Log regardless of rollback outcome

---

## 7. Approval Protocol

### 7.1 Approval Modes

| Risk Level | Approval Mode | What the user does |
|---|---|---|
| Low | None | Nothing — executes automatically |
| Medium | Affirmative | Confirms yes/no |
| High | Named Confirmation | Types the exact Action name |

### 7.2 Named Confirmation

For every `High`-risk Plan Step, the Shell MUST:

1. Display the Action name, summary, risk level, and an explicit irreversibility warning
2. Prompt the user to type the exact `action_name` string
3. Only forward the Approval attestation if the typed string exactly matches `action_name`

Named Confirmation MUST be case-sensitive. The Shell MUST NOT accept partial matches, abbreviations, or `y`/`yes` in place of the action name.

The design intent is analogous to `sudo` requiring a password: friction proportional to consequence.

### 7.3 Approval Scope and Expiry

An Approval attestation is scoped to exactly one Plan Step in one Plan instance. Approval for `AddLayeredPackage` in one plan does not constitute Approval for the same action in a later plan.

Implementations MAY support a `--yes` flag that pre-approves all `Medium`-risk steps in a single Plan invocation. Implementations MUST NOT pre-approve `High`-risk steps via any flag or configuration.

### 7.4 Approval Attestation Format

When forwarding an approved Plan to the Daemon, the Shell MUST include an attestation for each approved step containing:

- `action_name`: the action that was approved
- `approval_mode`: `affirmative` or `named_confirmation`
- `approved_at`: Unix timestamp of when the user gave Approval

The Daemon MUST verify that every `Medium` and `High` risk Plan Step has a corresponding valid attestation before execution begins.

---

## 8. Execution Protocol

### 8.1 Preview

Before executing any Plan Step, the Daemon SHOULD generate and return a human-readable preview of what the action will do. The preview MUST be generated without executing the action.

The Shell SHOULD display the preview to the user as part of the Approval flow.

### 8.2 Execute

Execution proceeds step by step in Plan order. For each step:

1. Daemon validates the Approval attestation
2. Daemon validates action parameters against the Catalogue schema
3. Daemon executes the action
4. Daemon streams progress output to the Shell in real time
5. Daemon writes the result to the Audit Log

### 8.3 Rollback

When a `High`-risk Action fails, the Daemon MUST:

1. Attempt to execute the rollback procedure declared in the Catalogue for that action
2. Not require additional user Approval for the rollback
3. Record the rollback attempt and its outcome in the Audit Log

If rollback itself fails, the Daemon MUST surface a clear error describing the original failure, the current system state, and the rollback failure. The Daemon MUST NOT silently continue.

---

## 9. Audit Requirements

### 9.1 Required Fields

For every execution event, the Daemon MUST write an audit record containing at minimum:

| Field | Type | Description |
|---|---|---|
| `id` | integer | Monotonically increasing record identifier |
| `action_name` | string | Exact action name from the Catalogue |
| `params` | object | Parameter values as submitted |
| `risk_level` | string | `Low`, `Medium`, or `High` |
| `approval_mode` | string | `none`, `affirmative`, or `named_confirmation` |
| `status` | string | `succeeded`, `failed`, `rolled_back`, `rollback_failed` |
| `started_at` | integer | Unix timestamp (seconds) when execution began |
| `finished_at` | integer | Unix timestamp (seconds) when execution completed |

Implementations MAY add additional fields.

### 9.2 Immutability

Audit records MUST NOT be modified or deleted after they are written. Implementations MUST use an append-only storage mechanism. The audit store MUST reject update and delete operations on existing records.

### 9.3 Retention

Implementations SHOULD retain audit records indefinitely. Implementations MAY implement configurable retention policies. If records are purged, the purge event itself MUST be recorded with a timestamp and the count of purged records.

---

## 10. Wire Protocol

### 10.1 Message Format

Messages between components MUST be encoded as JSON objects. Each message MUST include a `type` field identifying the message kind. Implementations SHOULD use newline-delimited JSON (NDJSON) for streaming progress events.

### 10.2 Transport Recommendations

The Shell-to-Daemon connection SHOULD use a Unix domain socket. The socket SHOULD be permission-restricted to the user or group authorized to issue commands.

Implementations targeting remote management MUST use a mutually authenticated encrypted transport (e.g., mTLS over TCP).

### 10.3 Peer Credential Verification

When using a Unix domain socket, the Daemon SHOULD verify the peer credentials (UID, GID) of connecting clients and map GID membership to role-based authorization tiers (see [Section 11.2](#112-role-based-authorization)).

---

## 11. Security Considerations

### 11.1 Threat Model

LACS is designed to protect against:

1. **Accidental destruction** — An AI agent proposing a destructive action that the user executes without fully understanding the consequences.
2. **Prompt injection** — An attacker embedding instructions in system data (a filename, a log line, a package description) that cause the Planner to propose unauthorized actions.
3. **Approval laundering** — A sequence of individually plausible steps that collectively achieve an unauthorized outcome.
4. **Audit erasure** — Tampering with the audit log to conceal what an agent did.

LACS is NOT designed to protect against:

- A user who deliberately approves destructive High-risk actions with Named Confirmation
- Compromise of the Daemon process itself
- Side-channel attacks on the IPC transport

### 11.2 Role-Based Authorization

Implementations SHOULD support role-based authorization mapping OS group membership to permission tiers:

| Role | Permitted risk levels |
|---|---|
| Observer | Low only (read-only) |
| Dev | Low + Medium |
| Admin | Low + Medium + High |
| Boot | All (reserved for boot-time automation) |

The Daemon MUST enforce authorization before executing any action, independently of the Approval attestation received.

### 11.3 The Planner is Untrusted

Implementations MUST treat the Planner as an untrusted input source. The Daemon MUST:

- Validate every field of every Plan Step independently
- Look up risk level from its own Catalogue (never trust the Plan Step's declared value)
- Validate parameters against its own schema

A Planner compromised by prompt injection can only propose actions. It cannot execute them, downgrade their risk level, or bypass the Approval gate.

### 11.4 Rate Limiting

Implementations SHOULD implement rate limiting on Planner invocations to protect against runaway loops and prompt-injection-driven abuse. Rate limits SHOULD persist across process restarts.

---

## 12. Conformance

A LACS-conforming implementation **MUST**:

- [ ] Run Planner, Shell, and Daemon as separate processes with no shared memory
- [ ] Implement the typed Action model with a versioned Action Catalogue
- [ ] Enforce risk levels as defined in Section 5 unconditionally
- [ ] Require Affirmative Approval for all Medium-risk actions before execution
- [ ] Require Named Confirmation for all High-risk actions before execution
- [ ] Prevent direct Planner-to-Daemon communication
- [ ] Write audit records for every execution event with the fields defined in Section 9.1
- [ ] Use an append-only audit store that rejects modification and deletion
- [ ] Reject Plan Steps whose `action_name` is not in the Daemon's Catalogue
- [ ] Reject Plan Steps whose parameters fail schema validation
- [ ] Attempt automatic rollback on High-risk action failure
- [ ] Derive risk level from the Daemon's Catalogue, not from the Plan Step

A LACS-conforming implementation **SHOULD**:

- [ ] Generate and display a preview before executing each Plan Step
- [ ] Implement role-based authorization on the Daemon
- [ ] Use a Unix domain socket with peer credential verification for Shell-to-Daemon IPC
- [ ] Implement rate limiting on Planner invocations
- [ ] Retain audit records indefinitely

---

## Appendix A — Action Definition Format

An Action Catalogue entry MUST be expressible in the following format:

```json
{
  "action_name": "AddLayeredPackage",
  "description": "Install a package as a layered package. Takes effect after the next reboot.",
  "risk_level": "Medium",
  "params": {
    "type": "object",
    "properties": {
      "package": {
        "type": "string",
        "description": "The package name to install",
        "minLength": 1
      }
    },
    "required": ["package"]
  }
}
```

A `High`-risk Action MUST additionally declare a `rollback`:

```json
{
  "action_name": "RemoveLayeredPackages",
  "description": "Remove all layered packages from the system.",
  "risk_level": "High",
  "params": {},
  "rollback": {
    "description": "Re-install the package list captured before removal.",
    "action_name": "RestoreLayeredPackages",
    "requires_pre_execution_snapshot": true
  }
}
```

---

## Appendix B — Approval UX Recommendations (non-normative)

**For Medium-risk steps:**

```
  AddLayeredPackage
  Install vim as a layered package
  Risk: MEDIUM  ·  Requires reboot

Approve? [y/N]
```

**For High-risk steps:**

```
  RemoveLayeredPackages
  Remove all layered packages from the system
  Risk: HIGH  ·  Irreversible without reinstall

  This action cannot be automatically undone.
To confirm, type the action name: _
```

The visual distinction between Medium and High SHOULD be significant — color, spacing, and an explicit irreversibility warning. The Named Confirmation prompt SHOULD appear on its own line with no default value.

**For the audit log display:**

```
$ sysknife history --limit 5

  2026-04-14 11:23  AddLayeredPackage      vim    succeeded   MEDIUM
  2026-04-14 11:20  EnableService          sshd   succeeded   MEDIUM
  2026-04-13 09:44  GetDiskUsage           --     succeeded   LOW
  2026-04-12 18:02  RemoveLayeredPackages  --     declined    HIGH
  2026-04-12 16:30  ListLayeredPackages    --     succeeded   LOW
```

Declined actions (where the user refused Approval) SHOULD appear in the audit log. The absence of a declined action from the log is itself a gap in the audit trail.

---

## Reference Implementation

The reference implementation of LACS is **SysKnife** — a Rust implementation targeting Fedora Atomic Desktops (Silverblue, Kinoite, and variants).

- Repository: https://github.com/lacs-foundation/sysknife
- Binary: `sysknife`
- Status: Active development

---

*LACS is an open protocol. Implementations in any language for any Linux distribution are encouraged.*  
*To list your implementation here, open an issue in this repository.*
