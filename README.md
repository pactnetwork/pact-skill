# pact-monitor skill

Claude Code skill for integrating the `@pact-network/monitor` SDK into your project.

Pact Network is parametric micro-insurance for AI agent API payments on Solana. This skill helps your AI coding agent wrap `fetch()` calls with reliability monitoring, configure backend sync, and integrate with the Pact Network scorecard.

## Install

```bash
npx skills add solder-build/pact-skill
```

Or manually copy `SKILL.md` to your project's `.claude/commands/` directory.

## What It Does

When invoked, the skill guides your agent through:

1. Installing `@pact-network/monitor`
2. Initializing the monitor with your config
3. Replacing `fetch()` calls with `monitor.fetch()`
4. Setting up graceful shutdown
5. Configuring payment header extraction (x402/MPP)
6. Reading local reliability stats

## Usage

In Claude Code, invoke with:

```
/pact-monitor
```

The skill knows the full SDK API surface -- config options, classification logic, payment extraction, sync behavior, and common integration patterns for AI agent frameworks, Express/Fastify, and Next.js.

## Links

- [Pact Network Scorecard](https://pactnetwork.io)
- [SDK Source](https://github.com/solder-build/pact-monitor/tree/main/packages/sdk)
- [API Documentation](https://github.com/solder-build/pact-monitor)

## License

MIT
