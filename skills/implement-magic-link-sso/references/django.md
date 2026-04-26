# Django

## Access assumptions

The package is published to PyPI as `magic-link-sso-django` and can be installed
from the public Python Package Index:

```sh
pip install magic-link-sso-django
```

## Required settings

```python
MAGICSSO_SERVER_URL = 'http://localhost:3000'
MAGICSSO_JWT_SECRET = 'your_jwt_secret'
MAGICSSO_PREVIEW_SECRET = 'your_preview_secret'
MAGICSSO_COOKIE_NAME = 'magic-sso'
```

Common optional settings:

```python
MAGICSSO_DIRECT_USE = False
MAGICSSO_COOKIE_DOMAIN = None
MAGICSSO_COOKIE_MAX_AGE = None
MAGICSSO_COOKIE_PATH = '/'
MAGICSSO_COOKIE_SAMESITE = 'Lax'
MAGICSSO_COOKIE_SECURE = True
MAGICSSO_AUTH_EVERYWHERE = False
MAGICSSO_PUBLIC_URLS = ['login']
MAGICSSO_REQUEST_TIMEOUT = 5
```

Important alignments:

- `MAGICSSO_COOKIE_NAME` must match server `[cookie].name`.
- `MAGICSSO_JWT_SECRET` must match server `[auth].jwtSecret`.
- `MAGICSSO_PREVIEW_SECRET` must match server `[auth].previewSecret`.
- `MAGICSSO_COOKIE_MAX_AGE` should not exceed the effective JWT lifetime.
- `MAGICSSO_COOKIE_SAMESITE` must be one of `'Lax'`, `'Strict'`, or `'None'`.

## Required server site config

The Django origin must exist in the matching `[[sites]].origins` entry inside
the Magic Link SSO server TOML config.

The site must grant email access through `allowedEmails` or
`[[sites.accessRules]]`.

If the login flow requests a non-default `scope`, that scope must be granted for
the email on that same site.

Because the package-generated `verifyUrl` stays on the Django origin, keep that
origin aligned with the selected site.

## What the package provides

Wiring in `magic_sso_django` gives the app:

- middleware `MagicSsoMiddleware`
- helper functions `is_authenticated()` and `redirect_to_login()`
- decorator `sso_login_required`
- package views `login`, `logout`, and `verify_token`
- packaged URLconf for those views

## Protection patterns

- `@sso_login_required` on specific views
- manual checks with `is_authenticated(request)` and
  `redirect_to_login(request)`
- `MAGICSSO_AUTH_EVERYWHERE = True` for middleware-driven global auth

`MagicSsoMiddleware` always sets:

- `request.is_magic_sso_authenticated`
- `request.magic_sso_user_email`
- `request.magic_sso_user_scope`

When app code needs the full verified payload, call `is_authenticated(request)`.
Its payload now includes `siteId` in addition to `email` and `scope`.

## Package login-view behavior

`magic_sso_django.views.login` supports two modes:

- `MAGICSSO_DIRECT_USE=True` redirect to `${MAGICSSO_SERVER_URL}/signin` with
  `returnUrl`, generated `verifyUrl`, and optional requested `scope`
- `MAGICSSO_DIRECT_USE=False` render the package login template and on POST send
  `email`, `returnUrl`, generated `verifyUrl`, and optional `scope` to the SSO
  server

`verify_token` is a two-step local callback:

- on `GET`, it reads `token` and `returnUrl`, calls
  `${MAGICSSO_SERVER_URL}/verify-email` with `MAGICSSO_PREVIEW_SECRET`, and
  renders the packaged confirmation page
- on `POST`, it exchanges the token at `${MAGICSSO_SERVER_URL}/verify-email`,
  verifies the returned access token against the Django request origin and the
  Magic Link SSO server origin, sets the auth cookie, and redirects to the
  normalized destination
- `logout` is POST-only and should run with Django CSRF protection enabled, so
  consuming templates should submit it with `{% csrf_token %}`

`is_authenticated(request)` now relies on the same site-bound verification. A
valid auth cookie must include `email`, `scope`, `siteId`, `aud`, and `iss`.

## Troubleshooting

### Redirect loop or unexpected public-page redirect

Check these first:

- `MAGICSSO_COOKIE_NAME` matches server `[cookie].name`.
- `MAGICSSO_JWT_SECRET` matches server `[auth].jwtSecret`.
- `MAGICSSO_PREVIEW_SECRET` matches server `[auth].previewSecret`.
- `MAGICSSO_SERVER_URL` points at the same Magic Link SSO origin that issued the
  token.
- `MAGICSSO_PUBLIC_URLS` includes every public Django URL name that should
  bypass auth middleware.
- `MAGICSSO_AUTH_EVERYWHERE` is set intentionally.

### Magic link email sends, but callback fails

Common causes:

- the Django origin is missing from the selected `[[sites]].origins`.
- the email is not granted access on that site through `allowedEmails` or
  `[[sites.accessRules]]`.
- the requested `scope` is not granted for that email on that site.
- `MAGICSSO_SERVER_URL` points at the wrong server.
- `MAGICSSO_PREVIEW_SECRET` does not match server `[auth].previewSecret`.
- `MAGICSSO_REQUEST_TIMEOUT` is too small for the environment.
- the returned token issuer or audience does not match the Django request origin
  and configured Magic Link SSO server.

### Cookie appears missing or auth never sticks

Check:

- `MAGICSSO_COOKIE_NAME` matches the cookie actually set by Django
- `MAGICSSO_COOKIE_SECURE` matches the deployment protocol
- `MAGICSSO_COOKIE_DOMAIN` is appropriate for the host being served
- `MAGICSSO_COOKIE_SAMESITE` is a valid value

### Decorator-protected view does not see the user email

Check whether the view is reading `request.magic_sso_user_email` after the
decorator or middleware has run.

### Global auth blocks too much of the site

This usually means `MAGICSSO_AUTH_EVERYWHERE=True` is correct but
`MAGICSSO_PUBLIC_URLS` is incomplete.

### Hosted page customization does not change the Django login template

That is expected. `[hostedAuth.copy]`, `[hostedAuth.branding]`, and per-site
`[sites.hostedAuth.*]` settings affect only the SSO server-hosted HTML pages.

### Token works once and then fails on replay

That is expected and desirable.

### Existing session cookie stops working after upgrade

That is expected after SEC-001. Access tokens are now site-bound and older
cookies without `siteId`, `aud`, and `iss` are rejected until the user signs in
again.
