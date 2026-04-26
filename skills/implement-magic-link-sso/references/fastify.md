# Fastify

## Intended shape

The intended Fastify integration is:

- plain Node + Fastify, not a meta-framework
- server-rendered HTML or route-owned responses
- same-origin cookie based
- app-owned at the route layer
- visually owned by the consuming app

## Route expectations

The integration usually exposes:

- `GET /`
- `GET /login`
- `POST /api/signin`
- `GET /verify-email`
- `GET /protected`
- `POST /logout`

## Required env vars

```env
MAGICSSO_COOKIE_NAME=magic-sso
MAGICSSO_COOKIE_MAX_AGE=3600
MAGICSSO_DIRECT_USE=false
MAGICSSO_JWT_SECRET=your_jwt_secret
MAGICSSO_PREVIEW_SECRET=your_preview_secret
MAGICSSO_SERVER_URL=http://localhost:3000
```

Optional:

```env
MAGICSSO_COOKIE_PATH=/
PORT=3005
```

Keep these aligned with the server:

- `MAGICSSO_COOKIE_NAME` must match server `[cookie].name`.
- `MAGICSSO_JWT_SECRET` must match server `[auth].jwtSecret`.
- `MAGICSSO_PREVIEW_SECRET` must match server `[auth].previewSecret`.
- The app origin must exist in the matching `[[sites]].origins`.
- The site must authorize the email through `allowedEmails` or
  `[[sites.accessRules]]`.
- If the app requests a non-default `scope`, that scope must be granted for the
  email on that site.
- If the app sends a custom `verifyUrl`, keep it same-origin with the selected
  site.

## Local sign-in flow

1. Normalize `returnUrl` against the app origin.
2. Build a local `verifyUrl` that points to `/verify-email`.
3. POST `email`, `returnUrl`, `verifyUrl`, and optional `scope` to
   `${MAGICSSO_SERVER_URL}/signin`.
4. Show a success or failure message on the local login page.

## Verify-email responsibilities

The Fastify callback must:

1. Read `token` and `returnUrl` from the query string.
2. Normalize `returnUrl` so only same-origin redirects are allowed.
3. Call `${MAGICSSO_SERVER_URL}/verify-email?token=...`, request JSON, and send
   `X-Magic-SSO-Preview-Secret` from `MAGICSSO_PREVIEW_SECRET`.
4. Render a confirmation form and bind it to a short-lived CSRF cookie or
   equivalent server-only state.
5. On POST, validate the CSRF token, then call
   `${MAGICSSO_SERVER_URL}/verify-email` with a JSON body containing `token` and
   request JSON.
6. Expect a payload containing `accessToken`.
7. Verify the returned token with `MAGICSSO_JWT_SECRET`.
8. Set the auth cookie and redirect to the normalized `returnUrl`.

## Protected routes

Read the configured cookie from the request, verify the JWT, and redirect
unauthenticated requests to either the local `/login` page or the hosted
`/signin` page depending on `MAGICSSO_DIRECT_USE`.

## Logout expectations

The logout handler should:

1. Accept only `POST`.
2. Require a same-origin `Origin` or `Referer` header before mutating cookie
   state.
3. Clear the auth cookie with the same attributes used when it was set.
4. Redirect with a post-redirect-get response.

## Troubleshooting

### `MAGICSSO_SERVER_URL` points back to the Fastify app

Point `MAGICSSO_SERVER_URL` at the hosted Magic Link SSO server, usually
`http://localhost:3000` in local development.

### `MAGICSSO_JWT_SECRET` does not match the SSO server

Make sure the Fastify app and SSO server share the same JWT secret.

### Callback redirects back to login before preview

Make sure `MAGICSSO_PREVIEW_SECRET` matches server `[auth].previewSecret`.

### Redirect target is rejected

Check:

- `returnUrl` is same-origin with the Fastify app
- the app origin exists in the matching `[[sites]].origins`
- the email is granted access through `allowedEmails` or `[[sites.accessRules]]`
- any requested `scope` is granted for that email

### Cookie exists but protected route still redirects

Check:

- the cookie name matches the server config
- the cookie path still includes the protected route
- the token can be validated with the configured JWT secret
