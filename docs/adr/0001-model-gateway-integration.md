# ADR-0001 — Integrate FreeLLMAPI as the AIK model gateway

- **Status:** Accepted
- **Date:** 2026-09-17
- **Upstream inspected:** `tashfeenahmed/freellmapi` v0.11.0 (`955e9cf6413314d461d8130f695a81b8c5f246fe`)

## Context

AIK needs a model gateway that can aggregate free cloud LLM providers, keep track of provider/model quotas and health, route requests automatically, fail over between providers, expose a stable API to Assistant Core, and provide web administration for credentials and routing.

FreeLLMAPI already provides most of this functionality. It is an MIT-licensed Node.js monorepo with a server, React dashboard, shared types, CLI, SQLite state, an OpenAI-compatible `/v1` API, provider adapters, quota/routing logic, analytics, model catalog updates and outbound proxy support.

Reimplementing those parts before validating the usefulness of the free-model pool would duplicate a substantial amount of existing work.

## Decision

### 1. FreeLLMAPI is integrated as a separate model-gateway service

AIK will reuse the complete upstream FreeLLMAPI implementation rather than copying individual router/provider files into Assistant Core.

The intended repository layout is:

```text
aik/
├── apps/
│   └── web/                       # future unified AIK UI
├── services/
│   ├── core/                      # future Assistant Core
│   └── model-gateway/             # vendored FreeLLMAPI source
├── nodes/
│   └── agent/                     # future remote-node bridge
├── workers/
│   ├── codex/                     # future optional worker
│   └── claude/                    # future optional worker
├── deploy/
│   └── docker/
└── docs/
    └── adr/
```

The gateway remains a separately runnable service/container even though its source is kept in the same canonical AIK repository.

### 2. Vendor/subtree, not Git submodule and not a separate canonical fork

The upstream source will be imported under `services/model-gateway/` using a vendored/subtree workflow and pinned to an upstream release/commit.

AIK will not depend on a Git submodule for normal builds. A normal clone or release archive of `mirivlad/aik` must contain everything required to build the model gateway.

AIK will also not maintain a separate `mirivlad/freellmapi` repository as part of its normal development workflow. `mirivlad/aik` remains the canonical project repository.

An `UPSTREAM.md` file will record:

- upstream repository URL;
- imported upstream tag and commit;
- applicable MIT attribution/license information;
- the procedure used to refresh the vendored source;
- AIK-specific patches that must be checked when rebasing/syncing.

Upstream updates are deliberate operations, not automatic tracking of `main`.

### 3. Assistant Core talks to the gateway only through stable APIs

Assistant Core must not depend on FreeLLMAPI database tables, provider classes, router internals or encryption implementation.

The normal inference boundary is the gateway's OpenAI-compatible API:

```text
Assistant Core
    -> http://model-gateway:3001/v1
    -> provider/model selected by gateway
```

The gateway is the source of truth for:

- provider credentials;
- provider enable/disable state;
- model catalog and availability;
- quota/rate-limit state;
- provider/model health;
- model routing and fallback;
- model request analytics;
- outbound model-provider networking/proxy configuration.

Assistant Core owns:

- conversations and persistent assistant state;
- memory/context retrieval;
- tasks and planning;
- tool/MCP access;
- policy and approvals;
- remote-node registry;
- task classification;
- escalation to specialized workers such as Codex/Claude CLI.

This boundary makes it possible to replace the gateway later without rewriting AIK Core.

### 4. Use the FreeLLMAPI dashboard during the gateway bootstrap phase

The existing React dashboard will be used initially for provider keys, models, routing, proxy/network settings and model-gateway analytics.

AIK will not build a second provider-settings database or duplicate those controls in Assistant Core.

Later, when the unified AIK web application exists, its `Models`, `Providers`, `Network` and `Analytics` sections may call the gateway administration API directly. The upstream dashboard can then become an advanced/internal view or be removed from the normal navigation.

This allows immediate use of a working web administration interface without making that dashboard the permanent architecture of AIK.

## AIK network-policy changes

FreeLLMAPI already has centralized provider-bound proxy handling through `proxyFetch()`, a dashboard-configured global proxy, per-platform bypass support, HTTP/HTTPS/SOCKS support, and Docker `host.docker.internal` support. AIK will reuse that implementation but change its policy semantics.

### 5. One global proxy, provider-level choice

The AIK MVP exposes one global outbound proxy in web administration.

