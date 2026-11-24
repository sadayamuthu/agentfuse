# AgentFuse: thesis

**In one line:** agents fail fast and at scale, so they need automatic
circuit breakers and a kill switch that revokes credentials, not just a
proxy that says no.

## The problem

A retry loop hits a failing deploy thousands of times; an agent deletes
resources one at a time because each step looks fine; one bad plan fans out
to twenty workers. Stopping an agent today means finding its processes,
containers and keys while it's still running.

## The bet

1. **Simple limits catch most runaway behaviour.** Rate, spend, distinct
   resources touched, error rate and bursts of destructive calls are cheap to
   count and catch most incidents early, without understanding intent.
2. **Revocation is the real stop.** Refusing calls at a proxy can be bypassed;
   revoking the agent's short-lived credentials can't.
3. **Containment needs the dependency graph.** Pausing downstream agents
   stops one failure from becoming many.
4. **Humans close breakers.** An agent never resets its own, which keeps the
   design simple and auditable.

## Why now

Agents hold real cloud and SaaS credentials, multi-agent pipelines are
becoming common, and short-lived workload identities make one-step
revocation feasible.

## What success looks like

- In AgentChaos runaway scenarios, AgentFuse trips before meaningful damage, and
  revocation stops the agent even when it bypasses the proxy.
- Default limits produce few false trips on normal workloads.
- `agentfuse kill` stops an agent or fleet in one step across AWS, GitHub and
  AgentID.

## What would prove us wrong

- Thresholds that catch runaways also trip constantly on legitimate bursts,
  so teams raise limits until they're useless.
- Credentials can't be revoked fast enough, or are shared across agents so
  revocation causes collateral outages.
- Agent platforms ship good built-in kill switches.

## Not a bet on

Diagnosing why an agent misbehaved, or undoing what it did (that's AgentRewind).

## Where it fits

Area X, AI Resilience & Recovery. Revokes AgentID identities; tested by
[AgentChaos](https://github.com/sadayamuthu/agentchaos); hands off to
[AgentRewind](https://github.com/sadayamuthu/agentrewind). Runs on the shared proxy core.
