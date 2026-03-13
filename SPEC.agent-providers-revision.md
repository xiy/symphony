# Symphony Spec Revision: Agent Providers

Status: Proposed revision to `SPEC.md` Draft v1

Purpose: extend Symphony's single-provider assumptions so one orchestration service can launch and
manage multiple coding agent providers, including Codex, Claude Code, Gemini, and OpenCode.

## 1. Summary

The current `SPEC.md` assumes one coding-agent integration profile centered on Codex app-server
semantics. That is a good bootstrap path, but it creates two operational limits:

- Provider outages, rate limits, or quota exhaustion can stall otherwise runnable work.
- Teams cannot route work to the provider that is best suited for a ticket, repository, or runtime
  constraint.

This revision keeps the existing Symphony shape intact while generalizing the execution layer from
"one agent runner profile" to "one normalized provider contract with multiple concrete adapters."

## 2. Goals

- Support multiple coding agent providers behind one orchestration contract.
- Preserve the existing per-issue workspace and prompt model.
- Let operators define preferred providers, routing policies, and fallback order.
- Normalize provider health, quota, and rate-limit signals so the orchestrator can make safe
  dispatch decisions.
- Preserve backward compatibility for repositories that only configure Codex.
- Produce a spec that can be decomposed into implementation tickets without reworking `SPEC.md`
  again first.

## 3. Non-Goals

- Mandating identical transport protocols across providers.
- Requiring every provider to support every optional tool or approval feature.
- Running multiple providers concurrently for the same issue in one attempt.
- Defining a marketplace, billing system, or automatic provider procurement flow.
- Replacing repository-owned workflow policy with hardcoded provider selection logic.

## 4. Core Revision

### 4.1 Replace the Single-Provider Assumption

Where the current spec says "coding agent" in the execution/configuration path, the revised model is
"provider-backed coding agent session." The orchestrator still owns issue selection, workspace
management, retries, and reconciliation. The new parts are:

1. `Provider Registry`
   - Loads configured provider definitions.
   - Exposes enabled providers and capability metadata.

2. `Provider Selector`
   - Chooses a provider for a dispatch attempt using workflow policy and runtime health data.

3. `Provider Adapter`
   - Wraps one concrete provider protocol and maps it into Symphony's normalized session contract.

4. `Provider Health Tracker`
   - Stores cooldown state, recent failures, and latest quota/rate-limit telemetry per provider.

The existing `Agent Runner` becomes a thin coordinator that delegates session startup and streaming
to a selected adapter.

### 4.2 Domain Model Additions

Add these logical entities to Section 4 of `SPEC.md`.

#### 4.2.1 Provider Definition

Static configuration for one provider.

Fields:

- `id` (string)
  - Stable local identifier, for example `codex`, `claude_code`, `gemini`, `opencode`.
- `kind` (string)
  - Provider implementation family. Defaults to `id` when omitted.
- `enabled` (boolean)
- `command` (string)
  - Launch command or provider entrypoint.
- `env` (map<string, string>)
  - Provider-specific environment variables after `$VAR` resolution.
- `capabilities` (object)
  - Declares what the adapter can do after normalization.
- `weight` (integer, default `100`)
  - Relative preference when multiple providers are healthy candidates.
- `cooldown_ms` (integer, optional)
  - Time a provider stays deprioritized after a fallback-triggering failure.

#### 4.2.2 Provider Capabilities

Normalized capability flags used by routing and validation.

Fields:

- `continuation_threads` (boolean)
- `dynamic_tools` (list of strings)
- `approval_control` (boolean)
- `sandbox_control` (boolean)
- `streaming_events` (boolean)
- `rate_limit_telemetry` (boolean)
- `usage_telemetry` (boolean)
- `resume_session` (boolean)

#### 4.2.3 Provider Attempt

One provider selection inside a run attempt.

Fields:

- `provider_id`
- `selection_reason`
- `attempt_index`
- `started_at`
- `ended_at`
- `outcome`
- `failure_category` (optional)

#### 4.2.4 Live Session Extensions

Extend `Live Session` with:

- `provider_id`
- `provider_kind`
- `provider_attempt_index`
- `provider_rate_limits`
- `provider_usage`
- `provider_session_id`
  - The upstream provider's native session/thread identifier when available.

### 4.3 Configuration Revision

Section 5 and Section 6 should be revised so provider configuration is first-class.

#### 4.3.1 New Top-Level `providers` Block

Add a new front matter object:

```yaml
providers:
  default: codex
  allow_fallbacks: true
  max_provider_attempts_per_run: 3
  selection_strategy: ordered-healthy
  fallback_on:
    - provider_unavailable
    - provider_rate_limited
    - provider_quota_exhausted
    - provider_startup_failed
  entries:
    codex:
      kind: codex
      enabled: true
      command: codex app-server
      weight: 100
      capabilities:
        continuation_threads: true
        dynamic_tools: [linear_graphql]
        approval_control: true
        sandbox_control: true
        streaming_events: true
        rate_limit_telemetry: true
        usage_telemetry: true
        resume_session: true
    claude_code:
      kind: claude_code
      enabled: true
      command: claude-code app-server
      env:
        ANTHROPIC_API_KEY: $ANTHROPIC_API_KEY
      weight: 90
      capabilities:
        continuation_threads: true
        dynamic_tools: []
        approval_control: false
        sandbox_control: false
        streaming_events: true
        rate_limit_telemetry: true
        usage_telemetry: true
        resume_session: false
```

