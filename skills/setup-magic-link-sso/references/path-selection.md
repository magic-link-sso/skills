# Path Selection

## Target Audience

This skill assumes one self-hosting operator persona. The same person may be
the end user of the private app, the administrator of access rules, and the
DevOps owner of the deployment.

## Choose the Deployment Shape First

### Choose classic mode when

- one operator-maintained TOML file is enough
- access changes can happen by editing server config directly
- you do not need a separate access-management control plane
- you want the smallest setup surface

Read next:

- `server.md`
- `shared-checklist.md`

### Choose Gate when

- the protected resource is static, CSR-only, or in an unsupported stack
- the upstream should stay unaware of Magic Link SSO
- you can put a reverse proxy in front of the upstream
- you need protected access to the optional manager UI

Read next:

- `server.md`
- `gate.md`
- `shared-checklist.md`

### Choose managed mode when

- you want the optional manager to own access grants and scope catalogs for
  selected sites
- you want CLI, API, or UI workflows for access administration
- you still want operators to keep base server config under manual control

Managed mode still requires the Magic Link SSO server. The manager is additive,
not a replacement for the server.

Read next:

- `server.md`
- `manager.md`
- `gate.md` when the manager UI will be Gate-protected
- `shared-checklist.md`

## Local Evaluation vs Production-Like

### Local evaluation

Prefer the repository's documented local flows when the goal is to learn the
product, validate secrets and redirect shapes, or prove the first sign-in path.

- server only: run the server from Docker or source with one local `[[sites]]`
  entry
- Gate: use the local Gate examples and Mailpit-backed email flow
- managed mode: use the manager-focused local stack to inspect generated runtime
  files and the Gate-protected manager flow

### Production-like deployment

Prefer the production-oriented docs when the goal is a real host layout, TLS,
private upstream networking, durable secrets, and repeatable operations.

- classic mode: deploy the server with a real config file and real SMTP
- Gate: keep the upstream private and expose only Gate publicly
- managed mode: keep file ownership boundaries explicit and treat the manager as
  an internal admin surface, preferably with the published-image
  `manager/docker-compose.prod.yml` stack that exposes only Gate publicly

## Hand-off Rule

If the operator also needs an app to integrate directly with Magic Link SSO,
stop after the infrastructure decision and switch to
`implement-magic-link-sso`.
