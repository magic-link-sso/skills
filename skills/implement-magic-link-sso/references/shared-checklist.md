# Shared Checklist

## Always confirm these before implementation

- Which framework is actually being integrated: Angular, Fastify, Next.js, Nuxt,
  Django, or whether the app should use Magic Link SSO Gate instead
- Whether the app wants a local login page or direct redirect to the hosted SSO
  `/signin` page
- Which app origin is used for `returnUrl`
- Whether the app itself owns the verify callback and therefore needs to send
  `verifyUrl`
- Whether the project should use the published framework package or a local
  workspace version for unreleased changes
- Which `[[sites]]` entry in the Magic Link SSO server TOML should handle the
  app
- Whether the app needs full access or a narrower requested `scope`
- For Gate: whether the protected resource should use a subdomain or path prefix
- For Gate: whether the upstream is static, dynamic, or both

## Always align these settings with the SSO server

- `[cookie].name`
- `[auth].jwtSecret`
- `[auth].previewSecret` when the framework, app, or Gate previews verification
  tokens before exchange
- the app's expected token issuer, derived from `MAGICSSO_SERVER_URL`
- the app origin inside the matching `[[sites]].origins`
- either `allowedEmails` for full access or `[[sites.accessRules]]` for scoped
  access
- the verify callback origin when the app sends a custom `verifyUrl`
- for Gate, the public Gate origin as the expected token audience
- for Gate, the upstream URL and any base-path expectations

If these are misaligned, the app may appear to integrate correctly while still
failing with redirect loops, forbidden sign-in responses, or missing-session
symptoms.

## Always validate these paths

- anonymous visit to a protected page
- login form or hosted sign-in redirect
- magic-link email submission success
- verify callback sets the auth cookie
- verify callback uses preview-on-GET at `/verify-email` and exchanges the token
  only on explicit POST
- verify preview requests send the configured preview secret and fail safely
  when it is wrong
- verify callback rejects tokens whose `aud` or `iss` do not match the app
  origin and Magic Link SSO server origin
- authenticated visit to a protected page
- authenticated code can read `email`, `scope`, and `siteId` when the framework
  exposes them
- logout clears the cookie
- logout is POST-only and uses the framework's CSRF or same-origin protection
- invalid or replayed token fails safely
- for Gate, anonymous document requests redirect while anonymous API requests
  fail safely without leaking upstream content
- for Gate, static assets, SPA bundles, SSE, and websocket upgrades behave as
  intended after authentication

## Security reminder

Access tokens are site-bound. Newly issued tokens must carry `siteId`, `aud`,
and `iss`, and direct token verification must supply the expected audience and,
when available, expected issuer.

After upgrading to this model, older auth cookies no longer verify and users
must sign in again.

## Keep context lean

For implementation work:

1. Use this umbrella skill only long enough to identify the framework.
2. Read exactly one bundled framework reference.
3. Read only the extra references needed for that framework.

Do not load all framework references unless the user explicitly asks for a
comparison or migration discussion.
