# Oodle plugin for Claude Code

Oodle observability for Claude Code. You can look into service performance, triage alerts, debug Kubernetes, and query metrics (PromQL), logs, and traces from your terminal, using your [Oodle](https://oodle.ai) telemetry.

The plugin connects Claude Code to the hosted Oodle MCP server for your region and installs the Oodle skills from [oodle-ai/agent-skills](https://github.com/oodle-ai/agent-skills) and [oodle-ai/discovery-agent-skills](https://github.com/oodle-ai/discovery-agent-skills).

## Install

```
/plugin marketplace add oodle-ai/claude-code-plugin
/plugin install oodle@oodle
```

### Sign in

The plugin declares one MCP server per Oodle region. Each signs in with OAuth, so there's no key to copy. After install, run `/mcp`, pick your region's server, and complete the browser login.

| Region | Server in `/mcp` | URL |
|--------|------------------|-----|
| US (`us1`) | `plugin:oodle:oodle-us1` | `https://prod-02-us-west-2.api.oodle.ai/mcp/v1` |
| EU (`eu1`) | `plugin:oodle:oodle-eu1` | `https://eu1.api.oodle.ai/mcp/v1` |
| APAC (`ap1`) | `plugin:oodle:oodle-ap1` | `https://prod-01-ap-south1.api.oodle.ai/mcp/v1` |

In `/mcp`, turn off the two regions you don't use so they stop asking for authentication.

To install from the shell:

```bash
claude plugin marketplace add oodle-ai/claude-code-plugin
claude plugin install oodle@oodle
```

## Skills

The skills come from two Oodle repos, which this plugin declares as dependencies. Installing `oodle@oodle` also installs `oodle-agent-skills@oodle` and `oodle-discovery-skills@oodle` from GitHub. Skill updates reach you when those repos change, with no change needed here.

### Agent skills ([oodle-ai/agent-skills](https://github.com/oodle-ai/agent-skills))

| Skill | Use it for |
|-------|------------|
| `oodle-cli` | Core CLI usage: auth, output formats, time flags, file input |
| `oodle-monitors` | Monitor CRUD, alerting thresholds, query scoping, runbook links |
| `oodle-dashboards` | Dashboard and folder management with safe update and delete |
| `oodle-alerting` | Notifiers, notification policies, muting rules |
| `oodle-metrics` / `oodle-metrics-query` | Label discovery, PromQL building, instant and range queries |
| `oodle-logs` | Log search, aggregations, index pattern discovery |
| `oodle-traces` | Trace search by service, operation, duration, error; TraceQL |
| `oodle-log-metrics` | Log-based metric rules |
| `oodle-drop-rules` | Metric drop rules for ingestion cost control |
| `oodle-synthetic` | HTTP and TCP synthetic monitors |
| `oodle-onboarding` | Integration onboarding: discover the environment and install collectors |
| `oodle-o11y-context` / `oodle-o11y-review` / `oodle-triage` | Observability across the SDLC: change context, telemetry review, alert triage |

### Discovery skill ([oodle-ai/discovery-agent-skills](https://github.com/oodle-ai/discovery-agent-skills))

| Skill | Use it for |
|-------|------------|
| `oodle-discovery` | Read-only discovery of environments, tech stack, observability tools, telemetry volumes and observability cost. Collector scripts compute every figure from vendor APIs (Datadog, CloudWatch, GCP, Elasticsearch, OpenSearch, Mimir, Prometheus/Loki/Tempo, Kubernetes) and save the raw evidence, then it writes a self-contained HTML report. Needs [`uv`](https://docs.astral.sh/uv/) |

Ask "Run the oodle-discovery skill" or "Discover my tech stack and observability setup and generate a report".

### Running skills

Agent skills run as `/oodle-agent-skills:<skill>`, for example `/oodle-agent-skills:oodle-monitors`, and Claude also picks them automatically.

Most of these skills drive the [`oodle` CLI](https://github.com/oodle-ai/oodle-cli), so install and sign in to it for full coverage (see the [Oodle CLI docs](https://github.com/oodle-ai/oodle-cli)). Without the CLI, Claude can still answer read questions through the MCP server's tools.

The Oodle MCP server also hosts its own skills, such as `kubernetes-debugging` and `alert-noise-analysis`. Claude reaches them through the server's `list_skills` and `load_skill` tools.

### Example prompts

- "What's firing right now, and why?"
- "How is the checkout service doing compared to yesterday?"
- "Why are payment-api pods restarting in prod?"
- "Create a monitor for 5xx ratio on the orders service, routed to the on-call notifier."

If you already installed these skills another way (`oodle-agent-skills@oodle-ai` from the agent-skills marketplace, or `npx skills add`), remove that copy so Claude doesn't load every skill twice.

## MCP servers

The servers are declared in [.mcp.json](.mcp.json):

| Server | Sign-in | URL |
|--------|---------|-----|
| `oodle-us1` | OAuth, client ID `9aSE9FiizGd2cvES9fkRXVeoH2NbMlkI` | `https://prod-02-us-west-2.api.oodle.ai/mcp/v1` |
| `oodle-eu1` | OAuth, client ID `60pHxRt5mb4M5J7pcFBN5TEWX25jthu7` | `https://eu1.api.oodle.ai/mcp/v1` |
| `oodle-ap1` | OAuth, client ID `BtkEridc4BuBIhm8E3IKK0XEDYh82s43` | `https://prod-01-ap-south1.api.oodle.ai/mcp/v1` |

All three use callback port `9400`. The regional URLs and client IDs match the Claude Code (MCP) tile on Oodle's Integrations page (`mcp-oauth-config.ts` in the Oodle frontend). Keep them in sync when a region is added or a client ID changes.

See the [Oodle MCP docs](https://docs.oodle.ai/integrations/mcp/claude) for the server's capabilities.

If you already added Oodle by hand with `claude mcp add ... oodle-ai ...`, remove that entry (`claude mcp remove oodle-ai`) so Claude doesn't see two copies of every tool.

## Data handling

### Oodle MCP servers (the declared connectors)

- **Reads:** metrics, logs, traces, RUM sessions, alerts, monitors, and dashboards in your Oodle account, plus queries against Grafana datasources you connected to Oodle (such as CloudWatch, BigQuery, ClickHouse, or SQL databases). Telemetry can contain personal data, such as user IDs or emails in spans and logs.
- **Writes:** with your approval for each tool call, unless you allow it in advance, Claude can create and update Grafana dashboards, create and update monitors, and mute monitors. Every action runs with the signed-in user's Oodle permissions, so a Viewer can't change anything.
- **Stores:** Oodle receives your question when Claude starts an investigation, and the arguments of each tool call, including a one-line statement of intent. Oodle keeps them under its [privacy policy](https://www.oodle.ai/privacy). Dashboards and monitors created through the plugin are saved in your Oodle account.
- **Sign-in:** OAuth through Oodle's sign-in service for your region. The plugin itself stores no data.

### Skills that contact other services

The skills come from the two dependency plugins. Some of them call services directly rather than through the MCP servers:

- **Agent skills** run the [`oodle` CLI](https://github.com/oodle-ai/oodle-cli), which calls the Oodle API with the CLI's own sign-in. The data goes only to Oodle.
- **The discovery skill** runs read-only collectors against the observability tools you choose: Datadog, AWS CloudWatch, GCP Cloud Operations, Elasticsearch, OpenSearch, Grafana Mimir, Prometheus, Thanos, VictoriaMetrics, Loki, Tempo, and your Kubernetes clusters. They use the cloud and kubectl sign-ins you already have, or read-only API keys you enter when asked. The results, including raw API responses with secrets redacted, are written to `discovery-output/` on your machine. They aren't sent to Oodle or anywhere else.

## Develop

Validate the plugin, then install it from this checkout through its own marketplace. The skill dependencies come from GitHub, as they do for users:

```bash
claude plugin validate .
claude plugin marketplace add ./
claude plugin install oodle@oodle
```

The `oodle` plugin is read from this folder in place, so after you edit files, run `/reload-plugins` in the session. Sign in from `/mcp`.

To remove it:

```bash
claude plugin uninstall oodle@oodle
claude plugin prune
claude plugin marketplace remove oodle
```

## Project structure

```
.claude-plugin/
  plugin.json          # Plugin manifest and dependencies
  marketplace.json     # Marketplace: this plugin + both skill repos (from GitHub)
.mcp.json              # Oodle MCP servers: one OAuth server per region
```

## License

MIT. See [LICENSE](LICENSE).
