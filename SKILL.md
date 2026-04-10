---
name: pact-monitor
description: Integrate @pact-network/monitor SDK to track API provider reliability for AI agents on Solana. Use when adding API monitoring, wrapping fetch() calls with reliability tracking, or integrating with Pact Network insurance rates.
argument-hint: [action]
allowed-tools: Read Write Edit Bash(npm *) Bash(pnpm *) Bash(npx *) Bash(node *) Grep Glob
---

# Pact Network Monitor -- SDK Integration Skill

You are integrating the `@pact-network/monitor` SDK into a TypeScript/JavaScript project. This SDK wraps `fetch()` to silently record API call reliability data (latency, failures, payment headers) and sync it to the Pact Network backend for insurance rate computation.

## What Pact Network Does

Pact Network is parametric micro-insurance for AI agent API payments on Solana. It monitors API provider reliability in real-time, computes actuarially-derived insurance rates from observed failure data, and publishes those rates on a public scorecard at pactnetwork.io.

The insurance rate formula is: `max(0.001, failureRate * 1.5 + 0.001)`

Provider tiers:
- RELIABLE: failure rate < 1%
- ELEVATED: failure rate 1%-5%
- HIGH RISK: failure rate > 5%

## Golden Rule

If the monitor fails for any reason, the underlying API call MUST still succeed. The SDK enforces this internally -- every recording/sync operation is wrapped in try/catch. Never break the agent's API calls.

## Step 1: Install the SDK

```bash
npm install @pact-network/monitor
# or
pnpm add @pact-network/monitor
```

## Step 2: Initialize the Monitor

```typescript
import { pactMonitor } from "@pact-network/monitor";

const monitor = pactMonitor({
  apiKey: process.env.PACT_API_KEY,           // Required for syncing to backend
  backendUrl: "https://pactnetwork.io",       // Default
  syncEnabled: true,                           // Enable background sync
  syncIntervalMs: 30_000,                      // Sync every 30 seconds (default)
  syncBatchSize: 100,                          // Records per sync batch (default)
  latencyThresholdMs: 5_000,                   // Classify as timeout above this (default)
  storagePath: "",                             // Custom path, or default ~/.pact-monitor/records.jsonl
});
```

Minimal config (local recording only, no sync):

```typescript
const monitor = pactMonitor();
```

## Step 3: Replace fetch() Calls

Replace `fetch()` with `monitor.fetch()`. It is a drop-in replacement:

```typescript
// Before
const response = await fetch("https://api.helius.xyz/v0/addresses/...", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ jsonrpc: "2.0", method: "getBalance", params: ["..."] }),
});

// After
const response = await monitor.fetch("https://api.helius.xyz/v0/addresses/...", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ jsonrpc: "2.0", method: "getBalance", params: ["..."] }),
});
```

The third argument is optional Pact-specific options:

```typescript
const response = await monitor.fetch(url, init, {
  // Validate response shape -- mismatches classified as "schema_mismatch"
  expectedSchema: { type: "object", required: ["result"] },
  // Manual USDC amount if no x402/MPP headers (in whole USDC, e.g. 0.01)
  usdcAmount: 0.01,
});
```

## Step 4: Graceful Shutdown

Call `shutdown()` to flush pending records before the process exits:

```typescript
process.on("SIGINT", () => {
  monitor.shutdown();
  process.exit(0);
});

// Or in a server framework:
server.addHook("onClose", () => monitor.shutdown());
```

## Step 5: Read Local Stats

The SDK stores records locally and exposes stats:

```typescript
const stats = monitor.getStats();
// {
//   totalCalls: 1234,
//   failureRate: 0.023,
//   avgLatencyMs: 145,
//   byProvider: {
//     "api.helius.xyz": { calls: 500, failureRate: 0.01 },
//     "quote-api.jup.ag": { calls: 734, failureRate: 0.032 }
//   }
// }

const records = monitor.getRecords({ limit: 50, provider: "api.helius.xyz" });
```

## API Reference

### `pactMonitor(config?: PactConfig): PactMonitor`

Factory function. Returns a configured monitor instance.

### `PactConfig`

| Field | Type | Default | Description |
|---|---|---|---|
| `apiKey` | `string` | `""` | Pact Network API key for backend sync |
| `backendUrl` | `string` | `"https://pactnetwork.io"` | Backend URL |
| `syncEnabled` | `boolean` | `false` | Enable background sync to backend |
| `syncIntervalMs` | `number` | `30000` | Milliseconds between sync attempts |
| `syncBatchSize` | `number` | `100` | Max records per sync batch |
| `latencyThresholdMs` | `number` | `5000` | Latency above this is classified as timeout |
| `storagePath` | `string` | `~/.pact-monitor/records.jsonl` | Local storage path |

