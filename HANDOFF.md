# HANDOFF

Sprites-Codespaces was built and tested in one session from a **GitHub
Codespace** against org `isak-fire-protection` on api.sprites.dev. Read
`CLAUDE.md` first: it holds the design and the gotcha list. This file covers
where things stand and what has not been proven yet.

---

## Where things stand

| | |
|---|---|
| First commit | `5f2ea38`: `sprite-codespace`, `sprite/codespace`, README, config.example |
| Live sprites | `cs-scratch`, an empty native-mode codespace, confirmed working in the user's browser. Test sprites `cs-proto`, `cs-hello-world` and `cs-vscode-remote-try-python` were destroyed |
| Versions seen | code-server 4.141.0, `@devcontainers/cli` 0.89.0, sprite env 0.0.2-beta.8, Node 24.18.0 on the sprite |

---

## Verified on real sprites

| What | Result |
|---|---|
| `create` native (`octocat/Hello-World`) | Ready in about 33 s |
| `create` devcontainer (`microsoft/vscode-remote-try-python`) | Ready in about 2:50, including `postCreateCommand` (pip install) and 4 of 5 extensions. Pylance was skipped: not on Open VSX |
| `new scratch` | Ready in about 28 s, **and opened in the browser by the user**: the URL proxy, Fly.io login and websockets all work |
| `shell -- cmd`, both modes | Lands in the workspace as the right user (`vscode` in the container), with `CODESPACE_NAME`, `GH_TOKEN` and git `user.name` set |
| `sync` | Re-pushes config and secrets, restarts the editor |
| `rebuild` | Recreates the container in about 42 s |
| Sleep and wake | Went `warm` after about 1 min idle, then `cold`, and the editor answered within 1 s of waking |
| Container stopped under the editor | Healed itself in 6 s through the watchdog. Stopping the service leaves no stray `socat` |
| `list`, `status`, the checkpoint taken on `create` | Work |
| `SWAP_SIZE` | On a throwaway sprite: created at 2G, re-enabled by an editor restart after `swapoff`, resized to 1G, an invalid value only warned, and `0` removed the file |
| `./.sprite-codespace/` config layer | `SSH_KEY` and `SWAP_SIZE` read from it alone, with the global config dir empty |
| `SSH_KEY` + `ssh-auth`, native | Passphrase prompt through `--tty`, then `Identity added`, `status` shows `unlocked`, a second run says already unlocked, `/dev/shm` left empty. In the shell: `SSH_AUTH_SOCK` set, `ssh-add -l` lists the key, `git commit` is SSH-signed, and `ssh -o BatchMode=yes -T git@github.com` gets past host-key checking (the test key is not on GitHub, so it then gets `Permission denied`, as expected) |
| `SSH_KEY` + `ssh-auth`, devcontainer (`vscode-remote-try-node`) | Same results inside the container. The container user `node` is uid 1001, which settles the `updateRemoteUserUID` assumption below |
| Abandoned `ssh-auth` (client killed at the prompt) | The session and the key directory were both gone 140 s later, through the 2-minute timeout |
| GitHub token precedence | Local harness only: `stage_secrets` with fake tokens and a fake `gh`, for every rung, empty values, the Codespace warning and no token at all. The sprite side is unchanged and receives the same `KEY=value` lines as before; not yet run against a sprite |
| First real use (`cs-ipm-simplified`, 2026-10-26) | Showed three bugs, all fixed and re-tested on a throwaway sprite: commits authored as `Sprite` (the image's `~/.gitconfig`), the private dotfiles clone missing its token, and the install-command typo passing silently. The fourth problem was the config itself: a passphrase-protected key was set as `SIGNING_KEY` instead of `SSH_KEY` |
| shellcheck | Clean on both scripts |

---

## Not yet tested

Built from documentation or reasoning only. Treat these as unproven:

- **A devcontainer-mode codespace opened in a browser.** Only native mode
  (`cs-scratch`) was opened. The devcontainer editor was checked with
  `curl /healthz` on the sprite.
- **Dotfiles in either mode**, including the user's private
  `AnimMouse/dotfiles-codespaces` and the `SSH_ANIMMOZ_KEY`-style secrets it
  expects. This is the most valuable next test.
- **A private repo clone and `git push` with a real token.** Every test used
  the Codespace's repo-scoped `gh` token against public repos. The credential
  helper was only checked for being configured.
- **Commit signing with `SIGNING_KEY`.** Signing through `SSH_KEY` is
  verified in both modes, and the uid 1001 mapping the container needs has
  been observed. The `SIGNING_KEY` path itself (a passphrase-less key copied
  to the sprite) has not been run.
- **`ssh-auth` in a real terminal.** It was only driven through `script` with
  piped input. Untested there: Ctrl-C at the prompt (it should reach the
  remote `ssh-add` as a raw `^C`, and the local trap also cleans up), and
  `sprite exec --tty` from Git Bash on Windows (mintty is not a native
  console, which some CLIs need `winpty` for).
- **The agent across a real sprite reboot.** It is expected to come back
  empty, with `status` saying `locked`. Not observed, because no real reboot
  has been seen yet.
- **A real key pushing over SSH**, as opposed to getting past host-key
  checking.
- **`TIMEZONE`, `CODE_SERVER_VERSION`, `DEFAULT_IMAGE`, `DEFAULT_MODE`, `SPRITE_ORG`**,
  and per-codespace `<name>.secrets` overrides.
- **A real sprite reboot.** `cold` turned out to be a suspend, so the
  boot-time service restart (dockerd, then editor, then `devcontainer up` on a
  stopped container) was only simulated with `docker stop` plus a service
  restart.
- **Port forwarding through `<url>/proxy/<port>/`** and `sprite proxy`.
- **`open` and `rm`** (`rm` relies on `sprite destroy`'s own prompt).
- **The local CLI on Git Bash for Windows, or on macOS.** It was only run on
  Linux. Watch for `column` (possibly missing in Git Bash), `jq` (not bundled
  with Git for Windows), `stat -c` versus `stat -f`, and `readlink -f`.
- **docker-in-docker inside a devcontainer on a sprite.** The sibling repo hit
  missing legacy iptables tables on its 6.18 kernel. Sprites run 6.12-fly,
  which has not been checked.

---

## Suggested next steps

1. Put a broad `GH_TOKEN` and the dotfiles secrets in
   `~/.config/sprite-codespace/secrets`, set `DOTFILES_REPOSITORY`, and create
   a codespace for one real private repo with a `devcontainer.json`. That
   covers most of the untested list at once.
2. Run the local CLI from Git Bash on the Windows PC and fix whatever breaks.
3. Consider an idle timeout for forgotten browser tabs, which keep a sprite
   running. Codespaces stops after 30 minutes. One way to do it: have the
   editor service watch code-server's websocket connections.
4. Consider a `gc` like the sibling repo's, for codespaces unused for 30 days.
   Checkpoints and `last_running_at` from `sprite --json list` would be the
   clock.
