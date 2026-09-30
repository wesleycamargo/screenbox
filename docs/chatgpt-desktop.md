# ChatGPT Desktop in Screenbox

Optional. The XFCE desktop image (`docker/Dockerfile`) installs the Linux ChatGPT Desktop `.deb`
only when you pass its URL at build time:

```bash
docker build -f docker/Dockerfile \
  --build-arg CHATGPT_DEB_URL=<url-to-chatgpt.deb> \
  -t screenbox:latest docker/
```

Without `CHATGPT_DEB_URL` the build skips it and the image is unchanged.

## Where it lives

| Path | What |
|---|---|
| `/usr/lib/chatgpt/` | App files from the package (`ChatGPT`, `codex-launcher`) |
| `/usr/bin/chatgpt` | Package launcher (untouched) |
| `/usr/local/bin/chatgpt-screenbox` | Screenbox wrapper — use this |
| `/usr/local/share/applications/*.desktop` | Menu entry override pointing at the wrapper |

## Launch manually

From a terminal in the desktop (or `docker exec -it -u screenbox <container> bash`):

```bash
chatgpt-screenbox
```

It also appears in the XFCE application menu.

## Why the wrapper

- **`--no-sandbox --disable-setuid-sandbox`** — Electron's sandbox needs unprivileged user
  namespaces, which the container blocks (`unshare -Ur true` → `Operation not permitted`), and the
  package ships no usable setuid `chrome-sandbox`. Screenbox does not add `SYS_ADMIN` or run
  privileged, so the sandbox is disabled instead — same approach as the Chromium wrapper.
  The container itself is the isolation boundary.
- **`DISPLAY` defaults to `:99`** — that is the Xvnc display the entrypoint starts. An existing
  `DISPLAY` is kept.
- **Creates `~/.local/share/pki/nssdb`** — ChatGPT fails to create it on its own. `/home/screenbox`
  is a per-desktop volume, so this is done at launch (not just at build) to cover existing desktops.
- **`--password-store=basic`** — stores app secrets in its own files instead of GNOME Keyring,
  which would otherwise pop up a "choose password for new keyring" dialog on menu launch.
  Same as Chromium in this image.
- **Runs as `screenbox`** — if started as root, it re-execs itself via `gosu screenbox`.

## Reuse your host login (optional)

Set in `.env`, then `docker compose up -d` to restart the MCP service:

```bash
SCREENBOX_CODEX_HOST_DIR=/home/you/.codex
```

New desktops then mount two files from that folder:

| Host file | In desktop | Mode |
|---|---|---|
| `auth.json` | `~/.codex/auth.json` | read-write (so token refreshes are saved) |
| `config.toml` | `~/.codex/config.toml` | read-only |

ChatGPT opens already signed in. Both files must exist, or desktop creation fails.
The path is resolved by the Docker daemon, so use the host path, not a path inside the MCP container.

Caveats:
- `auth.json` holds live tokens. Every desktop (and any agent driving it) can use your account.
- The host app and desktops share one refresh token. If one refreshes it, the others may need to sign in again.
- Existing desktops only get the mounts after they are recreated.

## Expected warnings (non-fatal)

DBus messages about a missing `/run/dbus/system_bus_socket`. There is no system bus in the
container; ChatGPT starts fine without it.
