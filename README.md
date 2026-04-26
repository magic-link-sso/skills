# Magic Link SSO Skills

Agent-agnostic [skills.sh](https://skills.sh/) skills for implementing,
comparing, and troubleshooting [Magic Link SSO](https://github.com/magic-link-sso/magic-sso)
across supported web stacks.

This repository is structured for the open Agent Skills ecosystem, so the same
skill pack can be installed for Codex, Claude, Gemini, and other agents that
support standard `SKILL.md`-based skills through `skills`.

## Install

Install the whole repository:

```shell
npx skills add magic-link-sso/skills
```

Install a specific skill:

```shell
npx skills add magic-link-sso/skills --skill implement-magic-link-sso
```

List available skills without installing:

```shell
npx skills add magic-link-sso/skills --list
```

## Included Skills

### `implement-magic-link-sso`

Routes an agent to the right Magic Link SSO integration path for:

- Next.js App Router
- Nuxt SSR
- Angular SSR
- Fastify
- Django
- Magic Link SSO Gate for static, CSR-only, unknown, or unsupported stacks

It acts as a dispatcher skill: detect the framework, open the matching bundled
reference, apply the shared checklist, and avoid re-deriving the integration
from scratch.

## Repo Layout

This repo follows the layout that `skills` discovers by default:

```text
skills/
  implement-magic-link-sso/
    SKILL.md
    references/
    agents/
```

Core compatibility comes from `skills/<skill-name>/SKILL.md` with YAML
frontmatter. Extra files such as bundled references or agent-specific metadata
can live alongside the skill without breaking discovery.

## About Magic Link SSO

The underlying product, server, and framework integrations live in the main
[magic-link-sso/magic-sso](https://github.com/magic-link-sso/magic-sso)
repository.
