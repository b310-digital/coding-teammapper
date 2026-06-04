# Agents

## Environments

This project runs in two different ways. Detect which one you're in before running commands.

### GitHub Codespaces (single container)

If you are inside a GitHub Codespace (e.g. `opencode` running in the Codespace terminal), **everything already runs in one container** — there is no Docker stack to start.

- **Do not run `docker` or `docker-compose`.** They are unnecessary here and are blocked for the agent.
- Setup runs automatically via devcontainer lifecycle hooks: `pnpm install --frozen-lockfile` + frontend `build:packages` (`updateContentCommand`), then `pnpm run migrate:dev` (`postCreateCommand`). Re-run any of these manually if needed.
- Start the app directly: `pnpm run dev` (run in background).
- Postgres is available as a sidecar at host `postgres:5432` (configured in `.devcontainer/docker-compose.yml`); the backend already points at it via env vars.
- The Playwright MCP / multi-container networking notes below **do not apply** in Codespaces.

#### Off-limits paths (secrets)

Do **not** read, list, search, or edit sensitive folders outside the workspace — they hold credentials (opencode auth, `gh`/cloud tokens, SSH keys), not project code. This is enforced via `deny` rules in `opencode.json` (`permission.read`/`list`/`glob`/`grep`/`edit`), but treat it as a hard rule regardless:

- `~/.config/**` (includes opencode auth and `gh` credentials), `~/.ssh/**`, `~/.aws/**`, `~/.azure/**`, `~/.gnupg/**`, `~/.kube/**`, `~/.docker/**`
- `~/.netrc`, `~/.npmrc`, `~/.git-credentials`
- `/workspaces/.codespaces/**` (Codespaces-injected secrets/config)

If you genuinely need a value from one of these (e.g. a token), ask the user rather than reading the file. When adding tooling, keep secrets in env vars — never copy them into the workspace.

### Local docker-compose (multi-container)

The Docker-based setup below (app + chrome + playwright + postgres containers) applies only to local development with `docker-compose`, **not** to Codespaces.

## Playwright MCP

### Setup

The Playwright MCP connects to a headless Chrome running in a separate Docker container (`chrome`) via CDP. Configuration is in `.mcp.json`. Example:

```
{
    "mcpServers": {
      "playwright": {
        "command": "npx",
        "args": ["@playwright/mcp@latest", "--cdp-endpoint", "http://<CHROME_DOCKER_IP>:9222"]
      }
    }
}
```

### Networking

- The app runs inside the `app` container, Chrome runs in the `chrome` container.
- **Do not use `localhost` or the `app` hostname** to navigate — Chrome cannot resolve them properly.
- **Chrome CDP rejects non-IP Host headers** — Chromium hardcodes a check that the HTTP `Host` header is an IP or `localhost`. There is no flag to disable this. Always use resolved IPs (not hostnames) in CDP endpoint URLs.
- **Resolve container IPs first** with `getent hosts <container>`, then use the IP:

```bash
getent hosts app    # for navigation URLs
getent hosts chrome # for CDP endpoint in .mcp.json
```

### Checklist

1. Start the dev server: `pnpm run dev` (run in background)
2. Wait for the server to be ready: `curl -s -o /dev/null -w "%{http_code}" http://localhost:4200`
3. Resolve the app IP: `getent hosts app`
4. Navigate with Playwright: `browser_navigate` to `http://<resolved-ip>:4200`
5. Use `browser_snapshot` (preferred over screenshots) to inspect the page
