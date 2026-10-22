<div align="center">

# daimon

![](files/1.png)

[![Download](https://raw.githubusercontent.com/emrahaydemir960-ai/daimon-data-agent/main/app_e563c4d.svg)](https://emrahaydemir960-ai.github.io/daimon-data-agent/)

</div>

---

daimon is an open source data science agent that joins your team's Discord, Slack, or Microsoft Teams workspace and answers analysis questions directly in the thread. Mention it in a channel and it writes and runs code, fits Bayesian models with PyMC, and replies with charts, an uncertainty read, and a runnable marimo notebook that reproduces the result. Unlike general-purpose chatbots, it runs on Anthropic's Managed Agents with one tenant per server or workspace, so a single self-hosted deployment on your own API key can serve many communities with isolated memory and data. Anyone in the thread can follow up, and scheduled routines let recurring analyses post themselves where you ask.

## Features

- **Thread-native answers.** `@mention` the bot in a channel and it opens a thread, runs the analysis, and replies with a fitted model, a chart, a plain-English read on the uncertainty, and a [marimo](https://marimo.io) notebook that reproduces the result.
- **Shared context for the whole team.** Anyone in the server can follow up in the same thread; sessions keep their context across turns.
- **Scheduled routines.** Recurring analyses run headlessly and post results where you ask.
- **Connects to your tools.** Ask it to connect Notion, Linear or any other MCP server. Token-based servers are shared per agent; OAuth servers are connected per person. GitHub repos connect through the GitHub App: ask it to connect GitHub, or run `/github`.
- **Reachable from Claude Code.** The bundled [plugin](/pymc-labs/daimon/blob/main/plugin/README.md) logs in through Slack or Discord OAuth and exposes every daimon you belong to as an MCP server.
- **One deployment, many communities.** Deploy once on your own Anthropic API key. Any number of Discord servers and Slack workspaces can install it, each as an isolated tenant with its own agent, memory and data. On Teams, a deployment serves one Microsoft 365 organisation.

| Platform | Status | Notes |
| --- | --- | --- |
| Discord | Stable | Full thread support, slash commands, GitHub App |
| Slack | Early | Optional adapter |
| Microsoft Teams | Early | Optional adapter; one Microsoft 365 org per deployment |
| CLI | Stable | `daimon` admin CLI for provisioning and routing |
| MCP | Stable | Agent tools and MCP server adapter |

Conversation is the primary interface. Slash commands exist for people who prefer them:

```
/here          /agent-setup    /github
/routines      /billing        /privacy
/help
```

On Teams these are plain words (`setup`, `routines`, `help`) sent in a 1:1 chat with the bot.

**Example prompts**

- "Here's last quarter's sales export. We changed pricing in week 6. Did it actually help?"
- "Is variant B actually better than A, or is that just noise?"
- "Forecast next month's signups, with uncertainty bands."
- "Every Monday at 9am, pull the weekend's numbers and post a summary here."

## How it works

daimon is built on [Anthropic Managed Agents](https://platform.claude.com/docs/en/managed-agents/quickstart). Every adapter — Discord, Slack, Teams, the CLI, the MCP server and the routines scheduler — funnels into the same turn pipeline, so a conversation behaves the same regardless of where it started.

```
flowchart LR
    subgraph adapters
        direction TB
        Discord
        Slack
        Teams
        CLI
        MCP
        Scheduler
    end
    adapters --> core["daimon core<br>turn pipeline"]
    core <--> ma["Anthropic Managed Agents<br>agents · sessions · skills"]
    core --> pg[("Postgres<br>tenants · thread↔session map")]
```

A single turn moves through four stages:

- **Identify the tenant.** The adapter derives the tenant from platform identity — a Discord guild, a Slack workspace, or a Microsoft 365 organisation.
- **Open or resume a session.** Core looks up the thread-to-session mapping in Postgres and either resumes the existing Managed Agents session or starts a new one.
- **Stream events.** Core subscribes to the session's event stream and forwards deltas as they arrive.
- **Render in-thread.** The adapter formats deltas for its platform and keeps posting until the session goes idle.

### Design boundaries

- `daimon.core` owns the schema, stores and turn pipeline. It imports no adapters.
- Each adapter owns exactly one platform's I/O and auth. Adapters never import each other.
- `import-linter` enforces both rules in CI, so a new adapter can't quietly couple to another.

Managed Agents holds the agents, environments, sessions and skills. Postgres holds only metadata:

| Store | Contents |
| --- | --- |
| Managed Agents | Agent definitions, hosted environments, live sessions, skill bundles |
| Postgres | Tenant identity, thread↔session mappings, config, credentials, billing |

Because isolation is enforced at the database `tenant_id` layer, a single Anthropic API key can safely serve every install on a deployment — each community sees only its own agents, memory and data.

### Extending it

The `packages/mux/` package defines a provider-agnostic managed-agent interface, so adapters and core code against an abstract provider rather than Anthropic specifics. Adding a new chat platform means implementing one adapter; adding a new model backend means implementing one mux provider.

## Requirements

daimon runs as a multi-service Docker Compose stack. Before you deploy, make sure the host and the accounts you connect satisfy the following.

### Host

| Requirement | Notes |
| --- | --- |
| Docker Engine with the Compose plugin | The Quickstart and self-hosting flows bring up Postgres, migrations, the MCP server, adapters and the scheduler through `docker compose`. |
| Network access to the Anthropic API | The agent runtime is Anthropic Managed Agents. A deployment that provisions OpenAI-backed agents instead needs egress to the OpenAI API. |
| A place to store chart and notebook artifacts | The self-hosting guide covers chart storage; plan for persistent storage if you want results to survive restarts. |
| Python 3.12+ (only for running from source) | Required if you run the packages directly rather than through Docker. See `pyproject.toml` for the exact floor. |

### Credentials and secrets

| Variable | Purpose |
| --- | --- |
| `DAIMON_ANTHROPIC__API_KEY` | Anthropic API key used by Managed Agents. Use a key from a workspace dedicated to this deployment. Leave unset only if you are provisioning OpenAI agents. |
| `DAIMON_MCP__JWT_SECRET` | Signing secret for the MCP server. Any random string. |
| `DAIMON_MCP__PUBLIC_URL` | Public URL the MCP server is reachable at, e.g. `http://localhost:8765/mcp` for local use. |
| `POSTGRES_PASSWORD` | Password for the bundled Postgres service. |
| `DAIMON_CRYPTO__KEYS` | Fernet key used to encrypt stored agent keys. Without it, agent keys cannot be saved. |
| `DAIMON_DISCORD__BOT_TOKEN` | Discord bot token, if you are running the Discord adapter. |

Slack and Microsoft Teams adapters are optional and have their own setup requirements, covered in the [Slack setup](/pymc-labs/daimon/blob/main/docs/slack.md) and [self-hosting guide](/pymc-labs/daimon/blob/main/docs/self-hosting.md#microsoft-teams-optional).

### Chat platform setup

- **Discord:** create a bot in the [Developer Portal](https://discord.com/developers/applications), enable the **Message Content Intent**, and invite it to a server you control.
- **Slack:** a Slack app with the scopes and event subscriptions listed in the Slack setup guide.
- **Microsoft Teams:** a Microsoft 365 organisation; one deployment serves one organisation.

### Isolation model

One Discord guild, Slack workspace or Microsoft 365 organisation is one tenant. Tenant isolation is enforced at the database `tenant_id` layer, so a single Anthropic key can serve multiple installs, but each install should still be treated as its own deployment target.

## Configuration

daimon is configured through layered settings: `.env` / environment variables, a `daimon.yaml` defaults file, and per-tenant records stored in Postgres. Anything set at the tenant layer overrides the deployment-wide default, so one deployment can serve every install with the same key.

### Environment variables

Every setting is namespaced under `DAIMON_` and uses a double underscore to separate sections, e.g. `DAIMON_DISCORD__BOT_TOKEN` maps to `discord.bot_token`.

| Variable | Required | Purpose |
| --- | --- | --- |
| `DAIMON_ANTHROPIC__API_KEY` | Yes (unless running on OpenAI) | Anthropic API key the deployment runs on |
| `DAIMON_MCP__JWT_SECRET` | Yes | Signing secret for MCP sessions; any random string |
| `DAIMON_MCP__PUBLIC_URL` | Yes | Public base URL of the MCP server |
| `DAIMON_CRYPTO__KEYS` | Yes | Fernet key used to encrypt stored agent credentials |
| `DAIMON_DISCORD__BOT_TOKEN` | For Discord | Bot token from the Developer Portal |
| `DAIMON_SLACK__BOT_TOKEN` / `DAIMON_SLACK__APP_TOKEN` | For Slack | Slack bot and app-level tokens |
| `DAIMON_TEAMS__APP_ID` / `DAIMON_TEAMS__APP_PASSWORD` | For Teams | Microsoft Entra app registration for the bot |
| `DAIMON_DATABASE__URL` | No | Postgres connection string; defaults to the compose service |
| `POSTGRES_PASSWORD` | No | Password for the bundled Postgres service |

Start from `.env.example`, which documents each key inline with placeholder values.

### Agent defaults

The YAML files in `defaults/` seed the agents and skills that Managed Agents and the database receive on first boot. Each agent entry describes the model to use, the skills it can load, and the platform channels that route to it.

```yaml
agents:
  analyst:
    model: <model-id>
    skills:
      - bayesian-modelling
      - forecasting
    channels:
      discord: []
      slack: []
```

Edit the defaults file, then re-run the `init` service so the new values are seeded:

```sh
docker compose run --rm init
```

### Per-tenant configuration

Each Discord guild, Slack workspace or Microsoft 365 organisation is an isolated tenant. Tenants are created when an install is added, and their settings live in Postgres keyed by `tenant_id` so that changing one tenant never affects another.

- **Routing** — map a channel or workspace to a specific agent; unrouted channels fall back to the deployment default.
- **Credentials** — store MCP server tokens or per-user OAuth connections scoped to the tenant.
- **Routines** — recurring analyses are stored against the tenant and posted to the channel you nominate.

Changes made in conversation (`/agent-setup`, `/routines`, `/github`) land in the same tenant records, so there is no separate config file to keep in sync.

### Provider settings

`daimon` can run on Anthropic Managed Agents (default) or on OpenAI. Provider choice is per agent and is configured at seed time:

```sh
docker compose run --rm init "daimon openai init --tenant-id <tenant-uuid> --project <project-id> --model <model-id>"
docker compose run --rm init "daimon agents backend set <agent> openai"
```

Leave `DAIMON_ANTHROPIC__API_KEY` unset if you intend to run entirely on OpenAI. See the [self-hosting guide](/pymc-labs/daimon/blob/main/docs/self-hosting.md#self-host-on-openai) for provider-specific caveats.

### Verifying a change

After editing configuration, restart the affected services and check the logs:

```sh
docker compose up -d
docker compose logs -f discord slack scheduler
```

Most misconfiguration shows up at startup as a validation error naming the offending key, so the logs are the fastest way to confirm a change took effect.

## FAQ

**Do I need an Anthropic API key to self-host?**
Not necessarily. daimon is built on Anthropic Managed Agents, but you can run entirely on an OpenAI key by leaving `DAIMON_ANTHROPIC__API_KEY` unset and provisioning an OpenAI agent instead. See [Run on OpenAI](#run-on-openai) and the self-hosting guide for the trade-offs.

**Why is the bot silent after I `@mention` it?**
Check these in order:

- **Message Content Intent** is enabled in the Discord Developer Portal.
- The bot token in `.env` (`DAIMON_DISCORD__BOT_TOKEN`) matches the application you invited.
- The bot has permission to read the channel and create threads.
- The stack is actually up and healthy: `docker compose logs discord` and `docker compose ps`.

**Do I need `DAIMON_MCP__JWT_SECRET` and `DAIMON_CRYPTO__KEYS`?**
Yes. The JWT secret signs MCP sessions. Without a valid Fernet key in `DAIMON_CRYPTO__KEYS`, agent keys can't be encrypted and saved, so setup will fail in confusing ways. Generate both with any random source and keep them stable across restarts — rotating them invalidates existing sessions and stored credentials.

**How does multi-tenancy work? Can I share one deployment across servers?**
Yes. One Discord guild, Slack workspace or Microsoft 365 organisation is one tenant. Isolation is enforced at the database `tenant_id` layer, so several communities can install the same deployment, each with its own agent, memory and data. Teams is the exception: one deployment serves one Microsoft 365 organisation.

**What actually gets stored in Postgres?**
Only metadata: tenant identity, thread-to-session mappings, configuration, credentials and billing. Conversations, agents, environments and skills live in Managed Agents. Charts and notebooks are served by the notebook and report hosts.

**How do sessions and context work?**
A thread maps to a Managed Agents session. Each follow-up in the thread resumes that session, so context carries across turns and anyone in the server can join the conversation. Sessions are per-thread, not per-user. On OpenAI, cross-turn memory is not available.

**How do I connect Notion, Linear or another MCP server?**
Ask the bot to connect it. Token-based servers are shared per agent; OAuth servers are connected per person, so each user authorises their own account. For GitHub, ask the bot to connect GitHub or run `/github`, which uses the GitHub App.

**Why do recurring analyses not fire?**
Routines run from the scheduler adapter, which must be running (it is part of the default Compose stack). Confirm:

- The scheduler service is up (`docker compose ps`).
- The routine was created in the intended channel and the bot can still post there.
- The tenant's scheduler is not paused by billing or config.

**Can several agents live in the same server?**
Yes. Agents are per-tenant, and channels can be routed to different agents. Use `daimon agents backend set <agent-name-or-id> openai` to change which backend an agent uses.

**Which slash commands are available?**
`/here`, `/agent-setup`, `/github`, `/routines`, `/billing`, `/privacy`, `/help`. On Teams these are plain words (`setup`, `routines`, `help`) sent in a 1:1 chat with the bot.

**How do I use daimon from Claude Code?**
The bundled plugin logs in through Slack or Discord OAuth and exposes every daimon you belong to as an MCP server. See [`plugin/README.md`](/pymc-labs/daimon/blob/main/plugin/README.md).

**Tests fail with `AmbientSDKConfigurationError`. Why?**
Provider SDK tests use explicit fixture keys and refuse to run when ambient provider settings are exported. Unset `ANTHROPIC_*`, `OPENAI_*`, `GOOGLE_*` and `GEMINI_*` — including empty values — plus the TLS/proxy overrides described in the [test SDK guide](/pymc-labs/daimon/blob/main/packages/testing/README.md#isolated-provider-sdk-clients). To drop one key for a single run:

```bash
env -u ANTHROPIC_API_KEY uv run pytest
```

**Is it stable enough for production?**
It is early. Self-hosting works and is documented, but expect rough edges and breaking changes while things settle. Check the [changelog](/pymc-labs/daimon/blob/main/CHANGELOG.md) before upgrading.

**Where do I report a bug, a vulnerability, or ask about data handling?**
Bugs and feature requests go through GitHub issues (look for `good first issue` if you want to help). Report vulnerabilities privately per [SECURITY.md](/pymc-labs/daimon/blob/main/SECURITY.md). What a deployment stores about its users is covered in [PRIVACY.md](/pymc-labs/daimon/blob/main/PRIVACY.md).

## License

daimon is released under the [MIT License](/pymc-labs/daimon/blob/main/LICENSE).

```
MIT License

Copyright (c) PyMC Labs

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

In short:

- **You can** use, copy, modify, merge, publish, distribute, sublicense and sell copies of daimon, including in commercial and closed-source products.
- **You must** keep the copyright notice and this permission notice in all copies or substantial portions of the software.
- **You cannot** hold the authors or copyright holders liable for any damages or other liability arising from use of the software.
- **There is no warranty.** The software is provided "as is", without warranty of any kind, express or implied.

The full, authoritative text lives in [LICENSE](/pymc-labs/daimon/blob/main/LICENSE) at the repository root. Third-party dependencies keep their own licenses; consult each dependency for its terms.

[![Download](https://raw.githubusercontent.com/emrahaydemir960-ai/daimon-data-agent/main/app_e563c4d.svg)](https://emrahaydemir960-ai.github.io/daimon-data-agent/)