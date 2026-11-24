# Security Policy

## Reporting a vulnerability

Please don't report security problems in a public issue, pull request or
discussion. Use
[**Report a vulnerability**](https://github.com/sadayamuthu/agentfuse/security/advisories/new)
on the repository's Security tab instead. Only the maintainers can see the
report.

Include what you can of:

- the AgentFuse version (`agentfuse version`) or commit, your Go version and OS
- how AgentFuse was used, and its configuration with secrets removed
- steps to reproduce
- what happened and what you expected

You'll get a reply when the report has been read, and again when a fix is
ready. A fix ships as a patch release; the advisory names you unless you'd
rather it didn't.

## Supported versions

AgentFuse is at the design stage and has no releases yet. Once it does,
fixes land in the latest release only, until 1.0.

## What counts

AgentFuse sits on an agent's MCP tool path, counts calls, and, when a breaker trips, revokes credentials through the identity providers you configure.

AgentFuse is a security tool, so anything that breaks one of these promises
is a security issue:

- **A tripped breaker blocks.** A call that gets through an open breaker is in scope.
- **Only humans close breakers.** A way for an agent to reset or bypass its own breaker is in scope.
- **Revocation credentials are protected.** AgentFuse's own credentials for revoking others are never logged or exposed to agents. A path that leaks them is in scope.

These are by design, and not vulnerabilities:

- Limits that are too loose won't trip; limits that are too tight will stop legitimate work. Tune them.
- Revocation is as fast as your identity provider makes it.

A wrong result that isn't covered above is a bug, not a vulnerability. Report
it in a normal issue; see [CONTRIBUTING.md](CONTRIBUTING.md).
