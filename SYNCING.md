# Syncing this fork with upstream

This fork keeps two small commits on top of upstream (`dmtrKovalenko/fff`):

- `perf(pi-fff): shorten tool descriptions to cut per-turn prompt tokens` — trims the pi-fff tool schema/guidelines sent to the model on every turn
- `pi: declare fff runtime deps at repo root for git-based pi install` — mostly merged upstream already; only the `dependencies` block remains

Upstream edits the same files (notably `packages/pi-fff/src/index.ts`), so a plain sync always shows conflicts. Use the rebase + rerere workflow below; it makes recurring conflicts resolve themselves.

## One-time setup

```bash
git remote add upstream https://github.com/dmtrKovalenko/fff
git config rerere.enabled true
git config rerere.autoUpdate true
```

`rerere` records conflict resolutions in `.git/rr-cache` and re-applies them when the same hunks clash again — which they will, on every sync of `index.ts`.

## Sync

```bash
git fetch upstream
git rebase upstream/main        # replay this fork's commits on top of upstream
# resolve any first-time conflicts; rerere handles repeats automatically
git push --force-with-lease origin main
```

Check divergence any time:

```bash
git rev-list --count upstream/main..main   # commits ahead (this fork)
git rev-list --count main..upstream/main   # commits behind (upstream)
```

## Rules

- Never use GitHub's "Sync fork" button or `git merge upstream/main` — merge commits re-create the same conflicts on every future sync.
- `--force-with-lease` is safe here: `main` only feeds this fork's own `pi install git:github.com/RaviEdho/fff`. Anyone re-installing just pulls the new history.
- When a custom commit lands upstream (via PR), drop it from this fork during the next rebase (`git rebase --drop` / skip) — divergence shrinks toward zero.
- Prefer PR-ing fork changes upstream over carrying them forever on `main`; every commit carried here is a permanent conflict surface.
- rerere is machine-local — the `.git/rr-cache` is never pushed. A fresh clone on another machine starts with an empty cache (re-enable the config there too).

## Local development gotchas

- **Never run `bun install` / `npm install` inside `packages/`.** Bun links the workspace
  packages (`@ff-labs/fff-node`, `@ff-labs/fff-bun`) into `packages/pi-fff/node_modules`
  as symlinks to the unbuilt source tree. Node resolution then finds those symlinks
  before the root `node_modules` (where `pi install git:...` runs `npm install` and gets
  the published, built packages) and the pi session fails with
  `Cannot find module .../fff-node/dist/index.cjs`.
- If it already happened, fix with: `rm -rf packages/pi-fff/node_modules/@ff-labs`
  (the rest of that directory — `tsc`, `@types`, etc. — can stay).
- The only supported dependency install for this repo is `npm install` at the repo root,
  which pi also runs automatically when it installs or reconciles the package.