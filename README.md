# Hypercerts Skills

Portable Agent Skills entrypoint for Hypercerts developer workflows.

This repository currently ships one meta-skill:

- [`hypercerts`](skills/hypercerts/SKILL.md) — a catalog skill that helps agents discover and install focused Hypercerts skills from their source repositories.

## Install

List available skills in this repository:

```bash
npx skills add kzoeps/meta-skills-test --list
```

Install the Hypercerts meta-skill:

```bash
npx skills add kzoeps/meta-skills-test --skill hypercerts
```

For local testing from this checkout:

```bash
npx skills add . --list
```

## Included pointers

The meta-skill points to:

- `epds-login` from [`hypercerts-org/ePDS`](https://github.com/hypercerts-org/ePDS/tree/main/.agents/skills/epds-login)
- `hyperindex` from [`GainForest/hyperindex`](https://github.com/GainForest/hyperindex/tree/main/.agents/skills/hyperindex)
- `building-with-hypercerts-lexicons` from [`hypercerts-org/hypercerts-lexicon`](https://github.com/hypercerts-org/hypercerts-lexicon/tree/main/.agents/skills/building-with-hypercerts-lexicons)

## Design

This is intentionally a thin catalog. Agent Skills do not define a portable mechanism for one skill to invoke another skill, so this repo documents how to install the focused skills and lets each installed skill activate through its own `description`.
