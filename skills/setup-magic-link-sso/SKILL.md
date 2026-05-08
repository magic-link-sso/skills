---
name: setup-magic-link-sso
description:
    Set up, deploy, bootstrap, or operate self-hosted Magic Link SSO
    infrastructure. Use when the work is about the Magic Link SSO server,
    Magic Link SSO Gate, or the optional manager in managed mode, when the
    operator needs help choosing between classic mode, Gate, and managed mode,
    or when the question is about local evaluation versus a production-like
    deployment shape rather than framework-specific app integration.
---

# Setup Magic Link SSO

## Overview

Use this skill for self-hosted Magic Link SSO infrastructure setup and operator
workflow. Keep it focused on the Magic Link SSO server, Magic Link SSO Gate,
and the optional manager. Do not use this skill for app-level framework wiring;
hand those questions to `implement-magic-link-sso`.

Start with `references/path-selection.md`, then read only the references needed
for the target deployment shape. Finish with
`references/shared-checklist.md`.

## Routing Workflow

1. Read `references/path-selection.md`.
2. Identify whether the operator needs:
   - server only in classic mode
   - server plus Gate
   - server plus manager in managed mode
   - a local evaluation stack or a production-like deployment
3. Read only the relevant reference files:
   - `references/server.md`
   - `references/gate.md`
   - `references/manager.md`
4. End with `references/shared-checklist.md`.
5. If the question becomes app or framework integration work, switch to
   `implement-magic-link-sso` instead of expanding this skill.

## Read Only What Applies

- Always read: `references/path-selection.md`
- Read `references/server.md` for any classic-mode or managed-mode server setup
- Read `references/gate.md` when Gate protects a private upstream or fronts the
  manager UI
- Read `references/manager.md` only when the operator wants managed mode
- Read `references/shared-checklist.md` before final recommendations

Keep context lean. Do not load every reference unless the user explicitly wants
comparisons across deployment shapes.

## Decision Boundaries

Use this skill when the task is primarily about:

- bootstrapping Magic Link SSO from Docker or source
- choosing between classic mode and managed mode
- deciding whether Gate is the right protection layer
- configuring the first server, Gate, or manager deployment
- validating a local or production-like Magic Link SSO setup

Do not use this skill as the main guide for:

- Angular, Next.js, Nuxt, Fastify, or Django app wiring
- app-owned login routes or verify-email callbacks
- framework package selection or framework comparisons

Those belong in `implement-magic-link-sso`.
