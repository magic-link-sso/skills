---
name: implement-magic-link-sso
description:
    Choose and coordinate the right Magic Link SSO integration for supported web
    frameworks. Use when an agent needs to implement, explain, compare, or
    troubleshoot Magic Link SSO in Angular SSR, Fastify, Next.js App Router,
    Nuxt SSR, or Django projects, when the framework is unclear and must be
    inferred from the codebase, when a static or CSR-only app should be
    protected through Magic Link SSO Gate, or when the agent should choose the
    correct bundled framework guide instead of re-deriving the integration from
    scratch. Assume the project has access to the Magic Link SSO server and,
    where relevant, the published framework packages on npm or PyPI unless the
    repo clearly indicates a local workspace or unreleased package build.
---

# Implement Magic Link SSO

## Overview

Use this skill as the umbrella router for Magic Link SSO work across the
supported frameworks and the Magic Link SSO Gate reverse-proxy path. Do not
repeat the full framework-specific instructions here. First identify whether the
target is a supported framework integration or a better fit for Gate, then read
exactly one bundled reference unless the user is explicitly asking for a
comparison across frameworks.

## Routing Workflow

1. Identify whether the target is:
    - a supported framework integration
    - a static or CSR-only app that should use Gate
    - an unknown or unsupported stack that should use Gate
2. If the target is a supported framework, read only the matching bundled
   framework reference.
3. If the target is better served by Gate, read `references/gate.md`.
4. If the framework is unclear, inspect the repo for framework markers and use
   `references/framework-selection.md`.
5. If the user is comparing frameworks, read `references/framework-selection.md`
   and keep the answer comparative rather than implementation-heavy.
6. For any implementation task, also apply the shared checks in
   `references/shared-checklist.md`.

## Read One Framework Reference

Use exactly one of these references for implementation work:

- Angular: `references/angular.md`
- Fastify: `references/fastify.md`
- Next.js: `references/nextjs.md`
- Nuxt: `references/nuxt.md`
- Django: `references/django.md`
- Gate: `references/gate.md`

Prefer the most specific reference that matches the framework. Treat this
umbrella skill as the dispatcher and shared decision layer, not the detailed
implementation manual.

## Framework Detection

If the user did not name the framework explicitly, infer it from the repo:

- Angular indicators
    - `angular.json`
    - `src/main.ts`
    - `src/server.ts`
    - dependency on `@angular/core` or `@angular/ssr`
- Fastify indicators
    - `fastify` in `package.json`
    - `src/app.ts` or `src/main.ts` with Fastify route handlers
    - `app.get(...)` / `app.post(...)` route registration
    - dependency on `@fastify/cookie` or `@fastify/formbody`
- Next.js indicators
    - `next.config.*`
    - `src/app/`
    - `proxy.ts`
    - dependency on `next`
- Nuxt indicators
    - `nuxt.config.ts`
    - `app/pages/`
    - `server/api/`
    - dependency on `nuxt`
- Django indicators
    - `manage.py`
    - `settings.py`
    - `urls.py`
    - dependency on Django or `pyproject.toml`

Prefer Gate when the repository instead looks like:

- static HTML, CSS, and JS assets with no server integration point
- a pure SPA build output behind a generic web server
- an unsupported backend or frontend framework
- an app that should remain unaware of Magic Link SSO entirely

When multiple frameworks exist in the same repository, prefer the one the user
named. If the user did not name one, anchor on the app they are currently
editing rather than the existence of unrelated sample apps elsewhere in the
repo.

## Shared Decision Points

Every framework-specific integration still needs these choices:

- local app-owned login flow vs hosted SSO sign-in flow
- exact auth cookie name and JWT secret alignment with the server TOML config
- which `[[sites]]` entry the app belongs to, including matching app origins
- whether email access should be full-site via `allowedEmails` or scoped via
  `[[sites.accessRules]]`
- whether the login flow should request a specific `scope`
- trusted `verifyUrl` origin alignment when the app owns the callback
- whether the project should consume the published framework package or a local
  workspace version for unreleased changes
- one successful browser round trip and one failure-path verification, including
  the granted `scope` when the app uses scoped access

Gate-based integrations still need these choices:

- whether the deployment should use a subdomain or path prefix
- whether hosted sign-in direct-use should stay enabled
- the Gate public origin used as the expected JWT audience
- the private upstream URL and whether it serves static files or dynamic traffic
- whether websocket and SSE proxying are needed
- how the upstream is kept private so Gate cannot be bypassed

Read `references/shared-checklist.md` before finishing any implementation.

## Comparison Mode

When the user asks which supported framework is easiest, smallest, or most
customizable:

- use `references/framework-selection.md`
- compare only Angular, Fastify, Next.js, Nuxt, and Django as covered by this
  skill pack
- highlight integration-model differences instead of generic framework opinions

## Unsupported Cases

If the user asks for a framework outside these supported paths, say that this
umbrella skill covers the current Magic Link SSO skill set:

- Next.js App Router
- Nuxt SSR
- Angular SSR
- Fastify
- Django
- Magic Link SSO Gate for static, CSR-only, unknown, or unsupported stacks

If the target cannot use one of the supported framework paths directly, prefer
the Gate path before falling back to a generic best-effort implementation.
