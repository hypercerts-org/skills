# Contributing

Before adding a skill to the Hypercerts catalog, verify the source skill is portable, focused, and installable.

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

1. Add it to the table in `skills/hypercerts/references/skill-map.md`.
2. Add the source link to the included pointers in `README.md`.
3. Update `skills/hypercerts/SKILL.md` only if the catalog description needs a new trigger category.

Keep `skills/hypercerts/references/install.md` generic. Skill-specific install commands and fallback URLs belong in the skill map.

## Validation

Before publishing a catalog update, validate the local skill and confirm it is discoverable:

```bash
npx skills-ref validate ./skills/hypercerts
npx skills add . --list
```

The validator should report `Valid skill: ./skills/hypercerts`, and the list command should show the `hypercerts` skill.
