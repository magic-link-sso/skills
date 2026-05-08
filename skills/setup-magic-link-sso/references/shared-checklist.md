# Shared Checklist

## Confirm the Chosen Path

- classic mode, Gate, and managed mode are not being mixed accidentally
- the operator knows whether this is a local evaluation or a production-like
  deployment
- any app-integration work has been separated from infrastructure setup

## Align These Settings

- server `[auth].jwtSecret`
- server `[auth].previewSecret` when Gate or an app previews verification
  tokens
- server `[cookie].name`
- the matching `[[sites]]` entry for each public origin
- redirect URIs for hosted verify callbacks or Gate verify callbacks
- Gate `publicOrigin` and `upstreamUrl`
- manager `managedSiteIds` and bootstrap `manager-admin` site boundaries

If these are misaligned, the setup may appear close to working while still
failing with redirects, forbidden responses, missing sessions, or invalid-token
errors.

## TLS and Network Expectations

- serve public browser-facing origins over HTTPS in production-like deployments
- keep private upstreams unreachable except through Gate
- keep the manager service and server reload endpoint on private networking
- decide whether proxy trust settings match the real deployment path

## Works Verification

### Server

- `GET /healthz` succeeds
- hosted sign-in renders
- email delivery works
- the verify flow returns a working session

### Gate

- anonymous traffic is blocked or redirected before the upstream is exposed
- the Gate verify callback succeeds
- authenticated traffic reaches the upstream
- logout clears the Gate session

### Manager

- validate and diff succeed before apply
- apply writes the intended runtime config
- the server accepts that runtime config
- the bootstrap manager admin path still works after apply

## Common Failure Points

- wrong server origin or Gate public origin
- wrong redirect URI shape
- cookie name mismatch
- JWT or preview secret mismatch
- SMTP not actually delivering
- Gate upstream reachable publicly
- manager trying to own the bootstrap manager-admin site
- base config, runtime config, and manager state drifting apart

## Hand-off Reminder

If the setup is correct but the protected app still needs route wiring, cookie
verification helpers, or framework-specific callback handling, switch to
`implement-magic-link-sso`.