For each provider the UI exposes a simple effective choice:

```text
Use proxy: on/off
```

Internally the existing upstream per-platform bypass mechanism may be reused: a provider with `Use proxy = off` is in the bypass set; a provider with `Use proxy = on` is not.

There is no per-model proxy configuration in the MVP.

FreeLLMAPI's existing per-key proxy override code may remain in the vendored source to minimize upstream divergence, but AIK does not expose it as part of the normal MVP UI/configuration.

### 6. Explicit proxy routing is fail-closed

Upstream FreeLLMAPI can fall back to a direct `fetch()` if a configured proxy dispatcher cannot be created. That behavior is not acceptable for AIK when a provider is explicitly configured to use the proxy.

AIK changes the behavior to:

```text
provider.use_proxy = false
    -> direct

provider.use_proxy = true
    + global proxy configured and working
    -> proxy

provider.use_proxy = true
    + proxy missing, invalid or unavailable
    -> request fails
    -> never retry that provider directly
```

This rule applies to provider-bound traffic and prevents accidental source-IP/network-policy leakage.

The failure must be visible in gateway health/analytics and should allow the router to try another eligible provider according to normal routing rules. The same provider must not be retried directly.

### 7. Web-admin configuration is authoritative for provider egress

AIK does not use ambient host proxy discovery as an authority for provider traffic.

The normal provider-egress policy is determined by application state configured through the web admin UI. In particular, AIK must not silently change provider routing because the container/host happens to have one of these configured:

- `ALL_PROXY`;
- `HTTP_PROXY` / `HTTPS_PROXY`;
- desktop/OS proxy settings;
- `NO_PROXY` rules that would bypass an explicitly required provider proxy.

Deployment environment variables may still exist for low-level container/bootstrap concerns where technically necessary, but routine provider credentials, model routing and outbound proxy policy are application settings managed from the web UI.

### 8. Provider outbound HTTP must use the centralized transport

Provider adapters must not create ad-hoc direct network paths that bypass AIK network policy.

Tests must cover at least:

- normal chat completions;
- streaming completions;
- provider key validation;
- provider health checks;
- model discovery where applicable;
- quota/status requests where applicable;
- token/OAuth exchange for future providers that require it.

A regression test must prove that a provider marked `Use proxy = on` cannot make a successful direct request when its proxy is unavailable.

## Catalog dependency

FreeLLMAPI uses a signed catalog feed to keep provider/model metadata current. AIK will retain this mechanism for the MVP because maintaining the free-provider catalog independently is not the first project goal.

The catalog feed is nevertheless an external dependency and must not become an architectural requirement for AIK Core. If the upstream catalog service disappears or its terms become unsuitable, AIK must be able to keep operating with the locally stored catalog and later replace the catalog update mechanism.

## Licensing and attribution

FreeLLMAPI is MIT-licensed. AIK must retain the upstream license and required copyright notice for vendored/derived source.

AIK-specific code remains clearly distinguishable from upstream-derived code where practical.

## Consequences

### Positive

- We reuse a mature provider/routing/quota implementation immediately.
- The free-model pool can be tested before Assistant Core is built.
- Provider credentials remain isolated from Assistant Core.
- Existing web administration is usable from the first runnable milestone.
- Upstream fixes and new providers can still be imported deliberately.
- The model gateway remains replaceable behind a stable API boundary.

### Negative / costs

- AIK carries a substantial vendored upstream codebase.
- Upstream synchronization will occasionally require resolving conflicts with AIK-specific network-policy/UI patches.
- The upstream catalog feed is an external dependency for automatic catalog freshness.
- Until the unified AIK web app exists, the model-gateway dashboard is a temporary UI surface rather than the final product UI.

## First implementation milestone

The first runnable AIK milestone is intentionally limited to the model substrate:

1. import FreeLLMAPI v0.11.0 into `services/model-gateway/`;
2. retain its existing dashboard and OpenAI-compatible API;
3. remove/disable ambient provider-proxy overrides for normal AIK operation;
4. expose one dashboard-configured global outbound proxy;
5. expose provider `Use proxy` behavior using the existing per-platform bypass mechanism;
6. implement fail-closed proxy semantics;
7. build/run it from AIK's Docker deployment;
8. configure several free providers only through the web UI;
9. verify routing, quota handling, failover, streaming and proxy behavior in real use;
10. only then begin Assistant Core.
