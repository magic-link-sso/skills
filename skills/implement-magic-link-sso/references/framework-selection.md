# Framework Selection

## Supported frameworks in this skill pack

- Angular SSR
    - integration model: helper package plus app-owned routes
    - package access: published npm package, or local workspace for unreleased
      changes
    - bundled reference: `references/angular.md`
- Fastify
    - integration model: fully app-owned
    - package access: none required
    - bundled reference: `references/fastify.md`
- Next.js App Router
    - integration model: package plus app-owned wiring
    - package access: published npm package, or local workspace for unreleased
      changes
    - bundled reference: `references/nextjs.md`
- Nuxt SSR
    - integration model: module-driven package plus app-owned login UX
    - package access: published npm package, or local workspace for unreleased
      changes
    - bundled reference: `references/nuxt.md`
- Django
    - integration model: package-managed views and middleware
    - package access: published PyPI package, or local workspace for unreleased
      changes
    - bundled reference: `references/django.md`

## Decision matrix

### Choose Angular when

- the app is already an Angular SSR project
- the team wants explicit app-owned Node SSR routes
- route protection should happen through an Angular `CanActivate` guard
- the app is comfortable composing helper functions into its own guard, session
  service, and Node server

Notable integration detail:

- the package is lower-level than the Nuxt and Django integrations because it
  provides helpers, not full app routes or a packaged guard
- the consuming app must implement its own `magicSsoAuthGuard`, `/api/session`,
  `/api/signin`, `/verify-email`, and `/logout`

### Choose Fastify when

- the app is a plain Node server without a meta-framework
- the team wants explicit server-owned routes and HTML rendering
- auth gating should happen in Fastify route handlers
- the app wants the smallest server-side integration surface while still owning
  the login and verify-email routes

Notable integration detail:

- there is no required reusable Fastify package; the integration is
  intentionally app-owned
- the app should own its login, verify-email, protected-route, and logout
  handlers directly

### Choose Next.js when

- the app is already a Next.js App Router project
- route protection should happen in `proxy.ts`
- server components or route handlers should verify auth directly
- the app wants a compact reusable package plus explicit app-owned wiring

Notable integration detail:

- the package includes middleware, token verification helpers, login helper, and
  logout helper
- the package helpers and auth payload support optional requested `scope`
- the consuming app still implements its own `/verify-email` route and custom
  login action when it needs to send `verifyUrl` and preserve `scope`

### Choose Nuxt when

- the app is already a Nuxt SSR project
- integration should feel module-driven
- the app benefits from built-in route middleware and built-in `/verify-email`
  and POST-only `/logout` routes
- the app can rely on server-side auth decisions in SSR

Notable integration detail:

- the module provides `magic-sso-auth`, `/verify-email`, POST-only `/logout`,
  `useMagicSsoAuth()`, and `useMagicSsoConfig()`
- auth helpers and payloads support optional requested `scope`
- the app still needs its own login page and sign-in API route if it wants a
  local login UX

### Choose Django when

- the app is already a Django project
- integration should be settings- and middleware-driven
- protected views should use decorators, manual checks, or middleware-enforced
  global auth
- the app wants a packaged login template and packaged login/verify/logout views

Notable integration detail:

- the package is fuller at the UI layer than the Nuxt module because it ships a
  login view and template
- middleware exposes `request.magic_sso_user_email` and
  `request.magic_sso_user_scope`
- the packaged login view can forward an optional requested `scope` and can
  enforce auth broadly with `MAGICSSO_AUTH_EVERYWHERE`

## Cross-framework differences that matter

- Angular
    - strongest fit for SSR apps that want explicit app-owned routing and Node
      server control around helper utilities
- Fastify
    - strongest fit for plain Node apps that want a server-rendered integration
      without framework-specific package abstractions
- Next.js
    - strongest fit for App Router server components and explicit route handlers
- Nuxt
    - strongest fit for SSR apps that want module-level wiring and built-in
      server routes
- Django
    - strongest fit for server-rendered Python apps that want package-managed
      views and middleware

## Shared invariants

Across all supported frameworks:

- use the published framework package by default where one exists; only switch
  to a local workspace or alternate source when the repo is intentionally
  testing unreleased changes
- cookie name must match server `[cookie].name`
- JWT secret must match server `[auth].jwtSecret`
- app origin must exist in the matching `[[sites]].origins`
- callback origin must match that same site when the app sends a custom
  `verifyUrl`
- requested `scope` must be granted by `allowedEmails` or
  `[[sites.accessRules]]`
- invalid or replayed verification links must never grant access

Read `shared-checklist.md` after choosing the framework.