Notes:

- Command names are illustrative. The spec defines normalized behavior, not vendor CLI spelling.
- `entries` must contain at least one enabled provider after environment resolution.
- `default` must reference an enabled provider.

#### 4.3.2 Routing Policy Block

Add a routing section that lets repositories steer work intentionally:

```yaml
routing:
  required_capabilities: []
  preferred_by_label:
    claude_code: ["provider:claude", "spec-heavy"]
    codex: ["provider:codex", "tooling-heavy"]
  forbidden_by_label:
    gemini: ["provider:no-gemini"]
  per_state_order:
    Todo: [claude_code, codex]
    In Progress: [codex, claude_code]
```

Routing rules:

- Label rules are additive filters, not implicit auth checks.
- A provider may only be selected when its required capabilities cover the route's declared needs.
- If no route matches, use `providers.default`.

#### 4.3.3 Backward Compatibility

Existing `codex.*` fields should remain valid as a compatibility profile:

- If `providers.entries` is absent, synthesize one enabled provider named `codex`.
- Map `codex.command`, `codex.approval_policy`, `codex.thread_sandbox`,
  `codex.turn_sandbox_policy`, `codex.turn_timeout_ms`, `codex.read_timeout_ms`, and
  `codex.stall_timeout_ms` onto the synthesized `codex` provider profile.
- Implementations may continue storing provider-specific config under namespaced blocks like
  `codex`, `claude_code`, or `gemini`, but the normalized `providers` block is authoritative for
  selection and fallback behavior.

### 4.4 Dispatch and Selection Behavior

Revise Sections 6 through 8 so dispatch is provider-aware.

#### 4.4.1 Preflight Validation

Dispatch preflight must also validate:

- At least one provider is enabled.
- The default provider exists and is enabled.
- Each enabled provider has a non-empty `command`.
- Required auth/env variables resolve successfully for enabled providers.
- Routing rules do not reference unknown providers.
- `max_provider_attempts_per_run >= 1`.

#### 4.4.2 Provider Selection Algorithm

Before launching a worker attempt:

1. Build candidate providers from enabled `providers.entries`.
2. Filter out providers that:
   - are in cooldown,
   - are missing required auth,
   - do not satisfy route-required capabilities,
   - have already failed in this run attempt up to the configured provider attempt cap.
3. Sort remaining providers by:
   - explicit routing order,
   - health state,
   - recent rate-limit/quota signal,
   - configured weight,
   - stable provider id tiebreaker.
4. Select the top candidate and record `selection_reason`.

#### 4.4.3 Fallback Rules

Fallback is allowed only when `providers.allow_fallbacks` is true and a failure is classified as
recoverable at the provider layer.

Fallback-triggering normalized failure categories:

- `provider_unavailable`
- `provider_startup_failed`
- `provider_rate_limited`
- `provider_quota_exhausted`
- `provider_transient_transport_error`
- `provider_capability_mismatch`

Fallback must not happen for:

- invalid workspace safety checks
- malformed workflow config
- repository validation failures
- user-input-required flows when the configured policy says to fail
- deterministic tool-contract violations caused by Symphony itself

When fallback happens:

- Reuse the same issue workspace.
- Re-render the prompt only if provider-specific prompt shaping requires it.
- Increment `provider_attempt_index`.
- Record the prior provider outcome in logs and runtime state.
- Do not retry the same provider again within the same worker run unless the routing policy
  explicitly allows repeated attempts.

### 4.5 Provider Adapter Contract

Replace the single Codex-specific execution contract in Section 10 with a normalized provider
adapter contract plus provider profiles.

#### 4.5.1 Normalized Adapter Interface

Every provider adapter must implement logical operations equivalent to:

- `start_session(workspace_path, launch_config) -> provider_session`
- `start_turn(provider_session, rendered_input, turn_config) -> provider_turn`
- `read_events(provider_session) -> normalized_event_stream`
- `stop_session(provider_session)`
- `extract_usage(normalized_event_stream) -> usage`
- `extract_rate_limits(normalized_event_stream) -> rate_limits`

The spec remains normative on event ordering and normalized outcome categories, not on vendor JSON
shape.

#### 4.5.2 Required Normalized Events

All adapters must map native events into Symphony events including:

- `session_started`
- `startup_failed`
- `turn_started`
- `turn_completed`
- `turn_failed`
- `turn_cancelled`
- `turn_input_required`
- `approval_requested`
- `tool_call_requested`
- `notification`
- `malformed`

If a provider cannot emit a native equivalent for one of these, the adapter must synthesize the
normalized event from the closest native signal or explicitly document the capability as unsupported.

