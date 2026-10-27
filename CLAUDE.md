# CLAUDE.md

Project context for Sprites-Codespaces: GitHub Codespaces on Fly.io Sprites.
The sibling project, and the source of most of the design, is
[NixOS-Codespaces-VirtualBox](https://github.com/AnimMouse/NixOS-Codespaces-VirtualBox).
Read its `CLAUDE.md` §6 and §8 before re-solving anything about dotfiles,
secrets or devcontainer plumbing. Most of those problems are already solved
there.

---

## 1. Goal

`sprite-codespace create owner/repo` gives you a browser VS Code on that repo,
at the sprite's own URL, with git push, `gh`, dotfiles and secrets working.
Repos with a `devcontainer.json` are built with the real devcontainer CLI,
which keeps `devcontainer.json` files portable to real Codespaces and to the
NixOS VM.

**Success criterion:** one command from a machine with `sprite login` done
gives a working codespace, with no manual steps on the sprite.

---

## 2. Environment facts — read before proposing anything

| Fact | Consequence |
|---|---|
| The user's PC is Windows with **WSL2 and Hyper-V deliberately disabled** (see the sibling repo's §2) | Never tell the user to use WSL. The local CLI runs from Git Bash on Windows, from the NixOS VM, or from a GitHub Codespace |
| The browser is Firefox, and the user dislikes Electron | The editor is code-server in the browser. Never propose VS Code Desktop or Remote-SSH as the primary interface |
| A sprite is Ubuntu 26.04, user `sprite` (uid 1001), passwordless sudo, 8 CPU / 15 GB / 99 GB as observed | Plenty of room for docker and devcontainers |
| **No systemd on a sprite** | Long-running processes are sprite services (`sprite-env services`), which restart at boot and on crash |
| The sprite image ships node, python, go, rust, java, ruby, bun and deno as shims in `/.sprite/bin` | Native mode needs no toolchain installs |
| The sprite URL defaults to `sprite` auth: a Fly.io login, org members only | That login is the editor's only lock, and code-server runs with `--auth none` |

---

## 3. Locked architectural decisions

These are settled. Do not re-open them unless the user explicitly asks.

### One sprite = one codespace
This is the same model as real Codespaces, where each codespace is its own VM.
A sprite has exactly one URL and one `http_port` service, so several editors
per sprite would need path routing, which code-server handles badly. The
sprite name is `cs-<slug>`, and the prefix (`SPRITE_PREFIX`) is what `list`
filters on.

### Two halves
- **`sprite-codespace`** runs on the user's machine. It drives everything
  through the `sprite` CLI and never reaches the sprite any other way.
- **`sprite/codespace`** is the agent. It is uploaded on every
  `create`/`sync`, installed to `/usr/local/bin/codespace`, and also serves as
  the `editor` service's entrypoint.

Keep the local half thin. Logic that touches the sprite belongs in the agent,
where it can be tested by `sprite exec` and runs at boot.

### Two modes, chosen like Codespaces chooses
- **auto** (default): devcontainer if the repo has
  `.devcontainer/devcontainer.json` or `.devcontainer.json`, native otherwise.
- **native**: code-server runs directly on the sprite. The sprite image plays
  the role of the Codespaces universal image.
- **devcontainer**: dockerd runs as a sprite service and the devcontainer CLI
  builds the container. code-server runs **inside** the container, and the
  `editor` service runs `devcontainer up`, then code-server, then `socat :8080`
  to the container IP.

### The editor is the `editor` sprite service on `http_port 8080`
The proxy auto-starts it on the first request after the sprite wakes, and it
restarts at boot. In devcontainer mode it is idempotent: every start runs
`devcontainer up`, which also runs `postStartCommand`. It then watches the
container and exits if the container stops, so the sprite restarts it.

### Auth is the sprite URL's, never code-server's
`--auth none` plus sprite-auth URL means no password to generate, store or
show. The local CLI **refuses** to work on a sprite whose URL auth is
`public`. Keep that check.

### Credentials: `GH_TOKEN` for HTTPS, optional `SSH_KEY` through an agent on the sprite
A git credential helper that only reads `$GH_TOKEN`, set with
`git config --system`, works in any image with or without `gh`. HTTPS remotes
always use it.

