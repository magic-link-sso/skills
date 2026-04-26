# Nuxt

## Access assumptions

The module is published to npm as `@magic-link-sso/nuxt` and can be installed
from the public npm registry:

```sh
npm install @magic-link-sso/nuxt
```

## Required config

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
- `MAGICSSO_PREVIEW_SECRET` must match server `[auth].previewSecret`.
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

## What the module provides

Registering `@magic-link-sso/nuxt` gives the app:

- `magic-sso-auth` route middleware
- `/verify-email` GET preview route and `/verify-email` POST confirmation route
- POST-only `/logout` server route
- `useMagicSsoAuth()` composable
- `useMagicSsoConfig()` composable
- server utilities exported from `@magic-link-sso/nuxt/server`

## Integration patterns

### Direct hosted sign-in

- Set `MAGICSSO_DIRECT_USE=true`.
- Protect pages with `magic-sso-auth`.
- Let middleware redirect straight to `${MAGICSSO_SERVER_URL}/signin`.
- Rely on the built-in `/verify-email` preview-plus-confirm flow and POST-only
  `/logout` route.

### Local login page with local sign-in API

- Set `MAGICSSO_DIRECT_USE=false`.
- Create a local `/login` page.
- Create a local sign-in API route that posts `email`, `returnUrl`, and
  `verifyUrl` to the SSO server.
- Forward `scope` too when the app uses scoped access.
- Reuse the built-in `/verify-email` preview-plus-confirm flow and POST-only
  `/logout` route.

## Important SSR nuance

The auth cookie is `httpOnly`, so client-side middleware cannot make a reliable
auth decision. Use full-document navigation to protected pages and let the
server-side middleware decide.

Verified auth state is now site-bound. The token must include `email`, `scope`,
`siteId`, `aud`, and `iss`.

## Local sign-in API expectations

The sign-in API route should:

- require `email`, `returnUrl`, and `verifyUrl`
- optionally accept and forward `scope`
- send them to `${serverUrl}/signin`
- return a user-facing success or failure message

## Built-in verify-email behavior

The bundled `/verify-email` route pair already handles the local callback flow:

- `GET /verify-email` reads `token` and `returnUrl`, calls
  `${MAGICSSO_SERVER_URL}/verify-email` with `MAGICSSO_PREVIEW_SECRET`, and
  renders a confirmation page
- `POST /verify-email` validates the CSRF token, exchanges the email token at
  `${MAGICSSO_SERVER_URL}/verify-email`, verifies the returned token against the
  current app origin and Magic Link SSO server origin, sets the auth cookie, and
  redirects to the normalized destination

The bundled `/logout` route is POST-only. Trigger it with a same-origin
`<form method="post" action="/logout">` so the route can reject cross-site
requests before clearing the cookie.

## Protected-page pattern

In a protected page:

```vue
<script setup lang="ts">
import type { AuthPayload } from '@magic-link-sso/nuxt';

definePageMeta({
    middleware: ['magic-sso-auth'],
});

const authState = useState<AuthPayload | null>('protected-auth', () => null);

if (import.meta.server) {
    authState.value = await useMagicSsoAuth();
}
</script>
```

## Troubleshooting

### Protected page still loads unauthenticated on client navigation

This can be expected if the app tries to decide auth entirely on the client.

### Redirect loop back to login

Check these first:

- `MAGICSSO_COOKIE_NAME` matches server `[cookie].name`.
- `MAGICSSO_JWT_SECRET` matches server `[auth].jwtSecret`.
- `MAGICSSO_PREVIEW_SECRET` matches server `[auth].previewSecret`.
- `MAGICSSO_SERVER_URL` points at the same Magic Link SSO origin that issued the
  token.
- `runtimeConfig.magicSso.excludedPaths` still includes `/login`, `/logout`,
  `/verify-email`, and static asset routes.
- The built-in or custom `/verify-email` route is actually setting the cookie.

### Magic link email sends, but callback fails

Common causes:

- the app origin is missing from the selected `[[sites]].origins`.
- the email is not granted access on that site through `allowedEmails` or
  `[[sites.accessRules]]`.
- the requested `scope` is not granted for that email on that site.
- the custom `verifyUrl` does not stay on the same site origin.
- the local sign-in API route sends a malformed `verifyUrl`.
- `MAGICSSO_SERVER_URL` is missing or points at the wrong origin.
- the app verifies the returned token without the current app origin and Magic
  Link SSO server origin.

### Cookie appears missing in the browser

Check:

- the built-in `/verify-email` route or custom route sets the cookie
- the app redirects to the same origin that issued the cookie
- production deployments use HTTPS when cookies are marked `secure`
- the browser is not landing on a different subdomain than expected

### Local login page works, but sign-in API route fails

Keep the sign-in API route small and explicit. It should validate input, post
JSON to the SSO server, and surface a structured success or failure message.

### Hosted page customization does not affect the Nuxt UI

That is expected. `[hostedAuth.copy]`, `[hostedAuth.branding]`, and per-site
`[sites.hostedAuth.*]` settings affect only the server-hosted HTML pages.

### Token works once and then fails on replay

That is expected and desirable.

### Existing cookie stops working after upgrade

That is expected after SEC-001. Access tokens are now site-bound and older
cookies without `siteId`, `aud`, and `iss` are rejected until the user signs in
again.
