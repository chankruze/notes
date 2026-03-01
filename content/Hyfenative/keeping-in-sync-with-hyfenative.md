---
title: Keeping the project in sync with hyfenative template
tags:
  - react-native
created: 2026-03-01
updated:
status: draft
---
## Create App Repo From Template

Inside `app-one`:

```bash
git clone https://github.com/chankruze/hyfenative.git app-name
cd app-name
git remote remove origin
git remote add origin <new-app-repo>
git push -u origin main
```

Now check remotes:

```bash
git remote -v

# should have similar to this
# origin    -> app-one
# upstream  -> hyfenative
```

## Keep Boilerplate Changes in Separate Branch

⚠️ Important discipline rule.

Create a branch dedicated to syncing boilerplate:

```bash
git checkout -b boilerplate-sync
```

## When We Update hyfenative

In `app-one`:

```bash
git fetch upstream
git checkout boilerplate-sync
git merge upstream/main
```

Fix conflicts (if any).

Then merge into your main branch:

```bash
git checkout main
git merge boilerplate-sync
```

## Golden Rule (Very Important)

Inside `app-one`, try to keep these untouched:

- Navigation root setup
- API client core
- Theme system
- Config layer
- Folder structure
### Avoid modifying:

- Shared utilities
- Core infra logic
- Base components

Instead, create:

```
src/features/
```

For all app-specific logic.

This reduces merge conflicts massively.

## Use Rebase Instead of Merge (Cleaner History)

Instead of:

```bash
git merge upstream/main
```

You can:

```bash
git rebase upstream/main
```

But only if you're comfortable resolving rebase conflicts.

## Pro Tip (Very Useful)

Tag stable boilerplate versions.

In `hyfenative`:

```bash
git tag v1.0.0
git push origin v1.0.0
```

Then in app:

```bash
git merge v1.0.0
```

This prevents unexpected breaking changes.