### `monitor.fetch(url, init?, pactOptions?): Promise<Response>`

Drop-in replacement for `fetch()`. Records call metadata silently.

**PactFetchOptions:**

| Field | Type | Description |
|---|---|---|
| `expectedSchema` | `{ type: string, required?: string[] }` | Validate response JSON shape |
| `usdcAmount` | `number` | Manual USDC payment amount (whole units, e.g. 0.01) |

### `monitor.getStats(): Stats`

Returns aggregated local stats: total calls, failure rate, avg latency, per-provider breakdown.

### `monitor.getRecords(options?): CallRecord[]`

Returns raw call records. Filter by `limit` and `provider` hostname.

### `monitor.shutdown(): void`

Flushes pending sync records and stops the background sync timer. Call before process exit.

### Classifications

The SDK classifies every API call into one of:

| Classification | Condition |
|---|---|
| `success` | 2xx status, under latency threshold, schema matches (if provided) |
| `error` | Non-2xx status or network error |
| `timeout` | 2xx status but latency exceeds threshold |
| `schema_mismatch` | 2xx status but response body doesn't match expected schema |

### Payment Header Extraction

The SDK automatically detects and extracts payment data from:

- **x402 protocol**: `PAYMENT-RESPONSE` header (base64 JSON)
- **MPP protocol**: `Payment-Receipt` header (base64 JSON)

If no headers are present but `usdcAmount` is provided, the SDK creates a manual payment record with the USDC mint address on Solana.

## Common Integration Patterns

### AI Agent Framework (LangChain, CrewAI, etc.)

Wrap the HTTP client used by your agent's tool calls:

```typescript
import { pactMonitor } from "@pact-network/monitor";

const monitor = pactMonitor({
  apiKey: process.env.PACT_API_KEY,
  syncEnabled: true,
});

// Use monitor.fetch as the HTTP client for your agent tools
async function callSolanaRpc(method: string, params: unknown[]) {
  const response = await monitor.fetch(process.env.RPC_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ jsonrpc: "2.0", id: 1, method, params }),
  }, {
    expectedSchema: { type: "object", required: ["result"] },
  });
  return response.json();
}
```

### Express/Fastify Middleware

Monitor all outbound API calls from your server:

```typescript
import { pactMonitor } from "@pact-network/monitor";

const monitor = pactMonitor({
  apiKey: process.env.PACT_API_KEY,
  syncEnabled: true,
});

// Replace global fetch (Node.js 18+)
const originalFetch = globalThis.fetch;
globalThis.fetch = (url: string | URL | Request, init?: RequestInit) => {
  return monitor.fetch(url.toString(), init);
};

// Restore on shutdown
process.on("SIGTERM", () => {
  globalThis.fetch = originalFetch;
  monitor.shutdown();
});
```

### Next.js API Route

```typescript
import { pactMonitor } from "@pact-network/monitor";

const monitor = pactMonitor({
  apiKey: process.env.PACT_API_KEY,
  syncEnabled: true,
});

export async function GET() {
  const data = await monitor.fetch("https://api.coingecko.com/api/v3/simple/price?ids=solana&vs_currencies=usd")
    .then(r => r.json());
  return Response.json(data);
}
```

### Checking Provider Reliability Before Calling

Use the public scorecard API to check a provider's risk tier before making expensive calls:

```typescript
const providers = await fetch("https://pactnetwork.io/api/v1/providers").then(r => r.json());
const helius = providers.find((p: any) => p.hostname === "api.helius.xyz");

if (helius?.tier === "HIGH_RISK") {
  console.warn(`Helius failure rate is ${(helius.failure_rate * 100).toFixed(1)}% -- consider fallback`);
}
```

## Environment Variables

| Variable | Purpose |
|---|---|
| `PACT_API_KEY` | API key for syncing records to Pact Network backend |
| `PACT_BACKEND_URL` | Override backend URL (default: https://pactnetwork.io) |

## Getting an API Key

Request an API key at pactnetwork.io or generate one if you're running the backend locally:

```bash
cd pact-monitor/packages/backend
pnpm run generate-key my-agent
```

## Troubleshooting

- **Records not syncing**: Ensure `syncEnabled: true` and `apiKey` is set. Check that the backend URL is reachable.
- **No local records**: Check `~/.pact-monitor/records.jsonl` exists. The SDK creates it on first write.
- **High latency classification**: Adjust `latencyThresholdMs` if your providers legitimately have higher response times.
- **Schema mismatch false positives**: Only use `expectedSchema` when you know the exact response shape. Omit it for endpoints with variable response formats.
