# HOWTO — pi-config sync

How my `pi` configuration stays mirrored across machines.

## The idea in one line
`~/pi-config` is a git repo and the **single source of truth**. Selected files
under `~/.pi/agent/` are **symlinks** into it, so editing config *is* editing the
repo. `./sync.sh` reconciles everything with GitHub.

`settings.json` is the exception: it is **generated** from two files, so shared
config syncs while each machine keeps its own default model/provider:

```
~/.pi/agent/settings.json  ◄──  agent/settings.json                (shared, in repo)
                          ◄──  agent/settings.local.<hostname>.json (per machine, in repo)
```

## Everyday workflow

### Change something (prompt, settings, an extension)
Just edit the file normally — you're editing the repo through the symlink. When
you want to publish it:
```bash
~/pi-config/sync.sh
```
That commits anything uncommitted, pulls, and pushes. Run it on the other
machine to pull the change down.

Settings are split:
- **Shared** (`theme`, `packages`, ...) → edit `agent/settings.json`.
- **This machine's default model/provider** → edit
  `agent/settings.local.<hostname>.json`, or just use `/model` in pi (sync.sh
  picks it up).

> Tip: for changes worth a real history entry, commit yourself first with a good
> message, then run sync — it only auto-commits leftovers.
> ```bash
> cd ~/pi-config
> git commit -am "prompt: tighten discussion-mode rules"
> ./sync.sh
> ```

### Golden rule
- **Changed** an existing file → `sync.sh` (or just `git pull` on the other box).
- **Added** a new file/extension → `sync.sh` (it creates the new symlink).

## Adding a new extension
Create it **inside the repo**, then sync:
```bash
mkdir -p ~/pi-config/agent/extensions/myext
$EDITOR ~/pi-config/agent/extensions/myext/index.ts
~/pi-config/sync.sh          # links it into ~/.pi/agent + commits + pushes
```
Accidentally created it live in `~/.pi/agent/extensions/myext` first? Adopt it:
```bash
~/pi-config/sync.sh --adopt myext
~/pi-config/sync.sh
```

## Adding npm-package extensions
Install as usual with `pi install npm:...`. That writes the package into
`~/.pi/agent/npm/package.json` (a symlink into the repo) **and** adds it to the
`packages` list in the live `~/.pi/agent/settings.json`. Then:
```bash
~/pi-config/sync.sh          # adopts the packages change + runs npm install if deps changed
```

## Setting up a NEW machine
```bash
# 1. authenticate (HTTPS + gh; login as peq22, browser device flow)
gh auth login --hostname github.com --git-protocol https
gh auth setup-git       # wires gh's credential helper into git

# 2. clone + activate
git clone https://github.com/peq22/Pi-sync.git ~/pi-config
~/pi-config/sync.sh          # links config, creates settings.local.<hostname>.json from example

# 3. set this machine's defaults, then re-sync
$EDITOR ~/pi-config/agent/settings.local.$(hostname -s).json   # defaultProvider/model/thinking
~/pi-config/sync.sh

# 4. start pi and log in once (creates local auth.json)
```

`gh` is not installed by default on Ubuntu. One-time install (Ubuntu official repo):
```bash
sudo apt-get update
sudo apt-get install -y wget
sudo mkdir -p -m 755 /etc/apt/keyrings
wget -qO- https://cli.github.com/packages/github.key | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt-get update
sudo apt-get install -y gh
```

## What is / isn't synced
**Synced (in the repo):** `APPEND_SYSTEM.md`, `settings.json` (shared keys),
`settings.local.<hostname>.json` (each machine's defaults), `keybindings.json`,
`npm/package.json` + `package-lock.json`, `extensions/*`, `skills/*` (if any).

**Local only (gitignored, per machine):** `auth.json` (login/secrets),
`models-store.json` (cache), `bin/`, `npm/node_modules/`, `sessions/`, and the
generated `~/.pi/agent/settings.json` (rebuilt from the two files above).

You log in to pi **once per machine** — auth never leaves the machine.

## Conflicts
Per-machine defaults (`defaultProvider`/`defaultModel`/`defaultThinkingLevel`)
live in separate `settings.local.<hostname>.json` files, so they never conflict.
Conflicts now only happen if both machines edited the *same shared* file. In
that case `sync.sh` stops at the pull:
```
ERROR: pull/rebase conflict. Fix in ~/pi-config, then: git rebase --continue
```
Do exactly that:
```bash
cd ~/pi-config
# edit the conflicted file(s), then:
git add -A
git rebase --continue
./sync.sh                    # finish the sync
```

## Troubleshooting
- **Push fails with `Authentication failed` / `could not read Username`** →
  gh token missing/stale on this machine. Check `gh auth status`, re-login with
  `gh auth login --hostname github.com --git-protocol https`, then `gh auth setup-git`.
- **`gh` not found** → install it (see Ubuntu repo step above).
- **git ignores gh's helper** → check `git config --get credential.helper` says
  `!f() { gh auth git-credential ... }`; re-run `gh auth setup-git` if not.
- **Repo moved / wrong remote** → confirm remote points at your fork:
  `git -C ~/pi-config remote get-url origin` should be `https://github.com/peq22/Pi-sync.git`.
- **pi doesn't pick up a change** → run `./sync.sh`. For everything except
  `settings.json`, confirm the file is a symlink: `ls -l ~/.pi/agent/<file>`.
  `settings.json` is generated (not a symlink) — its sources are
  `agent/settings.json` + `agent/settings.local.<hostname>.json`.
- **A symlink broke** (target renamed) → `./sync.sh` recreates it.
- **Restore a pre-symlink original** → backups are saved as
  `~/.pi/agent/<name>.bak.<timestamp>` the first time linking runs.

## Security notes
- Access is via your **GitHub account token** (HTTPS + gh). It can touch
  anything your account can — keep that token safe (`gh auth status` shows
  scopes). Revoke in GitHub → Settings → Developer settings.
- Keep the GitHub repo **private**. Secrets are gitignored, but private is the
  belt-and-suspenders default.