#### 4.5.3 Provider-Specific Profiles

The revised spec should split Section 10 into:

1. `10. Provider Adapter Contract`
   - Provider-neutral rules.
2. `10.A Codex Profile`
   - Existing app-server transcript and Codex-specific config.
3. `10.B Claude Code Profile`
   - Provider-specific launch/start/stream notes once implemented.
4. `10.C Gemini Profile`
   - Same structure.
5. `10.D OpenCode Profile`
   - Same structure.

Initial conformance bar:

- The main spec must define the neutral contract now.
- Concrete provider profile appendices may start as "minimum supported transport and capability
  expectations" until all adapters exist.

### 4.6 Observability and Runtime State

Extend Section 13 so monitoring answers "what provider ran this work and why?"

Required new log/context fields:

- `provider_id`
- `provider_kind`
- `provider_attempt_index`
- `selection_reason`
- `fallback_from_provider`
- `fallback_failure_category`
- `provider_rate_limits`
- `provider_cooldown_until`

Runtime snapshot additions:

- Healthy/unhealthy provider map
- Cooldown expiration per provider
- Recent provider failure counts by category
- Latest provider quota/rate-limit snapshot
- Active run counts per provider

### 4.7 Failure Model Revision

Extend Section 14 with provider-specific classes:

1. `provider_config_error`
2. `provider_auth_missing`
3. `provider_protocol_error`
4. `provider_unavailable`
5. `provider_rate_limited`
6. `provider_quota_exhausted`
7. `provider_capability_mismatch`

Recovery rules:

- Config/auth errors fail preflight or fail the run without retrying that provider.
- Rate-limit and quota failures open provider cooldown and allow fallback or later retry.
- Protocol errors may fallback immediately but should also increment provider health penalties.
- A run that exhausts all eligible providers should surface one normalized failure summarizing the
  attempted providers and their terminal categories.

### 4.8 Security and Tooling Implications

Provider diversity changes the trust boundary. The revised spec should explicitly state:

- Provider auth is namespaced per provider and must not be copied into prompts.
- Dynamic tool availability is provider-specific and must be advertised only when the adapter
  actually supports the tool contract.
- Approval and sandbox semantics are not portable across vendors; the workflow must declare whether
  a provider is acceptable when those controls are weaker than the Codex baseline.
- Repository operators may disable providers that do not satisfy required harness guarantees.

## 5. Recommended Normative Edits to `SPEC.md`

This revision should drive concrete edits in these sections of the main spec:

- Section 3: add `Provider Registry`, `Provider Selector`, and `Provider Health Tracker`.
- Section 4: add provider entities and extend `Live Session`.
- Section 5: add `providers` and `routing` front matter schema.
- Section 6: update validation rules and config cheat sheet.
- Sections 7-8: make dispatch and retry provider-aware.
- Section 10: split into provider-neutral contract plus provider profiles.
- Section 13: add provider observability requirements.
- Section 14: add provider failure taxonomy and fallback behavior.
- Section 17: add provider-routing, fallback, and capability tests.
- Section 18: update conformance checklist for multi-provider support.

## 6. Validation Matrix Additions

The revised Section 17 should add at least these cases:

- Config parses one provider and many providers.
- Backward-compatibility mode synthesizes `codex` provider from legacy config.
- Unknown provider references in routing fail validation.
- Dispatch skips providers missing required auth.
- Selection prefers a healthy provider in explicit route order.
- Rate-limit failure triggers cooldown and fallback to the next provider.
- Exhausting all eligible providers produces one normalized failure outcome.
- Provider-specific tool support is advertised only for supported adapters.
- Metrics/logs include provider identity on every run/session lifecycle event.

## 7. Rollout Plan

This revision is intentionally staged.

### Stage 1: Spec and Scaffolding

- Introduce normalized provider config and runtime state.
- Keep Codex as the only implemented adapter.
- Preserve existing behavior when only Codex is configured.

### Stage 2: Additional Adapters

- Add Claude Code adapter.
- Add Gemini and/or OpenCode adapters behind explicit feature flags.
- Validate fallback logic with synthetic provider failures.

### Stage 3: Smarter Routing

- Add health-weighted selection.
- Add label/state-aware routing.
- Add provider-specific prompt shaping where necessary.

## 8. Follow-On Planning Slices

This revision is detailed enough to split the implementation into separate tickets:

1. Provider config and validation model
2. Provider registry and selection engine
3. Provider-neutral runner contract
4. Codex adapter migration into the new contract
5. Claude Code adapter
6. Gemini adapter
7. OpenCode adapter
8. Observability and dashboard updates
9. Fallback and cooldown recovery tests

## 9. Open Questions

- Should provider selection remain repo-scoped, or should issue comments/metadata be allowed to
  request one-off overrides?
- Do we require all providers to support continuation turns, or can Symphony downgrade to
  single-turn workers for providers without resumable sessions?
- Should provider cooldown state survive process restarts, or is in-memory state sufficient for the
  first implementation?
- How much provider-specific prompt shaping belongs in the core spec versus provider appendices?
