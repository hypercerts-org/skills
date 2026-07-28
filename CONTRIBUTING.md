# Contributing

Before adding a skill to the Hypercerts catalog, verify the source skill is portable, focused, and installable.

App or service specific focused skills should not be vendored or copied into this repository; add catalog pointers to their source repository only.

Generic skills that are not unique to any specific app (protocol-level, e.g. `atproto-oauth`) may be hosted directly in this repository under `.agents/skills/<name>/`. Only host a skill here if it would apply the same way regardless of which app or repo is using it — if the skill is tied to a particular app's architecture or codebase, it belongs in that app's own repository as a pointer instead.

## Required checks

- The skill directory contains a `SKILL.md` file.
- `SKILL.md` has YAML frontmatter.
- Frontmatter includes `name` and `description`.
- `name` matches the parent directory name.
- `name` uses only lowercase letters, numbers, and hyphens.
- `name` does not start or end with a hyphen.
- `name` does not contain consecutive hyphens.
- `description` explains both what the skill does and when to use it.
- The skill does not rely on another skill being invoked by name.

## Discovery checks

List skills in the source repository:

```bash
npx skills add <owner/repo> --list
```

Confirm the specific skill can be installed:

```bash
npx skills add <owner/repo> --skill <skill-name> --yes
```

If repository-wide discovery fails, confirm the direct-directory path works:

```bash
npx skills add https://github.com/<owner>/<repo>/tree/main/<path-to-skill> --yes
```

## Catalog update

When a new skill passes the checks:

1. Add it to the table in `skills/hypercerts/references/skill-map.md`. For a locally hosted skill, use `npx skills add hypercerts-org/skills --skill <skill-name> --yes` as the source repository install command and `https://github.com/hypercerts-org/skills/tree/main/.agents/skills/<skill-name>` as the direct-directory fallback.
2. Add the source link to `README.md` — under "Included skills (hosted here)" for locally hosted skills, or "Included pointers (hosted elsewhere)" for skills in another repository.
3. Update `skills/hypercerts/SKILL.md` only if the catalog description needs a new trigger category.

Keep `skills/hypercerts/references/install.md` generic. Skill-specific install commands and fallback URLs belong in the skill map.

## Validation

Before publishing a catalog update, validate the local skill and confirm it is discoverable:

```bash
npx skills-ref validate ./skills/hypercerts
npx skills add . --list
```

The validator should report `Valid skill: ./skills/hypercerts`, and the list command should show the `hypercerts` skill.
