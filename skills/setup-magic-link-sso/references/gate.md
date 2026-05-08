# Magic Link SSO Gate

## When Gate Is the Right Fit

Use Gate when the protected resource should not integrate with a framework
package directly.

Gate is usually the right choice when:

- the upstream is static HTML or assets
- the upstream is CSR-only
- the stack is unsupported or intentionally auth-unaware
- you can keep the upstream private behind a reverse proxy

Gate can also protect the optional manager UI.

## Required Server-Side Setup

Gate still depends on a normal Magic Link SSO server and a matching `[[sites]]`
entry for the Gate public origin.

The matching server site must include:

- the Gate public origin in `origins`
- the Gate verify callback in `allowedRedirectUris`
- the allowed user emails or scoped access rules

Typical callback shape:

- `https://private.example.com/_magicgate/verify-email`

## Gate Config Essentials

The Gate TOML must align:

- `[gate]`
  - `mode`
  - `publicOrigin`
  - `upstreamUrl`
  - `namespace`
  - `directUse`
  - rate limits and timeouts
- `[auth]`
  - `serverUrl`
  - `jwtSecret`
  - `previewSecret` when Gate previews verification tokens before exchange
- `[cookie]`
  - cookie name
  - cookie path
  - optional max age

Useful starting points in the main repository:

- [docs/gate.md](https://github.com/magic-link-sso/magic-sso/blob/main/docs/gate.md)
- [gate/README.md](https://github.com/magic-link-sso/magic-sso/blob/main/gate/README.md)
- [gate/magic-gate.example.toml](https://github.com/magic-link-sso/magic-sso/blob/main/gate/magic-gate.example.toml)

## Subdomain vs Path Prefix

### Prefer subdomain mode when

- you control a dedicated host such as `private.example.com`
- the upstream assumes it runs at `/`
- you want the simplest cookie and asset behavior

### Use path-prefix mode only when

- the public host must be shared
- the upstream can run under a base path
- SPA routes, assets, SSE, and websocket endpoints all tolerate the prefix

If the upstream cannot reliably run under a prefix, use a subdomain instead.

## Private-Upstream Rule

Do not expose the upstream publicly. The protection only holds when all traffic
must pass through Gate first.

## First Verification Path

Always validate this sequence:

1. Anonymous document requests are redirected by Gate.
2. The magic-link email is sent through the server flow.
3. Gate receives the verify callback at `/_magicgate/verify-email`.
4. Gate validates the returned JWT and sets its own auth cookie.
5. Authenticated traffic reaches the private upstream.

Also verify static assets, SPA navigation, and any websocket or SSE traffic the
upstream depends on.