`SSH_KEY` (added 2026-10-25, at the user's request) brings back the sibling
repo's `ssh-auth`. `sprite exec` cannot forward the PC's agent, so each sprite
runs an `ssh-agent` service, and `sprite-codespace ssh-auth` loads the key
into it:
- The private key goes over `sprite exec --file` into a `mktemp -d` under
  `/dev/shm`, which is tmpfs, so it never reaches disk or a checkpoint.
- `ssh-add` prompts on a `--tty` session.
- The directory is removed by the agent's trap, by the local side, and by a
  5-minute sweep.

Only the `.pub` persists, in `/opt/codespace/git/ssh_key.pub`. It also signs
commits, through the agent, unless `SIGNING_KEY` (passphrase-less, copied to
the sprite) is set. One key does authentication and signing, as in the
sibling repo. Do not rewrite HTTPS remotes to SSH: `GH_TOKEN` already covers
them, and the agent is empty after a reboot until `ssh-auth` runs again.

### Persistence: the whole sprite, plus checkpoints
There is no `/persist` split. The sprite overlay persists everything, and
`create` takes a checkpoint of the fresh codespace.

---

## 4. Non-goals

- Do not add a password, OAuth proxy or tunnel in front of code-server. The
  sprite URL's auth is the design.
- Do not run several codespaces in one sprite (§3).
- Do not use Coder, Kubernetes, envbuilder or Podman (same as the sibling repo).
- Do not make the local half depend on anything beyond `sprite`, `bash`, `jq`,
  and optionally `gh` and `git`.
- Do not write into the user's checkout. A generated `devcontainer.json` goes
  to `/opt/codespace/devcontainer.json` and is passed with `--override-config`.

---

## 5. Repository layout

```
sprite-codespace      # local CLI: create/new/list/open/shell/sync/rebuild/logs/status/sleep/checkpoint/ssh-auth/rm
sprite/codespace      # in-sprite agent: setup/editor/refresh/rebuild/shell/logs/status/sleep/ssh-auth/ssh-agent/keepalive
config.example        # ~/.config/sprite-codespace/config template
README.md             # user-facing docs
CLAUDE.md             # this file: design authority
HANDOFF.md            # where things stand, what is untested
.gitignore            # .sprite-codespace/, the per-directory config layer
```

On the sprite:

```
/usr/local/bin/codespace          the agent
/opt/codespace/state/             name, mode, workspace, remote_user, remote_workspace,
                                  idle_since (set by `sleep`: heartbeats up to it are ignored)
/opt/codespace/config             non-secret settings, printf %q-quoted (sourced)
/opt/codespace/secrets            KEY=value, 0600 (parsed with jq, never sourced)
/opt/codespace/git/               0700; mounted at /mnt/codespace-git in containers:
                                    id_signing(.pub)  SIGNING_KEY
                                    ssh_key.pub       SSH_KEY's public half (never the private)
                                    known_hosts       GitHub's keys from api.github.com/meta
/opt/codespace/ssh-agent/         0700, agent.sock; the *directory* is mounted at
                                  /mnt/codespace-ssh, so a restarted agent's new socket shows up
/dev/shm/codespace-ssh.XXXXXX/    ssh-auth upload, seconds-lived (tmpfs)
/etc/ssh/ssh_known_hosts2         copy of git/known_hosts, sprite and container
/opt/codespace/cache/             code-server tarball cache; mounted at /var/cache/codespace
/opt/codespace/devcontainer-cli/  devcontainer CLI, own npm prefix
/var/lib/docker/swapfile          extra swap when SWAP_SIZE is set (ext4, see §7)
/workspaces/<name>                the checkout
~/.codespace-secrets              export lines, 0600, sourced by /etc/profile.d/codespace.sh
~/.codespace-inbox/               0700 upload drop; emptied by import_inbox
/.sprite/logs/services/*.log      service logs (editor, docker, ssh-agent, keepalive)
task codespace-idle               the keepalive's hold (Tasks API), while the editor is in use
```

Locally, the config is `~/.config/sprite-codespace/{config,secrets,<name>.secrets}`,
plus the same files in `./.sprite-codespace/` (relative to the working
directory), which are read second and so win. It is a directory with a
distinctive name, not a bare `./config`, because the CLI *sources* it, and
`~/.ssh/config` exists.

---

## 6. Secrets flow

1. The local CLI merges, in order, the global and then the local `secrets`,
   then the global and then the local `<name>.secrets` (later keys win). It
   then resolves **one** GitHub token, first non-empty wins (the user's order,
   2026-10-25): `GITHUB_TOKEN` in the secrets, `GH_TOKEN` in the secrets,
   `GITHUB_TOKEN` in the local environment, `GH_TOKEN` in the local
   environment, then `gh auth token`. Any existing `GH_TOKEN`/`GITHUB_TOKEN`
   lines are replaced by that one value under both names. Two different
   values would split identities: the git credential helper reads
   `GH_TOKEN`, and `gh` prefers `GH_TOKEN` over `GITHUB_TOKEN`. It reports
   the source, never the value.
2. It uploads the result with `sprite exec --file` into
   `~/.codespace-inbox` (0700). Values never appear in a command line.
3. `import_inbox` moves it to `/opt/codespace/secrets` (0600).
4. Native mode: the secrets are written as `~/.codespace-secrets` and sourced
   by `/etc/profile.d/codespace.sh`, which `/etc/bash.bashrc` also sources so
   that non-login shells get them. The editor sources it before `exec`, so its
   terminals inherit the values.
5. Devcontainer mode: `--secrets-file` covers `up`, lifecycle commands and the
   dotfiles install. Afterwards the same `~/.codespace-secrets` and profile
   snippet are copied into the container. code-server is started from
   `bash -l`.

---

## 7. Known gotchas — found while building, all encoded as fixes

| Gotcha | Handling |
|---|---|
| **A running service does not keep a sprite awake.** The bundled docs and CLI help say services keep it alive. The online docs (docs.sprites.dev, "Lifecycle" and "Keeping a Sprite Running") are right: activity means a command, a session producing output, a connection to the URL, or a service *handling traffic*. Idle for about 30 s → `warm` (suspended, not billed), later `cold` | Holding a sprite up takes the **Tasks API** (`/.sprite/api.sock`, `PUT /v1/tasks/<name>` with `expire` ≤ 1h). `IDLE_TIMEOUT` (default 30m, max 240m, 0 off): the `keepalive` service renews task `codespace-idle` (5 min TTL) every 60 s while code-server's `/healthz` `lastHeartbeat` is within the timeout, then deletes it |
| **The status from `sprite list` is not a reliable pause signal.** It reported `cold` one second after the keepalive renewed its task, with uptime unchanged | Measure pauses from inside instead: a throwaway service writing `date` every 10 s, then look for gaps. That showed a sprite held for the whole timeout, then paused **within 10 s of the task's delete** |
| **`sprite-codespace sleep`** (manual idle) | Writes `state/idle_since` and **restarts** the keepalive service, rather than just deleting the task. The restart's TERM trap releases the task, and the new loop reads the mark before it can renew, so no renewal can land after the release. Heartbeats at or before the mark are ignored, so the next real editor use re-arms it with nothing to undo. An open tab is detected as an established connection on `:8080` (our own probes don't outlive their call), and only warned about, because the tab would re-arm it within a minute. Measured: the sprite paused 53–63 s after the command's session ended, not 30 s |
| **Service log timestamps are when the log was collected**, which for a line written just before a pause is after the next wake (a 00:12:39 release was stamped 00:16:26) | Don't read pause timing from `/.sprite/logs`. Use the clock-service method |
| **code-server's heartbeat:** it beats on every real request, and every 60 s while any connection stays open (an open tab's websocket). Polling `/healthz` does **not** beat (`lastHeartbeat` stayed 0 for minutes) | `/healthz` is the activity signal, safe to poll. It resets to 0 when code-server restarts, so the loop tracks the newest value ever seen |
| **`npm i -g` lands in the active nvm version's directory**, which drops off PATH after `nvm use` | Install the devcontainer CLI with `--prefix /opt/codespace/devcontainer-cli` and symlink it into `/usr/local/bin` |
| **Exec sessions and services do not pick up supplementary groups** added after boot, so `usermod -aG docker sprite` does nothing and `sg` does not exist | Run `dockerd --group sprite`, so the socket is group-owned by sprite's primary group |
| **`sprite exec --file` uploads land 0644** and create parent directories | Create a 0700 inbox in a separate `exec` first, then upload into it |
| **The devcontainer CLI cannot publish ports** (same as the sibling repo) | `socat` on the sprite to the container IP, which is re-read on every editor start |
| **socat outlives a stopped container** and forwards into nothing | The editor service polls `docker inspect .State.Running` and exits when it is false. The sprite restarts the service, and `container_up` brings everything back (measured: 6 s) |
| **Only one service may have `http_port`** | One `editor` service. `register_editor_service` deletes and recreates it |
| **`sprite info` has no `--json`** | Use `sprite --json list` and select by name (`sprite_json`) |
| **Inside a GitHub Codespace, both `$GITHUB_TOKEN` and `gh auth token`** are that codespace's token, scoped to one repo | Warn when `CODESPACES=true` and the token came from the environment or `gh`. A token in a secrets file outranks both |
| **Reading the Sprites API token is blocked** for Claude by the auto-mode classifier, so the URL proxy cannot be curl-tested from here | Test the editor inside the sprite (`curl localhost:8080/healthz`) and ask the user to open the URL |
| **`sprite exec` prints `bind: warning: line editing not enabled`** from the sprite's `.bashrc` | Harmless. Filter it when reading output |
| **The code-server installer draws a progress bar on stderr** | Send install output to `/opt/codespace/logs/code-server-install.log` |
| **The sprite root is overlayfs, and swapon refuses files on it.** `/var/lib/docker` is a separate ext4 mount (`/dev/nbd1`) in both modes, even when docker is not installed. Nothing re-runs swapon at boot | `SWAP_SIZE` puts the file at `/var/lib/docker/swapfile`, and `ensure_swap` runs on `setup`, `refresh` and every editor start. A changed size recreates the file, `0` removes it, and failures only warn |
| **A sprite keeps a TTY exec session alive after its client goes away.** A killed `ssh-auth` left `ssh-add` waiting at its prompt indefinitely, holding the uploaded key, and live sessions can keep a sprite awake | `ssh-add` runs under `timeout --foreground 120`, so the session ends and the trap removes the key directory. Plain `timeout` puts `ssh-add` in a background process group, where reading the terminal stops it with SIGTTIN. Inspect with `sprite sessions -s <sprite> list` and `kill <id>` |
| **`readpassphrase` flushes pending input** (`TCSAFLUSH`), so a passphrase piped in before the prompt appears is discarded | Only matters for testing. Drive it with `script -qec` and a delay of 15 s or more before writing the passphrase. Real typing is unaffected |
| **`sprite exec -- swapon ...`** (no shell) reports `executable file /usr/bin/swapon not found`: a bare exec does not search `/usr/sbin` | Wrap one-off commands in `bash -c` |
| **The devcontainer CLI gives the container user the sprite user's uid** (`node` became 1001), through `updateRemoteUserUID` | This is what lets the container reach the 0700 `/opt/codespace/git` and `/opt/codespace/ssh-agent` mounts. Observed with `javascript-node`. An image that sets `updateRemoteUserUID: false` would lose both |
| **The sprite image ships `~/.gitconfig` with `user.name = Sprite`, `user.email = noreply@sprites.dev`.** Global beats system, so commits were authored by "Sprite" despite the right identity in `/etc/gitconfig` | `native_configure` unsets those two values **only when they are exactly the image defaults**, so an identity your dotfiles write to `~/.gitconfig` still wins |
| **The dotfiles clone ran without `GH_TOKEN`** (native mode): the repo clone sourced the secrets, but the dotfiles clone did not, so a private dotfiles repo failed with `could not read Username` | `apply_dotfiles_native` exports the secrets in its subshell *before* cloning. A `403` instead of `could not read Username` shows that the helper did send a token |
| **A `DOTFILES_INSTALL_COMMAND` typo** (the user's config had `install.sh.`) silently fell through to symlinking | Warn that it is not in the repo, then try the usual install scripts, as Codespaces does without a command |
| **A passphrase-protected key set as `SIGNING_KEY`** is rejected, and that is easy to mistake for `ssh-auth` not working | The warning names `SSH_KEY` as the option for passphrase-protected keys |
| **Open VSX, not the Marketplace** (same as the sibling repo) | Install `customizations.vscode.extensions` one by one and skip failures with a note. The sha256 marker of the customizations avoids reinstalling on every boot |

Carried over from the sibling repo, and still true here: `--remote-env` and
`--secrets-file` do not persist past `up`; the dotfiles clone runs before any
credentials exist (fixed here with the `GIT_CONFIG_*` credential helper via
`--remote-env`); `devcontainer up` with no workspace path opens no folder; the
CLI refuses to guess a `devcontainer.json`; and liveness is checked by
probing `/healthz` over HTTP, never with `pgrep`.

---

## 8. Working agreements

- Run `shellcheck -x sprite-codespace sprite/codespace` before every commit.
  It must come out clean.
- Test against a real sprite. `create` a throwaway `cs-*`, then `sync` to push
  agent changes to it. **A failed shellcheck in a `&&` chain once meant a test
  ran the old agent**: run `sync` as its own step, then test.
- Throwaway sprites cost money. Destroy the ones you created when you are
  done, but only after the user agrees. `destroy` cannot be undone and takes
  every checkpoint with it.
- Comments explain *why*, as in the sibling repo. Match its tone.
- Never print secret values, not even while debugging. Test for presence
  instead (`test -n "$GH_TOKEN" && echo present`).
- If something in §3 looks wrong given new evidence, say so and explain the
  tradeoff. Don't silently substitute an alternative.
