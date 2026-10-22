# Sprites-Codespaces

GitHub Codespaces on [Fly.io Sprites](https://sprites.dev): one sprite per
codespace, and the sprite's own URL is a VS Code editor on your repo.

```bash
sprite-codespace create AnimMouse/some-project
#   cs-some-project is ready
#   https://cs-some-project-<org>.sprites.app
```

Open the URL, sign in with Fly.io, and you are in VS Code (code-server) with
the repo open, a terminal, `git push`, `gh`, your dotfiles and your secrets.
This is the cloud version of
[NixOS-Codespaces-VirtualBox](https://github.com/AnimMouse/NixOS-Codespaces-VirtualBox):
the same model, but nothing runs on your PC.

| | GitHub Codespaces | NixOS-Codespaces-VirtualBox | **Sprites-Codespaces** |
|---|---|---|---|
| A codespace is | a VM | a container on one VM | **a sprite** (a Firecracker VM) |
| Editor at | `*.github.dev` | `http://localhost:800x` | `https://cs-<name>-<org>.sprites.app` |
| Who can open it | you | whoever is on localhost | members of your Sprites org (Fly.io login) |
| `devcontainer.json` | yes | yes | yes, with the real devcontainer CLI |
| No `devcontainer.json` | universal image | `base:ubuntu` image | the sprite itself: node, python, go, rust, java, ruby, bun, deno ... |
| What persists | `/workspaces` | `/persist` | **the whole sprite** and checkpoints |
| Idle | stops after 30 min | always on | sleeps when nobody is connected and wakes on the next request |
| Secrets | Codespaces secrets | `/persist/codespace/secrets` | `~/.config/sprite-codespace/secrets` |
| Forwarded ports | `*-3000.app.github.dev` | ports on localhost | `<url>/proxy/3000/` or `sprite proxy 3000` |

## Setup

You need the [`sprite` CLI](https://sprites.dev), logged in (`sprite login`),
plus `bash`, `jq` and, optionally, `gh`. On Windows, use Git Bash (install
`jq` separately), or run it from the NixOS VM or a GitHub Codespace.

```bash
git clone https://github.com/AnimMouse/Sprites-Codespaces ~/Sprites-Codespaces
ln -s ~/Sprites-Codespaces/sprite-codespace ~/.local/bin/sprite-codespace

mkdir -p ~/.config/sprite-codespace
cp ~/Sprites-Codespaces/config.example ~/.config/sprite-codespace/config   # then edit
```

### Secrets

These are `KEY=value` lines, the equivalent of Codespaces user secrets:

```bash
install -m 600 /dev/null ~/.config/sprite-codespace/secrets
cat >> ~/.config/sprite-codespace/secrets <<'EOF'
NPM_TOKEN=npm_xxxxxxxx
SSH_ANIMMOZ_KEY=...
EOF
```

`~/.config/sprite-codespace/<name>.secrets` overrides the global file for one
codespace. Secrets reach `postCreateCommand`, the dotfiles install script, and
every terminal. After editing, run `sprite-codespace sync <name>` to push the
change.

**`GH_TOKEN`** is how git and `gh` authenticate inside a codespace. A git
credential helper turns it into HTTPS credentials, the way Codespaces uses
`GITHUB_TOKEN`. If you don't set it, it defaults to `gh auth token` from your
machine. Inside a GitHub Codespace that token only covers one repo, so in that
case put a real token in the secrets file.

## Daily use

```bash
sprite-codespace create owner/repo        # new codespace (about 30s native, 2-4 min with a devcontainer)
sprite-codespace new scratch              # empty workspace, no repo
sprite-codespace list                     # name, running/warm/cold, URL
sprite-codespace open repo                # the URL, in your browser
sprite-codespace shell repo               # a terminal in the workspace (inside the container, if any)
sprite-codespace shell repo -- npm test   # one-shot
sprite-codespace sync repo                # after changing secrets, config or your gh login
sprite-codespace rebuild repo             # after changing devcontainer.json
sprite-codespace checkpoint repo "before upgrade"
sprite-codespace logs repo                # when the URL doesn't load
sprite-codespace rm repo                  # destroy it (asks first)
```

`<name>` takes the short form (`repo`) or the full sprite name
(`cs-repo`). Running `create` again on an existing codespace re-runs setup;
it doesn't start over.

### Modes

- **native**: the sprite is the machine and code-server runs on it directly.
  The sprite image already has most language toolchains installed, so there is
  nothing to build. This is used when the repo has no `devcontainer.json`.
- **devcontainer**: dockerd runs as a sprite service, the official
  devcontainer CLI builds `.devcontainer/devcontainer.json` (features,
  `postCreateCommand` and so on), and code-server runs inside the container.
  This is used when the repo has a `devcontainer.json`. Extensions and settings
  listed under `customizations.vscode` are installed too. Extensions come from
  Open VSX, so Microsoft-only extensions such as Pylance are skipped, with a
  note saying so.

Force either mode with `--native` or `--devcontainer`.

### Ports

The sprite URL serves the editor, and the editor proxies everything else.
Start a dev server on port 3000 and open
`https://cs-repo-<org>.sprites.app/proxy/3000/`, or use `/absproxy/3000/` for
apps that expect to be served from the root. You can also forward the port to
your machine with `sprite proxy -s cs-repo 3000`.

## How it works

```
your machine                                  sprite cs-repo
──────────────                                ─────────────────────────────────────────
sprite-codespace ──sprite exec──▶  /usr/local/bin/codespace  (sprite/codespace)
                   (+ uploads:      ├─ /workspaces/repo          the clone
                    config,         ├─ /opt/codespace/{config,secrets,git/}
                    secrets,        └─ sprite services:
                    signing key)       editor  (http_port 8080, behind the sprite URL)
                                         native:       code-server on the workspace
                                         devcontainer: devcontainer up → code-server in
                                                       the container → socat :8080
                                       docker  (devcontainer mode only)
```

- **The URL is the login.** The sprite URL uses `sprite` auth, which means a
  Fly.io sign-in restricted to org members. That is why code-server runs with
  `--auth none`. `sprite-codespace` refuses to work on a sprite whose URL has
  been made public, because anyone could then open the editor and its
  terminal.
- **Sleeping and waking.** Sprites pause when there are no HTTP requests or
  sessions. Opening the URL again wakes the sprite, services restart, and the
  editor service runs `devcontainer up` again, which also runs
  `postStartCommand`. A devcontainer codespace can take a few seconds to
  answer after a cold start, so if the first load fails, refresh.
  **An open editor tab keeps the sprite awake**, so close the tab when you're
  done.
- **Persistence.** The entire sprite filesystem persists, including `$HOME`,
  installed packages, docker images and `/workspaces/*`, so this is more
  persistent than real Codespaces. `create` takes a checkpoint of the fresh
  codespace, and `sprite restore` rolls back to it.
- **No secrets in URLs or command lines.** Secrets are uploaded as files into
  a `0700` inbox, then moved to `/opt/codespace/secrets` (`0600`) and
  `~/.codespace-secrets` (`0600`). The world-readable
  `/etc/profile.d/codespace.sh` only sources them.

## Limits

- One editor per sprite, because a sprite has one URL. One codespace per
  sprite is also how Codespaces works.
- Extensions come from Open VSX, not the Microsoft Marketplace.
- Commit signing needs a passphrase-less key (`SIGNING_KEY`), because there is
  no agent forwarding into a sprite. Use a dedicated key for this.
