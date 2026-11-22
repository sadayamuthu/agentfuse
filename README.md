# fuse

Circuit breakers and a kill switch for fleets of AI agents. `fuse` sets limits
per agent and per fleet (actions per minute, spend, number of distinct
resources touched, error rate). When an agent trips one, `fuse` revokes its
credentials in one step and pauses the agents that depend on it, so one
misbehaving agent doesn't set off failures across the rest.

> **Status: design stage.** There is no code yet. The policy format,
> commands and output below are the proposal, and they will change. Part of
> the [OpenAstra](https://openastra.org) research agenda, area X: AI
> Resilience & Recovery.

## The problem

Agents fail fast and at scale: a loop that retries a failing deploy 4,000
times, an agent that deletes resources one by one because each step "looks"
fine, a planner whose bad output fans out to twenty workers. Today, stopping
an agent means finding which process, container and API keys belong to it,
usually while it's still running. There's no standard "stop this agent, now,
everywhere" button, and no automatic one.

## How it works

1. **Observe.** `fuse` sits on the agent's tool path (an MCP proxy or a
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
   [Rewind](https://github.com/sadayamuthu/rewind) rollback.

## Example (proposed)

```yaml
# fuse.yaml
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
         run `fuse status release-bot` to review, `fuse reset release-bot` to resume
```

## Planned surfaces

- MCP proxy and sidecar enforcement
- Credential revocation for AgentID, AWS IAM, GCP and GitHub tokens
- `fuse kill <agent|fleet>`: a manual kill switch that works the same way
- Events to webhooks, PagerDuty and OpenTelemetry

## Maps to

| Framework | Items |
|-----------|-------|
| OWASP Agentic Top 10 | Cascading failures, rogue agents |
| OWASP LLM Top 10 (2025) | LLM10 Unbounded Consumption |
| NIST SP 800-53 Rev. 5 | IR-4 Incident Handling, AC-2 Account Management, SI-4 System Monitoring, SC-5 Denial-of-Service Protection |

## Design principles

- **Stop first, investigate second.**
- **Revocation, not just refusal.** A tripped agent loses its credentials, so it can't route around the proxy.
- **Humans close breakers.** Agents never reset their own.

## Roadmap

1. Policy format and in-proxy counters
2. Breaker logic and manual kill switch
3. Credential revocation (AgentID, AWS, GitHub)
4. Dependency-aware containment
5. Pattern-based trips

## Built on switchyard

`fuse` will be a Go binary built on [switchyard](https://github.com/sadayamuthu/switchyard), OpenAstra's shared MCP proxy core: counters and breakers are a switchyard hook with per-session state; a tripped breaker blocks calls before revoking credentials. It stacks with the other switchyard tools by chaining (`agent → fuse → intentlock → taintline → rewind → MCP server`).

## Related OpenAstra projects

- AgentID (in the lab): the identities Fuse revokes
- [Rewind](https://github.com/sadayamuthu/rewind): rollback after a trip
- [AgentChaos](https://github.com/sadayamuthu/agentchaos): tests that the breakers actually trip

## License

To be decided before the first release.
