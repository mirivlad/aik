# AIK Decision Log

This file records architectural decisions made before formal ADRs are introduced.

## D-0001 — One canonical repository

**Status:** accepted

All AIK documentation, source code, deployment files and release work live in `mirivlad/aik`.

GitHub Releases from this repository are the canonical release channel. Docker images/deployment artifacts will be produced from this repository.

## D-0002 — Web is the primary interface

**Status:** accepted

AIK will use a web interface as the primary UI on Linux, Windows and mobile devices.

Existing open-source UI/dashboard code may be reused only if it can be modified to fit AIK's needs.

## D-0003 — Runtime/provider configuration belongs in web admin

**Status:** accepted

Normal user-facing configuration must be available through the web admin interface.

This includes at minimum:

- provider credentials;
- provider enable/disable state;
- proxy configuration;
- model/routing settings;
- other routine runtime settings.

Hand-editing `.env` is not an acceptable normal administration workflow.

## D-0004 — Reuse FreeLLMAPI for the first model-gateway implementation

**Status:** accepted for investigation/bootstrap

Rather than reimplementing aggregation of free cloud LLM providers, quotas, routing and fallback from scratch, AIK will first reuse/fork suitable MIT-licensed FreeLLMAPI code.

The implementation must preserve applicable upstream copyright/license notices.

This decision may be revisited if technical inspection shows that maintaining the fork would cost more than implementing the required subset independently.

## D-0005 — One global outbound proxy initially

**Status:** accepted

The first version supports one centrally configured outbound proxy.

Each provider has a `use proxy` control.

There is no per-model proxy configuration and no custom per-provider proxy URL in the initial implementation.

If a provider requires proxy routing and the proxy fails, AIK must not silently fall back to a direct connection.

## D-0006 — GigaChat and Yandex are deferred

**Status:** accepted

GigaChat and Yandex AI adapters are useful future providers but are not part of the first model-gateway milestone.

The initial goal is to validate the existing pool of supported free cloud providers and the routing layer before expanding the provider set.

## D-0007 — Prove the model pool before building the full assistant

**Status:** accepted

Development order is:

1. get the free-cloud model gateway running;
2. make provider configuration and proxy routing reliable;
3. validate automatic routing/fallback in real use;
4. then build Assistant Core around it.

This avoids spending time on agent orchestration before the model substrate is known to work well enough.

## D-0008 — Remote nodes should prefer outbound connections

**Status:** accepted at architecture level

Linux/Windows nodes should normally initiate an authenticated outbound connection to AIK Core rather than requiring a separately exposed public MCP endpoint for every machine.

Local tools/MCP servers can remain local and be bridged through the node.

## D-0009 — Tool policy is enforced outside prompts

**Status:** accepted

Potentially sensitive tool actions must be governed by application policy/approval rules, not merely by system-prompt instructions.

The intended policy outcomes are broadly:

- automatic allow;
- require user confirmation;
- deny.

## D-0010 — Specialized coding agents are workers, not the main assistant

**Status:** accepted at architecture level

External CLI agents such as Codex or Claude Code may be invoked for suitable bounded tasks, but they are treated as specialized workers/tools under AIK Core rather than as AIK's persistent identity or primary conversational state holder.
