# T3 Code on Wodby

What Wodby sets up for T3 Code on this service. It runs the `wodby/t3code` image, which starts `t3 serve` on port 3773. The port is public, and every browser or app must pair before it can do anything.

## Signing in

T3 Code has no password. A device signs in once with a one-time pairing link, which expires after 5 minutes:

- Run the service action "Generate pairing link" and open the link from the task log.
- Or run `t3-pairing-link` in the container's web terminal.

The link uses `WODBY_PRIMARY_URL`, so it points at the environment's main domain. `t3 serve` also prints a pairing link in the container log at each start, but with the pod's internal address: use its token, not its address. List and revoke signed-in devices with `t3 auth session list` and `t3 auth session revoke <id>`.

## Agents

- Codex (`codex`) and OpenCode (`opencode`) are in the image.
- Claude Code (`claude`) is not, because its license doesn't allow redistributing it. The container downloads it from Anthropic to the volume on its first start, in the background, and it updates itself there.
- None of them is signed in. Sign in from T3 Code under Settings → Providers or in the web terminal (`codex login`, `claude auth login`, `opencode auth login`). Those logins are stored on the volume. API keys such as `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` can instead be set as service environment variables, which agents inherit, or per provider in T3 Code's settings.

New threads use **Full access**, T3 Code's default, where agents act without approval prompts and the container is the boundary. Codex's **Supervised** and **Auto** modes run commands in a sandbox that needs unprivileged user namespaces. If the cluster's seccomp or kernel settings block them, those commands fail with a `bwrap` error: use **Full access** for that thread.

T3 Code's built-in browser downloads itself to the volume on first use and uses the same namespaces for Chrome's sandbox. If a browser tab reports that the host blocks the sandbox, `t3 browser setup` can't help inside the container. Setting `T3CODE_SERVER_BROWSER_SANDBOX=0` runs the browser without Chrome's sandbox, so a compromised web page could then reach everything in the container.

## Data

The `data` volume is mounted at `/data`, the home directory of the `node` user (uid 1000). It holds T3 Code's database (`~/.t3`), agent logins, Git credentials and `~/projects`, the default working directory. Everything outside `/data` resets when the container restarts. Tools installed with `npm install -g` or into `~/.local/bin` stay on the volume and take priority over the image's.

Run one replica only: the database is a single SQLite file.

## Changing configuration

- `T3CODE_TELEMETRY_ENABLED` is `false` on this service. Set it to `true` to send T3 Code's product usage data.
- Other `T3CODE_*` variables are read by T3 Code. Keep `T3CODE_HOST` as it is, and keep `T3CODE_PORT` set to `3773`: the endpoint and probes expect that port, and without the variable Kubernetes supplies its own `T3CODE_PORT` value, which stops the server from starting.

Change environment variables on the service, then deploy it.

## Check the result

`curl -fsS http://localhost:3773/.well-known/t3/environment` inside the container reports the running version. It needs no sign-in and the Kubernetes probes use it. `curl -fsS http://localhost:3773/api/auth/session` answers `"authenticated":false` without a session.
