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
- **Commit signing (`SIGNING_KEY`).** Untested end to end. In devcontainer
  mode it depends on the devcontainer CLI's `updateRemoteUserUID` giving the
  container user uid 1001, so that it can read the 0700
  `/opt/codespace/git` mount. That is assumed, not observed.
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
