# Fly Scanner - Working Reference Implementation

This repository contains the **working Fly.io scanner deployment** from July 27, 2025 (commit `aab1b84`). All modules were confirmed functional on this deployment.

## ✅ Status: WORKING

This codebase is the definitive working reference for the DealBrief security scanner. Use this as the baseline for any new implementations.

## Architecture

- **Backend**: PostgreSQL with direct SQL queries  
- **Module Execution**: Sequential processing (no hanging issues)
- **Tech Detection**: Simple, reliable approach using WebTech/headers
- **Infrastructure**: Fly.io with Docker containers
- **Frontend**: Next.js dashboard with Supabase integration

## Key Working Components

### Security Modules (31 total)
- ✅ `techStackScan` - Technology fingerprinting
- ✅ `tlsScan` - SSL/TLS security analysis  
- ✅ `spfDmarc` - Email security (SPF/DMARC/DKIM)
- ✅ `endpointDiscovery` - URL/endpoint enumeration
- ✅ `nuclei` - Vulnerability scanning
- ✅ `shodan` - Internet-wide asset discovery
- ✅ All other modules in `/apps/workers/modules/`

### Core Infrastructure
- ✅ PostgreSQL artifact storage (`apps/workers/core/artifactStore.ts`)
- ✅ Sequential worker processing (`apps/workers/worker.ts`)
- ✅ Simple, reliable subprocess execution
- ✅ Working Dockerfile with all security tools
- ✅ Fly.toml configuration for multi-process deployment

## Performance Metrics (Confirmed Working)

- **Scan Duration**: 35-90 seconds for Tier-1 scans
- **Module Success Rate**: 100% completion
- **No Hanging Issues**: Sequential execution prevents module hangs
- **Subprocess Reliability**: All tools (httpx, dig, sslscan) work correctly

## Deployment History

- **Last Deployed**: July 27, 2025 on Fly.io
- **Status**: Suspended but ready for redeployment  
- **Commit**: `aab1b84423eb560d62be0b730bc5246f51f1552e`
- **Verification**: All modules confirmed working by user

## Usage

```bash
# Install dependencies
pnpm install

# Start locally (requires PostgreSQL)
fly dev

# Deploy to Fly.io
fly deploy
```

## Differences from GCP Version

The parallel `gcpscanner` repository contains a GCP Cloud Run version with:
- ❌ Firestore backend (had persistence issues)  
- ❌ Complex concurrent execution (causes hangs)
- ❌ IPv6 DNS resolution problems
- ⚠️ Promise.race() timeout issues

**Use this flyscanner version as the working baseline.**

## API Keys Required

See `API_KEYS_REQUIRED.md` for complete list of required API keys and environment variables.

---

**Repository**: https://github.com/rrh1441/flyscanner  
**Created**: August 20, 2025  
**Reference Commit**: aab1b84 (July 27, 2025)