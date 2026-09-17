# AIK Roadmap

This roadmap is intentionally milestone-based. Dates are not assigned yet.

## M0 — Repository bootstrap

- [x] Create canonical repository.
- [x] Add project README.
- [x] Document initial architecture.
- [ ] Choose project license for AIK-owned code.
- [ ] Decide exact FreeLLMAPI import/fork strategy and preserve upstream notices.
- [ ] Add contribution/development notes once the first code lands.

Exit condition: architectural decisions are recorded and the repository is ready for implementation.

## M1 — Model Gateway bootstrap

Goal: prove that AIK can reliably use a changing pool of free cloud LLMs before building the full assistant.

- [ ] Import/fork selected FreeLLMAPI code into this repository.
- [ ] Keep a modifiable web dashboard/admin UI.
- [ ] Verify provider credential management from the web UI.
- [ ] Verify provider enable/disable controls.
- [ ] Verify existing automatic routing/fallback behavior.
- [ ] Verify quota/rate-limit accounting.
- [ ] Verify health/latency metrics.
- [ ] Expose a stable OpenAI-compatible API for AIK.
- [ ] Add Docker/Compose deployment.

Exit condition: a fresh Docker deployment can be configured from the browser and can answer via an automatically selected free cloud model.

## M2 — Central outbound proxy policy

- [ ] Add one global outbound proxy setting in web admin.
- [ ] Add `use proxy` setting per provider.
- [ ] Apply proxy routing to all provider HTTP(S) traffic.
- [ ] Support the proxy scheme(s) required by the initial deployment.
- [ ] Add `Test proxy` diagnostics.
- [ ] Add provider connectivity diagnostics showing direct/proxy path.
- [ ] Enforce fail-closed behavior for providers configured to use the proxy.
- [ ] Test streaming, model discovery, health checks and authentication through the proxy.

Exit condition: each provider can be deliberately routed either direct or through the single configured proxy with no silent direct fallback.

## M3 — Routing validation and observability

- [ ] Measure routing quality on real workloads.
- [ ] Record provider/model success rate, latency and rate-limit failures.
- [ ] Verify fallback behavior during 429/5xx/timeouts.
- [ ] Define initial task classes (general/chat/code/reasoning/fast).
- [ ] Decide which routing decisions belong in the gateway vs Assistant Core.
- [ ] Add useful diagnostics without turning logs into permanent prompt/context storage.

Exit condition: automatic free-model routing is demonstrably useful and understandable enough to trust as AIK's model substrate.

## M4 — Assistant Core MVP

- [ ] Web chat backed by Assistant Core rather than direct gateway access.
- [ ] Conversation/session storage.
- [ ] Task planner/orchestrator.
- [ ] Capability-based model requests.
- [ ] Tool invocation framework.
- [ ] Policy/approval layer.
- [ ] Basic task history and diagnostics.

Exit condition: AIK behaves as one persistent assistant whose model backend can change transparently.

## M5 — Linux remote node

- [ ] Implement authenticated outbound node connection.
- [ ] Register node identity and capabilities.
- [ ] Bridge local MCP/stdio tools.
- [ ] Filesystem read/write capability with policy controls.
- [ ] Shell/process/systemd capability with policy controls.
- [ ] Docker inspection/log access where enabled.
- [ ] Node online/offline state in the web UI.

Exit condition: the central AIK instance can safely inspect and operate an explicitly configured remote Linux machine.

## M6 — Windows node

- [ ] Port node runtime to Windows.
- [ ] PowerShell/process/service integration.
- [ ] Filesystem and clipboard integration where appropriate.
- [ ] Apply the same central policy model.

## M7 — Specialized CLI workers

- [ ] Codex CLI worker integration.
- [ ] Claude Code worker integration if useful/available.
- [ ] Bounded task handoff and result collection.
- [ ] Repository/worktree isolation for coding tasks.
- [ ] Escalation rules and usage accounting.

## M8 — Memory, events and proactive work

- [ ] Long-term assistant memory/context store.
- [ ] Event bus.
- [ ] Scheduled tasks.
- [ ] Event-triggered agent runs.
- [ ] Notification subsystem.
- [ ] Integrations with AIK-adjacent tools where useful.

## M9 — Android companion

- [ ] Mobile-friendly/PWA experience first.
- [ ] Notifications.
- [ ] Share-to-AIK / file handoff.
- [ ] Clipboard/file/photo capabilities where useful and permission-safe.
- [ ] Decide whether a native Android companion is justified beyond PWA capabilities.

## Later / explicitly deferred

These are useful ideas but are not first-version requirements:

- GigaChat provider adapter.
- Yandex AI provider adapter.
- Multiple proxy profiles.
- Per-provider custom proxy URL.
- Per-model proxy selection.
- Complex multi-agent hierarchies before a single-agent core is proven.
