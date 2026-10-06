# Git Instructions (bitwarden/clients)

## Create and push a new branch

```bash
cd ~/Bitwarden/clients

# 1. Start from an up-to-date main
git checkout main
git pull origin main

# 2. Create the branch (team format: ac/pm-<ticket>-<short-description>)
git checkout -b ac/pm-12345-short-description

# 3. Push it and link it to GitHub (-u is only needed the first time)
git push -u origin ac/pm-12345-short-description
```

## After that, on the same branch

```bash
git add <files>
git commit -m "your message"
git push            # no branch name needed, -u already linked it
```

## Bring your branch up to date with main

```bash
git fetch origin main
git rebase origin/main
git push --force-with-lease   # only needed if the branch was already pushed
```

## Squash

```bash
git fetch --all && git checkout main && git reset --hard origin/main
git checkout ac/pm-44178/providers-guard-spec-service-ts-strict
git reset $(git merge-base origin/main $(git rev-parse --abbrev-ref HEAD))
```
