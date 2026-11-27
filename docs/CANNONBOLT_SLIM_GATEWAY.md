# CannonBolt slim-gateway spike

Branch: `cannonbolt/slim-gateway`  
Base commit: `4de5c9b65f4cd1f966dbdf09c9d6ff510baa97a1`  
Measured on Windows with Electron `ELECTRON_RUN_AS_NODE=1`.

## Build

The repository-supported minimal profile was used without changing upstream source:

```powershell
$env:OMNIROUTE_BUILD_PROFILE = "minimal"
$env:NODE_ENV = "production"
node scripts/build/build-next-isolated.mjs
```

The npm `build:secure` script uses POSIX inline environment syntax and does not run directly under Windows `cmd.exe`; the equivalent process-local PowerShell/cmd environment was used above.

Measured output:

| Artifact | Size |
|---|---:|
| `.build/next/standalone` unpacked | 1,711,818,717 bytes (~1.60 GiB) |
| `omniroute-slim-gateway.zip` | 517,644,619 bytes (~494 MiB) |
| Generated pages | 673 |

The minimal profile removes/stubs selected privileged integrations and disables dashboard/feature paths at build time, but the supported standalone assembler still co-locates the full traced runtime, native assets, compression workers, SQLite/vector assets, and supporting server wrappers.

## Runtime measurements

Electron `45.5.0` was launched with `ELECTRON_RUN_AS_NODE=1` against `.build/next/standalone/server.js`.

```text
health: HTTP 200
cold start: 6,844 ms
RSS at health readiness: 427,876,352 bytes (~408 MiB)
```

The runtime emitted a production warning that `STORAGE_ENCRYPTION_KEY` was unset and credentials would use plaintext passthrough in the disposable test data directory. It also reported the known embedded WebSocket port conflicts on 20131/20132.

## Decision gate

**NOT MET.** Addendum A.2 requires both `<=400 MB` unpacked and `<=300 MiB` idle RSS. The supported minimal build measured ~1.60 GiB unpacked and ~408 MiB RSS. Size is ~4.1x the maximum and RSS is ~1.36x the maximum.

## 9Router vs thin custom gateway

No 9Router production build measurement is claimed in this spike: the public `decolua/9router` repository was cloned only for source/package inspection, and its current package is a full Next.js application (`9router-app` v0.5.123) with a `next build --webpack` pipeline, not a previously demonstrated small sidecar. Building it would require a separate dependency install/build run and does not produce a valid measured comparison yet.

| Option | Evidence/measurement | Strengths | Costs/risks |
|---|---|---|---|
| OmniRoute minimal profile | 1,711,818,717 bytes unpacked; 517,644,619-byte zip; 6.844s / ~408 MiB Electron RSS | Existing provider catalog, quota/usage APIs, translations, routing/failover, compression, SQLite/vector support, Gemini proof | Still far above size/RAM gate; broad dependency/native surface; dashboard/full-stack trace remains |
| 9Router | No real build measurement in this spike; inspected package is another Next.js app with `next build --webpack` and custom server | MIT lineage, potentially simpler provider/router surface; worth a controlled feasibility build | Unknown output/RSS/API compatibility; no basis to claim it is lighter until built and probed |
| Thin custom CannonBolt Gateway | No build measurement yet; design option only | Could target OpenAI-compatible `/v1/models` + chat/SSE/tools, Gemini key fallback, encrypted local credential store, failover, and usage endpoints under the gate | New code/security/ToS/provider maintenance; loses OmniRoute’s broad catalog, dashboard, compression, and mature provider adapters unless rebuilt |

Recommendation: do not bundle OmniRoute as-is. Run a separate 9Router feasibility build, but plan the thin custom gateway as the likely route if 9Router also fails the size/RSS gate. Preserve the OpenAI-compatible contract so the Kilo/CannonBolt agent boundary remains stable.

## Gemini/HTTP proof status

The full Gemini normal/stream/tool proof was completed earlier against the same pinned OmniRoute code and is recorded in the parent CannonBolt repository. This slim build reached health, but `/v1/models` returned HTTP 401 in the production standalone launch because the standalone bootstrap did not honor the anonymous development setting; the production auth/bootstrap behavior must be corrected or explicitly configured before embedding.