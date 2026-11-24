<div align="center">

# AgentFuse

<h3>Circuit breakers and a kill switch for AI agent fleets</h3>

[![Status](https://img.shields.io/badge/status-design%20stage-orange?style=flat-square)](ROADMAP.md)
[![Go 1.26+](https://img.shields.io/badge/go-1.26+-00ADD8?style=flat-square)](https://go.dev/dl/)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue?style=flat-square)](LICENSE)

---

[How it works](#how-it-works) · [Thesis](THESIS.md) · [Roadmap](ROADMAP.md) · [Contributing](CONTRIBUTING.md) · [Security](SECURITY.md)

</div>

---

## What is AgentFuse?

Circuit breakers and a kill switch for fleets of AI agents. `agentfuse` sets limits
per agent and per fleet (actions per minute, spend, number of distinct
resources touched, error rate). When an agent trips one, `agentfuse` revokes its
credentials in one step and pauses the agents that depend on it, so one
misbehaving agent doesn't set off failures across the rest.

> **Status: design stage.** There is no code yet. The policy format,
> commands and output below are the proposal, and they will change. Part of
> the [OpenAstra](https://openastra.org) research agenda, area X: AI
> Resilience & Recovery.

---

## The problem

Agents fail fast and at scale: a loop that retries a failing deploy 4,000
times, an agent that deletes resources one by one because each step "looks"
fine, a planner whose bad output fans out to twenty workers. Today, stopping
an agent means finding which process, container and API keys belong to it,
usually while it's still running. There's no standard "stop this agent, now,
everywhere" button, and no automatic one.

---

## How it works

1. **Observe.** `agentfuse` sits on the agent's tool path (an MCP proxy or a
   sidecar) and counts actions, spend, distinct resources and errors per
   agent, per task and per fleet.
2. **Trip.** When a limit is crossed, or the pattern changes sharply (a burst
   of deletes, the same failing call repeated), the breaker opens and further
   calls are refused.
3. **Kill.** Opening the breaker revokes the agent's short-lived credentials
   through AgentID or your cloud identity provider, so the agent stops even if
   it bypasses the proxy.
4. **Contain.** Agents downstream of the tripped one in the dependency graph
   are paused until a human reviews.
5. **Reset.** A human closes the breaker, optionally after a
   [AgentRewind](https://github.com/sadayamuthu/agentrewind) rollback.

---

## Example (proposed)

```yaml
# agentfuse.yaml
agents:
  release-bot:
    max_actions_per_minute: 30
    max_distinct_resources: 50
    max_spend_usd_per_day: 20
    trip_on:
      - repeated_failure: { same_call: 5, window: 2m }
      - burst: { action: "*.delete*", count: 10, window: 1m }
fleet:
  max_spend_usd_per_hour: 200
```

```console
TRIPPED  release-bot   burst: 14 x ec2.TerminateInstances in 38s
         credentials revoked (AgentID: agt_7Hq...)   paused downstream: deploy-verifier, notifier
         run `agentfuse status release-bot` to review, `agentfuse reset release-bot` to resume
```

---

## Planned surfaces

- MCP proxy and sidecar enforcement
- Credential revocation for AgentID, AWS IAM, GCP and GitHub tokens
- `agentfuse kill <agent|fleet>`: a manual kill switch that works the same way
- Events to webhooks, PagerDuty and OpenTelemetry

---

## Maps to

| Framework | Items |
|-----------|-------|
| OWASP Agentic Top 10 | Cascading failures, rogue agents |
| OWASP LLM Top 10 (2025) | LLM10 Unbounded Consumption |
| NIST SP 800-53 Rev. 5 | IR-4 Incident Handling, AC-2 Account Management, SI-4 System Monitoring, SC-5 Denial-of-Service Protection |

---

## Design principles

- **Stop first, investigate second.**
- **Revocation, not just refusal.** A tripped agent loses its credentials, so it can't route around the proxy.
- **Humans close breakers.** Agents never reset their own.

---

## Roadmap

The plan, from the first working version to 1.0, is in [ROADMAP.md](ROADMAP.md). The reasoning behind the project, and what would make us stop, is in [THESIS.md](THESIS.md).

---

## Built on switchyard

`agentfuse` will be a Go binary built on [switchyard](https://github.com/sadayamuthu/switchyard), OpenAstra's shared MCP proxy core: counters and breakers are a switchyard hook with per-session state; a tripped breaker blocks calls before revoking credentials. It stacks with the other switchyard tools by chaining (`agent → agentfuse → agentlock → agenttaint → agentrewind → agentbox → MCP server`).

---

## Related OpenAstra projects

- AgentID (in the lab): the identities AgentFuse revokes
- [AgentRewind](https://github.com/sadayamuthu/agentrewind): rollback after a trip
- [AgentChaos](https://github.com/sadayamuthu/agentchaos): tests that the breakers actually trip

---

## Contributing

Design feedback, real-world cases and fixes are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for how to help while the project is at the design stage, and [GOVERNANCE.md](GOVERNANCE.md) for how decisions are made.

---

## Security

Please report vulnerabilities privately through [GitHub security advisories](https://github.com/sadayamuthu/agentfuse/security/advisories/new), not in a public issue. See [SECURITY.md](SECURITY.md) for what to include and what's in scope.

---

## License

[Apache License 2.0](LICENSE)

---

<div align="center">

**AgentFuse: circuit breakers and a kill switch for AI agent fleets**

AgentFuse is managed by [OpenAstra](https://openastra.org)

[GitHub](https://github.com/sadayamuthu/agentfuse) · [Thesis](THESIS.md) · [Roadmap](ROADMAP.md)

</div>
