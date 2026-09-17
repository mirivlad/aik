# AIK Architecture

This document describes the current target architecture. It is intentionally high-level while the project is in bootstrap stage.

## 1. System overview

```text
                         Browser / PWA
                    Linux / Windows / Android
                               |
                            HTTPS/WSS
                               |
                         reverse proxy
                               |
                 +-------------+-------------+
                 |                           |
                 v                           v
              Web UI                    Assistant Core
                                             |
                                  +----------+----------+
                                  |                     |
                                  v                     v
                              Policy layer         Model Gateway
                                                        |
                                            +-----------+-----------+
                                            |           |           |
                                            v           v           v
                                          Groq       Gemini     OpenRouter
                                            ... free/cloud providers ...

                              Remote node plane

                    Home Linux ----+
                    Work Linux ----+---- outbound connection ----> Core
                    Windows -------+
                    Android companion (later)
```

## 2. Design principles

### 2.1 Models are replaceable compute backends

AIK must not make a specific LLM provider the architectural center of the system.

Conversation state, permissions, tasks, tool access, memory and routing policy belong to AIK. Models are selected execution backends and may be replaced as provider availability changes.

### 2.2 Web-first operation

The primary user interface is web-based. The UI must be under project control and modifiable for AIK-specific workflows.

User-facing settings must be editable through the web administration interface.

### 2.3 No provider secrets in routine hand-edited configuration

Provider API keys, provider enable/disable state, model-routing settings and outbound proxy configuration must be managed through the web admin UI and stored by the application.

Container bootstrap configuration may still require minimal deployment-time values where technically unavoidable, but normal operation must not depend on editing `.env` files.

### 2.4 Fail closed for explicit network policy

If a provider is configured with `use_proxy = true`, AIK must not silently retry that provider directly when the proxy is down.

This prevents accidental source-IP or routing-policy leakage.

### 2.5 Tool permissions are enforced outside the LLM

Tool safety cannot rely only on prompts.

The core must have an explicit policy layer capable of deciding whether an operation is:

- allowed automatically;
- allowed only after confirmation;
- denied.

The same principle applies to remote-node capabilities.

## 3. Model Gateway

The first implementation will reuse/fork suitable components from FreeLLMAPI rather than implementing free-provider aggregation from scratch.

Expected gateway responsibilities:

- provider credentials;
- provider enable/disable state;
- model discovery/catalog;
- unified OpenAI-compatible endpoint;
- routing between providers/models;
- quota awareness;
- retries and fallback;
- health and latency measurements;
- rate-limit handling;
- routing profiles such as fast/smart/coding where useful;
- request/usage analytics;
- outbound proxy support.

### 3.1 Initial proxy model

AIK v0.x starts with one global proxy configuration:

```text
Global outbound proxy
    enabled
    URL
```

Each provider has:

```text
enabled
use_proxy
```

Effective behavior:

```text
provider disabled
    -> no requests

provider enabled + use_proxy = false
    -> direct connection

provider enabled + use_proxy = true
    -> global proxy

provider enabled + use_proxy = true + proxy unavailable/misconfigured
    -> provider request fails; no direct fallback
```

Custom per-provider or per-model proxy URLs are explicitly deferred.

### 3.2 Proxy coverage

The proxy selection must apply consistently to all outbound HTTP(S) activity for a provider, including where applicable:

- chat/completions;
- streaming responses;
- model discovery;
- credential validation;
- health checks;
- quota/status calls;
- token/OAuth exchange.

## 4. Assistant Core

The core will be built after the model-gateway MVP is proven useful.

Planned responsibilities:

- conversation/session state;
- task planning;
- model-selection requests to the gateway;
- tool invocation;
- policy enforcement;
- memory/context retrieval;
- events and scheduled/proactive work;
- task history;
- remote-node registry;
- handoff to specialized workers.

The core should request capabilities, not hard-code model names. For example:

```text
task class: coding
reasoning: high
tools: required
latency: interactive
privacy: cloud-allowed
```

The gateway can then choose an appropriate currently available model/provider.

## 5. Remote nodes

Linux and Windows machines will run an AIK node/bridge.

Preferred network behavior:

- the node establishes an outbound authenticated connection to the Core;
- local MCP servers/tools may use stdio or local-only HTTP;
- the node publishes only explicitly enabled capabilities;
- machines do not need individually exposed public MCP endpoints.

Potential capabilities include:

```text
filesystem.read
filesystem.write
shell.exec
process.list
systemd.status
docker.inspect
docker.logs
desktop.notification
clipboard.read
clipboard.write
screen.capture
```

Capabilities and approval policies will vary by node.

Android support is expected to start as a companion rather than as a general-purpose shell node.

## 6. Specialized workers

Some expensive or difficult tasks may be escalated to external CLI agents such as Codex or Claude Code.

These workers are not the main conversational model. They are tools invoked by the Assistant Core for suitable tasks.

Typical flow:

```text
Assistant Core
    -> identifies large coding task
    -> creates bounded worker task
    -> invokes CLI worker
    -> receives diff/results/tests
    -> validates/summarizes
    -> continues normal conversation
```

## 7. Web application

The eventual web application should contain at least:

```text
Chat
Tasks
Nodes
Tools
Models
Providers
Network
Analytics
Settings
```

For the model-gateway bootstrap phase, an existing modifiable React admin/dashboard may be reused and then gradually integrated into the full AIK UI.

## 8. Deployment

The canonical deployment target is Docker/Compose.

Expected eventual service split:

```text
web
core
model-gateway
state/database
node (separate machines)
workers (optional)
```

The exact number of containers is not fixed yet; unnecessary service fragmentation should be avoided.

GitHub Releases and container images built from this repository will be the canonical release artifacts.
