# LACS — Linux Agent Control Standard

**LACS is an open protocol for AI agents that operate at the Linux system level.**

It defines how an AI planning layer, a user-facing approval layer, and a privileged execution layer interact — with typed actions, formal risk levels, mandatory approval gates, and an immutable audit trail.

---

## The Problem

Every current AI system agent operates on raw shell text. The agent generates a command. The shell runs it. The user finds out what happened after.

There is no formal model of what "safe" means. No type system. No mandatory review. No audit trail.

LACS defines that model.

---

## Core Concepts

**Typed actions, not shell commands.** A LACS action is a named, structured operation with a declared parameter schema — not a string of bytes handed to `/bin/sh`. The executor knows what `AddLayeredPackage` *is*. It knows what it will do, how to preview it, and how to roll it back.

**Formal risk levels.** Every action carries a risk level — `Low`, `Medium`, or `High` — enforced by the executor, not suggested to the user.

| Risk Level | Semantics | Approval Required |
|---|---|---|
| Low | Read-only, no side effects | None |
| Medium | Reversible side effect | Yes/No confirmation |
| High | Irreversible or access-control mutation | Must type the action name |

**Privilege separation.** Three processes with three trust levels:

```
Planner  →  Shell  →  Daemon
(untrusted) (user)  (privileged)
```

The Planner proposes. The Shell approves. The Daemon executes. The Planner never touches the Daemon.

**Immutable audit trail.** Every execution is logged with the action name, parameters, risk level, approval mode, status, and timestamps. Append-only. No deletions.

---

## Specification

→ [spec.md](spec.md) — The full LACS protocol specification (version 2026-04-15)

The specification uses [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) language (MUST, SHOULD, MAY) for normative requirements.

---

## Reference Implementation

**SysKnife** is the reference implementation of LACS — a Rust binary targeting Fedora Atomic Desktops.

→ https://github.com/lacs-foundation/sysknife

---

## Why This Matters

LACS is to AI system agents what MCP is to AI tool integration: a standard interface so that safety properties can be verified, audited, and built upon — rather than re-invented (incorrectly) by each implementation.

If you are building an AI agent that touches a Linux system, you should either implement LACS or have a clear answer for why your trust model is stronger.

---

## Contributing

LACS is an open protocol. Contributions to the specification are welcome via issues and pull requests.

- Found an ambiguity in the spec? Open an issue.
- Building a LACS-conforming implementation? Open a PR to add it to the reference implementations list.
- Proposing a new concept? Start with an issue to discuss before drafting a spec change.

---

## License

LACS specification — [CC0 1.0 Universal](LICENSE) (public domain dedication)
