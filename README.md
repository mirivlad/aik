# AIK

AIK is a self-hosted personal AI assistant platform.

The project is intended to provide one central assistant core running in Docker, a web interface, access to a pool of cloud LLM providers, and remote tool access to the user's computers through node/MCP bridges.

> Status: architecture/bootstrap stage. No usable release yet.

## Goals

- One central assistant instance deployed with Docker.
- Web UI as the primary interface on desktop and mobile.
- Web administration for provider keys, routing, proxy and runtime settings.
- Pool of free cloud LLMs with automatic routing, health checking, quota awareness and fallback.
- OpenAI-compatible model gateway for the assistant core.
- Optional outbound proxy, configured centrally, with per-provider `use proxy` control.
- Remote nodes for Linux, Windows and Android-facing integrations.
- MCP/tool access controlled by an explicit policy layer rather than by prompt instructions alone.
- Optional escalation of suitable tasks to external CLI workers such as Codex or Claude Code.
- Project documentation, development, Docker images and releases live in this repository.

## Initial architecture

AIK will be developed around four main areas:

1. **Web UI / Admin** — chat, settings, providers, nodes, tools, tasks and diagnostics.
2. **Assistant Core** — conversation orchestration, planning, memory, events, policy and task execution.
3. **Model Gateway** — provider aggregation, routing, quotas, failover, metrics and proxy handling.
4. **Remote Nodes** — outbound-connected agents exposing explicitly allowed local capabilities/MCP tools.

For the first implementation of the Model Gateway, the project plans to reuse/fork suitable code from the MIT-licensed FreeLLMAPI project instead of reimplementing provider aggregation and free-model routing from scratch. Required upstream license notices will be preserved when code is imported.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/ROADMAP.md](docs/ROADMAP.md).

## Configuration principles

User-facing runtime configuration belongs in the web admin interface, not in hand-edited `.env` files.

The initial network model is deliberately simple:

- one optional global outbound proxy URL;
- each provider is independently enabled/disabled;
- each provider has a `use proxy` switch;
- if a provider is configured to use the proxy and that proxy is unavailable, AIK must fail closed for that provider rather than silently sending the request directly.

Custom per-provider/per-model proxy URLs are intentionally out of scope for the first version.

## Repository layout

The intended layout is:

```text
.
├── docs/               Architecture, decisions and roadmap
├── web/                Web UI and admin
├── core/               Assistant orchestration and policy
├── model-gateway/      Provider adapters and model routing
├── node/               Remote node/bridge
├── workers/            Optional external CLI task workers
└── deploy/             Docker/Compose deployment files
```

Directories will be added as implementation begins rather than committed as empty placeholders.

## Releases

GitHub Releases will be the canonical release channel. Container images and Docker/Compose deployment artifacts will be versioned from this repository as the implementation matures.
