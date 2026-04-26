# Next.js

## Access assumptions

The package is published to npm as `@magic-link-sso/nextjs` and can be installed
from the public npm registry:

```sh
npm install @magic-link-sso/nextjs
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

Important alignments:

- `MAGICSSO_COOKIE_NAME` must match server `[cookie].name`.
- `MAGICSSO_JWT_SECRET` must match server `[auth].jwtSecret`.
- `MAGICSSO_PREVIEW_SECRET` must match server `[auth].previewSecret` when the
  app owns `/verify-email`.
- `MAGICSSO_COOKIE_MAX_AGE` should not exceed the effective JWT lifetime.
- `MAGICSSO_SERVER_URL` must point at the reachable SSO server origin.

## Required server site config

The app origin must exist in the matching `[[sites]].origins` entry inside the
Magic Link SSO server TOML config.

The site must grant email access through `allowedEmails` or
`[[sites.accessRules]]`.

If the app requests a non-default `scope`, that scope must be granted for the
email on that same site.

When the app sends a custom `verifyUrl`, keep it same-origin with the selected
site.

## Integration patterns

### Direct hosted sign-in

- Set `MAGICSSO_DIRECT_USE=true`.
- Protect routes with `authMiddleware`.
- Let middleware build the redirect to `${MAGICSSO_SERVER_URL}/signin`.
- Keep a local `/verify-email` only if the app still wants to own the final
  token exchange and cookie issuance.

### Local login page and local verify callback

- Set `MAGICSSO_DIRECT_USE=false`.
- Build a `/login` page that posts `email`, `returnUrl`, and `verifyUrl`.
- Build a local `/verify-email` route pair that previews the email address on
  GET, then exchanges the token on POST after a short-lived CSRF check.
- Use a custom login action when the email link must return to the app first.

## Package exports and when to use them

- `authMiddleware(request, options?)` Use in `proxy.ts` to protect routes.
- `buildLoginUrl(request, pathname, scope?)` Use when code needs the exact login
  redirect target without executing the full middleware.
- `verifyToken()` Use in server components and route handlers to read and verify
  the auth cookie. It derives the expected audience from the current request
  origin and the expected issuer from `MAGICSSO_SERVER_URL`.
- `verifyAuthToken(token, secret, options)` Use when code already has the raw
  cookie value and secret bytes. Pass `expectedAudience` and, when available,
  `expectedIssuer`.
- `redirectToLogin(returnUrl, scope?)` Use in server components after a failed
  auth check.
- `buildAuthCookieOptions(value)` Use when a local route sets the auth cookie
  after the verify-token exchange.
- `sendMagicLink(email, returnUrl, scope?)` Use for the basic flow where the
  server can handle the default verification path and optional scope request.
  Use a custom action instead when the app needs to send `verifyUrl` and own the
  callback.
- `LogoutRoute(request)` Use for a standard POST-only logout route that clears
  the auth cookie, enforces same-origin mutation checks, and redirects home.

## Local verify-email responsibilities

The local `/verify-email` route should:

1. Read the email verification token from the query string.
2. Normalize or reject unsafe `returnUrl` values.
3. Call the SSO server `GET /verify-email?token=...` endpoint with
   `accept: application/json` and `X-Magic-SSO-Preview-Secret` set from
   `MAGICSSO_PREVIEW_SECRET`.
4. Render a confirmation form and bind it to a short-lived CSRF cookie or
   equivalent server-only state.
5. On POST, validate the CSRF token, then call the SSO server
   `POST /verify-email` endpoint with `accept: application/json` and a JSON body
   containing `token`.
6. Validate that the response contains `accessToken`, verify it locally, set the
   auth cookie with `buildAuthCookieOptions`, and redirect to the normalized
   destination. Local verification must check that the token carries `email`,
   `scope`, `siteId`, `aud`, and `iss`, with `aud` matching the app origin and
   `iss` matching the Magic Link SSO server origin.

## Protected-page pattern

In App Router server components:

```ts
import { redirectToLogin, verifyToken } from '@magic-link-sso/nextjs';

export default async function ProtectedPage(): Promise<JSX.Element | null> {
    const auth = await verifyToken();
    if (auth === null) {
        redirectToLogin('/protected');
    }

    return (
        <main>
            Signed in as {auth.email} on site {auth.siteId} with scope {auth.scope}
        </main>
    );
}
```

## Troubleshooting

### Redirect loop back to login

Check these first:

- `MAGICSSO_COOKIE_NAME` matches server `[cookie].name`.
- `MAGICSSO_JWT_SECRET` matches server `[auth].jwtSecret`.
- `MAGICSSO_SERVER_URL` points at the same Magic Link SSO origin that issued the
  token.
- `proxy.ts` and `authMiddleware` both exclude `/login`, `/logout`, and
  `/verify-email`.
- The app actually sets the auth cookie in the local `/verify-email` route.

### Magic link email sends, but callback fails

Common causes:

- the app origin is missing from the selected `[[sites]].origins`.
- the email is not granted access on that site through `allowedEmails` or
  `[[sites.accessRules]]`.
- the requested `scope` is not granted for that email on that site.
- the custom `verifyUrl` does not stay on the same site origin.
- the login action sends a malformed `verifyUrl`.
- the local `/verify-email` route does not request JSON from the server.
- `MAGICSSO_PREVIEW_SECRET` does not match server `[auth].previewSecret`.
- the app verifies the returned token without passing the current app origin and
  Magic Link SSO server origin.

### Cookie appears missing in the browser

Check:

- the local `/verify-email` route calls `buildAuthCookieOptions(accessToken)`.
- the app is redirecting to the same origin that set the cookie.
- production deployments use HTTPS when cookies are marked `secure`.
- the browser is not landing on a different subdomain than the one expected by
  the cookie settings.

### Package helper is too limited for local callback flow

That is expected. The package helper `sendMagicLink(email, returnUrl, scope?)`
is enough for the basic server-driven flow. Use a custom login action whenever
the app needs to send `verifyUrl` alongside the form submission and own the
`/verify-email` route.

### Logout does not clear the cookie

Check:

- the logout route is using the same cookie name as the auth route.
- the route is reachable without auth redirects.
- the app is not setting a second cookie under a different name or domain.

### Hosted page customization does not affect the Next.js UI

That is expected. `[hostedAuth.copy]`, `[hostedAuth.branding]`, and per-site
`[sites.hostedAuth.*]` settings only affect the server-hosted HTML pages.

### Token works once and then fails on replay

That is expected and desirable.

### Older session cookie stops working after upgrading

That is expected after SEC-001. Access tokens are now site-bound and must carry
`siteId`, `aud`, and `iss`, so older cookies minted before that change no longer
verify and users need to sign in again.
