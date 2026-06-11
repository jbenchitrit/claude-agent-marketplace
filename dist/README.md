# Packaged Skill Bundles

This directory contains `.skill` packages built by CI from the `skills/` source folders.

These bundles are generated automatically on every push to `main` by the GitHub Actions workflow at `.github/workflows/package-skills.yml`.

**Do not edit files in this directory manually.** They are overwritten on each CI run.

## Bundle format

Each `.skill` file is a zip archive containing:
- `SKILL.md` — the skill definition
- `references/` — any reference files
- `manifest.json` — metadata extracted from `catalog.json`

## Installing a bundle

```bash
# Unzip directly into ~/.claude/skills/
unzip dist/volatility-design.skill -d ~/.claude/skills/volatility-design/
```
