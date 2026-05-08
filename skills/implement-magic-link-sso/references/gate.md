# Magic Link SSO Gate

Use Magic Link SSO Gate when the protected resource should not integrate with a
framework-specific Magic Link SSO package directly.

This path is the default recommendation when:

- the app is fully CSR
- the app is a static site or plain asset server
- the framework is unknown or unsupported by this skill
- the app should stay completely unaware of the auth flow
- you can place a programmable reverse proxy in front of the upstream

## Decision Rule

Prefer Gate over a framework-native integration when the protected app cannot
reliably own:

- the login route
- the verify callback
- the auth cookie lifecycle
- server-side session checks before protected content is served

Instead, put Gate in front of the upstream and keep the upstream private.

## What Gate Owns

Gate is responsible for:

- redirecting anonymous users to hosted sign-in or its own login handoff
- receiving the verify callback at `/_magicgate/verify-email`
- exchanging the one-time token with the Magic Link SSO server
- validating the returned JWT locally with `aud` and `iss`
- setting its own `httpOnly` auth cookie
- proxying HTML, assets, API calls, SSE, and websocket upgrades after auth

The upstream does not need to know about Magic Link SSO.

## Default Production Shape

- `sso.example.com` -> Magic Link SSO server
- `private.example.com` -> Magic Link SSO Gate
- `private-upstream.internal` -> protected upstream reachable only from Gate

## What To Read Next

For implementation or operator guidance, use:

- [docs/gate.md](https://github.com/magic-link-sso/magic-sso/blob/main/docs/gate.md)
- [gate/README.md](https://github.com/magic-link-sso/magic-sso/blob/main/gate/README.md)

For repository examples, see:

- [examples/gate-private1-app](https://github.com/magic-link-sso/magic-sso/tree/main/examples/gate-private1-app)
  for a dynamic upstream
- [examples/gate-private2-static](https://github.com/magic-link-sso/magic-sso/tree/main/examples/gate-private2-static)
  for a static site
