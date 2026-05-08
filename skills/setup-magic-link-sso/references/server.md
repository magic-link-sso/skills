# Magic Link SSO Server

## What the Server Owns

The server sends magic-link emails, previews and exchanges verification tokens,
issues JWT access tokens, and hosts the default sign-in and verification pages.

## Minimal Bootstrap

For the fastest standalone start:

1. Create a Magic Link SSO TOML file from the server example.
2. Set `MAGICSSO_CONFIG_FILE` to that file path.
3. Start the server from Docker or from source.
4. Verify `GET /healthz`.

Useful starting points in the main repository:

- [Repository README](https://github.com/magic-link-sso/magic-sso/blob/main/README.md)
  for Docker and source bootstrap
- [server/magic-sso.example.toml](https://github.com/magic-link-sso/magic-sso/blob/main/server/magic-sso.example.toml)
  for the full config shape
- [server/docker-compose.yml](https://github.com/magic-link-sso/magic-sso/blob/main/server/docker-compose.yml)
  for the server container example

## Required Config Areas

The first working server config must align these sections:

- `[server]`
  - `appUrl`
  - `appPort`
  - `logLevel`
  - `logFormat`
- `[auth]`
  - `jwtSecret`
  - `csrfSecret`
  - `emailSecret`
  - `previewSecret`
  - expiration settings
- `[cookie]`
  - cookie name and browser policy
- `[email]`
  - sender details
  - SMTP transport settings
- `[[sites]]`
  - `id`
  - `origins`
  - `allowedRedirectUris`
  - `allowedEmails` or `[[sites.accessRules]]`

## Expectations Before First Sign-In

- Secrets are long, random, and distinct from one another.
- SMTP is reachable and can send the magic-link email.
- The target app or protected origin appears in the matching `[[sites]]` entry.
- Redirect URIs cover the intended verify callback or protected return path.
- Access is granted either by `allowedEmails` for full access or by
  `[[sites.accessRules]]` for scoped access.

## Local Evaluation

For a quick local proof:

- use one localhost origin in `[[sites]]`
- use Mailpit or another reachable SMTP sink
- confirm the server starts and logs the loaded config without validation errors

## Production-Like Notes

- serve the public origin over HTTPS
- keep the config file outside ephemeral container state
- use real SMTP and durable secrets
- choose the server security-state adapter that matches single-node or
  multi-node deployment
- treat the optional reload endpoint as private internal infrastructure only

## First Verification Path

Always validate this sequence:

1. `GET /healthz` succeeds.
2. The hosted sign-in page renders.
3. A magic-link email is sent.
4. The verify flow returns a JWT-backed session.
5. The user lands on the trusted return URL.

If the next problem is framework-specific wiring, switch to
`implement-magic-link-sso`.
