# Roadmap

Planned improvements to make Observal easier to install, configure, and use out of the box.

---

## 1. Split Installation: Admin vs User Paths

**Problem:** Today, every user must clone the repo and run `docker compose up` to get the full stack (Postgres, ClickHouse, Redis, OTel Collector, API, Web UI, Worker). This is fine for an admin setting up the platform, but individual developers who just want to send traces shouldn't need to run any containers.

**Goal:** Two distinct installation paths:

### Admin Install (platform operator)

The admin deploys the full stack once — on a shared server, VM, or Kubernetes cluster. This is the only person who touches Docker.

```
observal admin deploy   # or docker compose up
```

The admin gets a URL like `https://observal.internal.company.com` and distributes it to the team.

### User Install (developer)

A developer installs only the CLI and points it at the admin's URL. No Docker, no databases, no containers.

```
pip install observal          # or: uv tool install observal
observal init
# Server URL: https://observal.internal.company.com
# Email: dev@company.com
```

That's it. The CLI saves the server URL and API key to `~/.observal/config.json`. All telemetry, dashboard access, and MCP registration goes through the remote server.

### What changes

- `observal init` should detect whether it's talking to a local or remote server and skip any Docker/infra prompts for remote users.
- The CLI should never require Docker as a dependency for non-admin users.
- Documentation should clearly separate the two paths: "Setting up Observal (admin)" vs "Connecting to Observal (developer)".
- Consider a `pip install observal` distribution (PyPI) so users don't need to clone the repo at all.

---

## 2. Native Claude Code OpenTelemetry Integration via `observal init`

**Problem:** Getting traces from Claude Code currently requires manually exporting environment variables before launching Claude:

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
claude
```

This is fragile — it only works for that shell session, it's easy to forget, and it's not something you'd demo to a new user.

**Goal:** `observal init` should automatically configure Claude Code's telemetry so traces flow immediately, with zero manual env var setup.

### How Claude Code telemetry works

Claude Code reads OpenTelemetry configuration from environment variables. These can be set persistently via Claude Code's **managed settings** file at `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317",
    "OTEL_EXPORTER_OTLP_HEADERS": "Authorization=Bearer <api-key>"
  }
}
```

When these are set in `settings.json`, every Claude Code session automatically exports telemetry — no shell env vars needed.

### What `observal init` should do

After the existing init flow (server URL, email, API key), add a step:

1. Detect if Claude Code is installed (`which claude` or check `~/.claude/` exists).
2. If found, ask: *"Configure Claude Code to send telemetry to Observal? [Y/n]"*
3. If yes, read `~/.claude/settings.json`, merge the `env` block with the correct OTEL vars pointing at the user's Observal server, and write it back.
4. Preserve any existing settings in the file (plugins, other env vars, etc.).

```
$ observal init
Server URL [http://localhost:8000]: https://observal.internal.company.com
Email: dev@company.com
Name: Dev

✓ Account created. API key saved to ~/.observal/config.json

Detected Claude Code installation.
Configure Claude Code telemetry → Observal? [Y/n]: y
✓ Updated ~/.claude/settings.json — Claude Code will send traces to Observal.

You're all set. Open Claude Code and traces will appear in the dashboard.
```

### Endpoint resolution

- If the Observal server is local (`localhost`), point OTEL at `http://localhost:4317` (the collector).
- If the server is remote, point OTEL at `https://<server>:4317` or whatever the admin has configured as the collector's public endpoint.
- The collector endpoint should be discoverable from the API (e.g. `GET /api/v1/config/otel-endpoint`) so the CLI doesn't have to guess.

### The demo experience

After these two changes, the full onboarding for a new user becomes:

```bash
pip install observal
observal init
# point at server, say yes to Claude Code integration
claude
# traces are already flowing
```

No Docker. No env vars. No config files to edit. Open the dashboard and see traces from the first Claude Code session.


---

## 3. Comparative Testing Across IDEs

**Problem:** Teams use a mix of agentic IDEs — Claude Code, Cursor, Kiro, Gemini CLI, Windsurf, Codex CLI — but have no way to objectively compare how they perform on the same tasks. Choosing an IDE is vibes-based. So is deciding whether to renew a $20/month seat.

**Goal:** Let teams run the same task across multiple IDEs and compare results side-by-side with real data.

### How it works

Define a test task (e.g. "add pagination to the users endpoint") and run it across IDEs. Observal captures traces from each run and produces a comparison:

```
$ observal bench run --task "add pagination to /api/users" --ide claude-code --ide cursor --ide kiro
```

Each IDE session is traced independently. After all runs complete, Observal generates a comparison scorecard:

| Dimension         | Claude Code | Cursor | Kiro   |
|-------------------|-------------|--------|--------|
| Total time        | 2m 14s      | 3m 01s | 2m 48s |
| Tool calls        | 12          | 18     | 14     |
| Errors            | 0           | 2      | 1      |
| Tokens used       | 24,310      | 41,200 | 29,800 |
| Estimated cost    | $0.07       | $0.12  | $0.09  |
| Eval score        | 8.2/10      | 6.9/10 | 7.5/10 |
| Code correctness  | ✓           | ✓      | ✓      |
| Tests passing     | 14/14       | 12/14  | 13/14  |

### What this enables

- **IDE selection with data.** Instead of "I like Cursor" vs "I like Claude Code", teams can see which IDE actually produces better results for their codebase and task types.
- **Cost justification.** Show leadership exactly what each IDE costs per task and what quality it delivers.
- **Regression detection.** Re-run the same benchmark after an IDE update to see if things got better or worse.
- **Prompt/config tuning.** Compare the same IDE with different system prompts, MCP servers, or skill configurations to find the optimal setup.

### What needs to be built

- A `bench` CLI command group for defining, running, and comparing benchmark tasks.
- Task definitions — either freeform prompts or structured specs with expected outcomes (files changed, tests that should pass).
- A way to launch the same task in each IDE programmatically (or guide the user through manual runs with trace correlation).
- Comparison view in the web dashboard — side-by-side traces, cost breakdown, eval scores.
- Historical benchmarks so teams can track IDE performance over time.

Nobody else has cross-IDE telemetry data. This is the unique thing Observal can do that no individual IDE vendor will ever build.
