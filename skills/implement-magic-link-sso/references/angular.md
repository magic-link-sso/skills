# Angular

## Access assumptions

The package is published to npm as `@magic-link-sso/angular` and can be
installed from the public npm registry:

```sh
npm install @magic-link-sso/angular
```

## Required app env vars

```env
MAGICSSO_COOKIE_NAME=magic-sso
MAGICSSO_COOKIE_MAX_AGE=3600
MAGICSSO_DIRECT_USE=false
MAGICSSO_JWT_SECRET=your_jwt_secret
MAGICSSO_PREVIEW_SECRET=your_preview_secret
MAGICSSO_SERVER_URL=http://localhost:3000
```

The bundled repository example in `examples/angular/.env.example` uses the local
server defaults:

```env
MAGICSSO_JWT_SECRET=VERY-VERY-LONG-RANDOM-JWT-SECRET
MAGICSSO_PREVIEW_SECRET=VERY-VERY-LONG-RANDOM-PREVIEW-SECRET
```

Important alignments:

- `MAGICSSO_COOKIE_NAME` must match server `[cookie].name`
- `MAGICSSO_JWT_SECRET` must match server `[auth].jwtSecret`
- `MAGICSSO_PREVIEW_SECRET` must match server `[auth].previewSecret` when the
  app owns `/verify-email`
- `MAGICSSO_COOKIE_MAX_AGE` should not exceed the effective JWT lifetime
- `MAGICSSO_SERVER_URL` must point at the reachable SSO server origin

## Required server site config

The Angular app origin must exist in the matching `[[sites]].origins` entry
inside the Magic Link SSO server TOML config.

The site must grant email access through `allowedEmails` or
`[[sites.accessRules]]`.

If the app requests a non-default `scope`, that scope must be granted for the
email on that same site.

When the app sends a custom `verifyUrl`, keep it same-origin with the selected
site.

## Important implementation nuance

`@magic-link-sso/angular` is intentionally helper-only. It does not provide a
packaged Angular route guard, login page, `/api/signin`, `/verify-email`, or
`/logout` route.

Those pieces live in the consuming app. Implement them around the helpers.

## Package exports and when to use them

- `resolveMagicSsoConfig(config?)` Use when app code needs the normalized
  `MAGICSSO_*` values.
- `getMagicSsoConfig(config?)` Use when app code wants the resolved config
  through the package helper.
- `verifyRequestAuth(request, config?)` Use in SSR request handling and
  same-origin session endpoints. It derives the expected audience from
  `request.url` and the expected issuer from the resolved `serverUrl`.
- `verifyAuthToken(token, secret, options)` Use when code already has the raw
  JWT and secret bytes. Pass `expectedAudience` and, when available,
  `expectedIssuer`.
- `buildLoginPath(appOrigin, returnTarget, config?, scope?)` Use when the app
  owns a local `/login` page.
- `buildLoginTarget(appOrigin, returnTarget, config?, scope?)` Use when code
  should respect `directUse` and possibly redirect to the SSO server directly.
- `buildVerifyUrl(appOrigin, returnUrl)` Use to create the local verify callback
  URL for sign-in submission.
- `normaliseReturnUrl(returnUrl, appOrigin, fallback?)` Use before redirects or
  login-target construction.
- `buildAuthCookieOptions(value, config?)` Use when a local verify route sets or
  clears the auth cookie.
- `getJwtSecret(config?)` Use before locally verifying the returned
  `accessToken`.

If the consuming app wants Angular-specific providers, guards, or session
services, treat those as app-owned glue. The example app's `provideMagicSso()`
and `magicSsoAuthGuard` live in the app, not in the published package.

## Typical app-owned responsibilities

- `magicSsoAuthGuard`
- `/api/session` for the guard and client refresh flow
- `/api/signin` for posting `email`, `returnUrl`, `verifyUrl`, and optional
  `scope`
- `/verify-email` GET for previewing the email address and issuing a short-lived
  CSRF cookie with `MAGICSSO_PREVIEW_SECRET`
- `/verify-email` POST for exchanging the email token for the auth cookie after
  confirmation and verifying the returned access token against the app origin
  and Magic Link SSO server origin
- `/logout` as a POST-only route that clears the cookie after a same-origin
  mutation check

In Angular 21 SSR development, prefer the modern `AngularNodeAppEngine` style
instead of manually reading `index.server.html`.

## Protected-route pattern

```ts
import type { Routes } from '@angular/router';
import { magicSsoAuthGuard } from './magic-sso';

export const routes: Routes = [
    {
        path: 'protected',
        canActivate: [magicSsoAuthGuard],
        loadComponent: () =>
            import('./protected-page.component').then(
                (value) => value.ProtectedPageComponent,
            ),
    },
];
```

## Troubleshooting

### Angular dev server warns about `reqHandler`

Make sure the SSR server exports a named handler:

- `export const reqHandler = createNodeRequestHandler(app)`
- `export default reqHandler`
- `export { AngularAppEngine } from '@angular/ssr'`

### Angular SSR tries to read `index.server.html` and fails

Prefer `AngularNodeAppEngine` plus `writeResponseToNodeResponse(...)` instead of
manually wiring `CommonEngine` against a hardcoded `index.server.html` path.

### Protected route keeps redirecting back to login

Check these first:

- `MAGICSSO_COOKIE_NAME` matches server `[cookie].name`
- `MAGICSSO_JWT_SECRET` matches server `[auth].jwtSecret`
- `MAGICSSO_PREVIEW_SECRET` matches server `[auth].previewSecret`
- the local `/verify-email` route actually sets the auth cookie
- the local `/verify-email` route verifies the returned token with the current
  app origin and SSO issuer before setting the cookie
- `/api/session` returns the verified payload instead of always returning `null`

### Sign-in form says it failed to send the verification email

Common causes:

- `MAGICSSO_SERVER_URL` is missing
- `MAGICSSO_SERVER_URL` points at the Angular app itself instead of the SSO
  server
- the app origin is missing from the selected `[[sites]].origins`
- the email is not granted access on that site through `allowedEmails` or
  `[[sites.accessRules]]`
- the requested `scope` is not granted for that email on that site
- the custom `verifyUrl` does not stay on the same site origin
- the SSR server route is not forwarding `email`, `returnUrl`, `verifyUrl`, and
  optional `scope`
- the app verifies the returned token without passing the Angular app origin and
  Magic Link SSO server origin

### Cookie appears missing in the browser

Check:

- the local `/verify-email` route uses `buildAuthCookieOptions(accessToken)`
- the app redirects back to the same origin that set the cookie
- production deployments use HTTPS when cookies are marked `secure`
- `MAGICSSO_COOKIE_PATH` is not narrower than the routes that need auth

### Local `.env` changes do not affect the SSR server

If local configuration appears ignored:

- confirm the SSR server imports `dotenv/config`
- confirm the `.env` file is in the app root
- restart the Angular dev server after editing env values
- for the bundled example, confirm `.env` includes
  `MAGICSSO_PREVIEW_SECRET=VERY-VERY-LONG-RANDOM-PREVIEW-SECRET`

### Hosted page customization does not change the Angular login page

That is expected. `[hostedAuth.copy]`, `[hostedAuth.branding]`, and per-site
`[sites.hostedAuth.*]` settings affect only the SSO server-hosted HTML pages.

### Token works once and then fails on replay

That is expected and desirable.

### Existing cookie stops verifying after the security upgrade

That is expected after SEC-001. Access tokens are now site-bound and must carry
`siteId`, `aud`, and `iss`, so sessions minted before that change need a fresh
sign-in.
