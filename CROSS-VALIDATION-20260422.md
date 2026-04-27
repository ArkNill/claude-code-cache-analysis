# Cross-Validation Report: Three Independent Datasets Converge

> Generated: 2026-04-22
> Based on: Issue #3 (seanGSISG), Issue #4 (cnighswonger), ArkNill proxy DB

## 1. Dataset Overview

| Metric | ArkNill | seanGSISG | cnighswonger |
|--------|---------|-----------|--------------|
| **Total API calls** | 45,884 (41,602 Claude) | 215,000+ | 101,506 |
| **Plan** | Max 20x ($200/mo) | Max 20x ($200/mo) | Max 5x ($100/mo) |
| **Date range** | Apr 1 – Apr 22, 2026 | Dec 23, 2025 – Apr 21, 2026 | Jan – Apr 2026 |
| **Collection method** | mitmproxy TLS intercept → SQLite | CC session JSONL parsing | Interceptor telemetry + JSONL |
| **Model breakdown** | Opus 31,535 / Haiku 10,067 | Opus primary (4.6 + 2,615 4.7) | Opus 4.6 primary, 4.7 testing |
| **Geography** | Asia (KR) | Unknown | US (NC) |
| **Ratelimit headers** | 37,363 captured (all Claude models) | Not available (JSONL only) | Available via claude-code-meter |

## 2. Core Finding: cache_read Weight Change Confirmation

### 2.1 CacheRead per Estimated 1% Utilization

| Source | Jan | Feb | Mar | Apr |
|--------|-----|-----|-----|-----|
| ArkNill (02_RATELIMIT-HEADERS.md) | — | — | — | 1.5–2.1M |
| seanGSISG (215K dataset; per-1% from 178K non-synthetic) | — | 1.62–1.72M | 1.62–1.72M | 1.62–1.72M |
| cnighswonger (101K dataset) | 1.73M | 1.67M | 1.74M | 1.77M |

**Convergence**: All three datasets land in the **1.5–2.1M range**. Different accounts, different plans (5x vs 20x), different geographies, different collection methods — same result.

### 2.2 Quota Multiplier (0x vs 1x weight)

| Source | Jan | Feb | Mar | Apr |
|--------|-----|-----|-----|-----|
| seanGSISG | — | 9.8x | 12.3x | 14.7x |
| cnighswonger | 23.9x | 13.8x | 28.0x | 38.6x |

cnighswonger's higher multipliers correlate with multi-agent sessions (97–99% cache-read ratio). The pattern is consistent: **zero rate-limit events under 0x formula, dozens under 1x**.

Note: fgrosswig's reported 64x multiplier requires cache_read at ~98.4% of visible tokens (seanGSISG analysis of Mar 21 peak: 356.2M visible, quota(0x) = 27.4M, quota(1x) = 359.0M, multiplier = 13.1x on that day; 64x reachable at sustained 98.4%+ ratios).

### 2.3 Counterfactual: Zero Overages Under Old Formula

**seanGSISG (215K dataset, 178K non-synthetic):**

| Month | Active Days | Days > 180M quota (0x) | Days > 180M quota (1x) |
|-------|------------|----------------------|----------------------|
| Feb | 26 | 0 | 1 |
| Mar | 31 | 0 | 11 |
| Apr | 11 | 0 | 6 |

Maximum 5h window under 0x formula across entire dataset: **21.2M tokens = 11.8%** of budget. "We would never have been rate-limited under the old formula. Not in any month."

**cnighswonger January baseline (unique data point):**
- 474 calls, heaviest window: 0x = 1.2%, 1x = 13.8%. Even at low volume, >20x multiplier.

### 2.4 5-Hour Window Simulation

**seanGSISG (215K, Max 20x):**

| Month | Windows | >80% | >100% | Max Util |
|-------|---------|------|-------|----------|
| Feb | 70 | 0 | 0 | 57.4% (tumbling) / 76.7% (sliding) |
| Mar | 103 | 7 | 1 | 123.9% (tumbling) / 140.9% (sliding) |
| Apr | 47 | 1 | 1 | 156.1% (tumbling) / 195.7% (sliding) |

Sliding window (15-min step, two-pointer sweep) systematically produces higher peaks than tumbling — as predicted in ArkNill's 4/15 review. The story strengthens under the more realistic simulation.

**cnighswonger (101K, Max 5x):**

| Month | Windows | >80% | >100% | Max Util |
|-------|---------|------|-------|----------|
| Feb | 75 | 0 | 0 | 16.3% |
| Mar | 111 | 19 | 17 | 392.0% |
| Apr | 98 | 33 | 21 | 284.9% |

**ArkNill (45K, Max 20x):**

| Q5h Bucket | Calls | Avg cache_read | Avg output |
|------------|-------|----------------|------------|
| 0–25% | 18,267 | 152,695 | 549 |
| 25–50% | 7,711 | 165,030 | 613 |
| 50–80% | 1,655 | 129,876 | 653 |
| 80–100% | 169 | 113,976 | 803 |
| 100%+ | 0 | — | — |

Note: ArkNill's Max 20x budget is larger than Max 5x (exact ratio undisclosed by Anthropic; price is 2x but token budget scaling may differ), explaining zero 100%+ events despite higher absolute token volumes.

## 3. Cache Hit Rate Comparison

| Source | Overall | With interceptor | Without interceptor |
|--------|---------|-----------------|---------------------|
| ArkNill (proxy DB) | **98.06%** | — | — |
| cnighswonger Cache Agent | **99.4%** | ✅ | — |
| cnighswonger Code Agent | — | — | **98.2%** |
| cnighswonger Sim Agent | — | — | **97.9%** |
| cnighswonger E3B Agent | — | — | **96.6%** |

ArkNill's 98.06% (proxy-level, no interceptor) falls between cnighswonger's interceptor-on (99.4%) and interceptor-off (96.6–98.2%) ranges — consistent.

**ArkNill override vs stock comparison (JSONL turn-level analysis, 285K+ conversation turns — see [15_ENV-BREAKDOWN.md](15_ENV-BREAKDOWN.md) for methodology; "turns" differ from "API requests" in the proxy DB):**
- Override environment: 97.08% cache hit (24,694 turns post-Apr 10)
- Stock environment: 96.00% cache hit (4,357 turns post-Apr 10)
- Delta: +1.08pp — directionally consistent with cnighswonger's 1–3% interceptor effect

### 3.1 Cache Hit Rate Over Session Lifetime

**cnighswonger weekly progression (Issue #4):**

| Agent | Week 1 | Week 2 | Week 3 |
|-------|--------|--------|--------|
| Cache Agent (interceptor) | 99.2% | 99.5% | 99.9% |
| Sim Agent (no interceptor) | 93.1% | 96.3% | 99.1% |
| Code Agent (no interceptor) | 99.5% | 96.9%\* | 98.9% |

\*Code Agent Week 2 dip coincides with ~March 6 TTL change window.

**ArkNill cache rebuild rate (proxy DB, 121 sessions with ≥50 Opus turns):**
- Q1 (session start): 3.93% cache_creation / (cache_read + cache_creation)
- Q4 (session end): 1.01%
- ~4x drop from session start to end — sessions that survive long enough show improving cache efficiency regardless of interceptor.

### 3.2 Cold Start Frequency by Workload Type (cnighswonger Issue #4)

| Agent | Workload | Cold start freq | cache_read impact |
|-------|----------|-----------------|-------------------|
| Cache Agent | Research/writing | 1 per 442 calls | Low (text-heavy, small tool results) |
| Sim Agent | Mixed coding+sim | 1 per 90 calls | Medium (burst coding + 5-min cron gaps) |
| Code Agent | Heavy ML coding | 1 per 68 calls | High (large file reads, frequent edits) |
| E3B Agent | Intensive coding | 1 per 39 calls | Highest (short session, rapid changes) |

Coding workloads bust cache 5–11x more often than research workloads on the same account.

## 4. Model Substitution Verification

| Source | Requests checked | Mismatches | Method |
|--------|-----------------|------------|--------|
| ArkNill | 36,956 (as of Apr 19; current: 41,306) | **0** | request model vs raw_usage.model |
| cnighswonger | 14,000+ | **0** | claude-code-meter v0.3.0 |
| fgrosswig | ~500 | **76** (on Pro, ~587K cache_read) | Gateway proxy |
| cnighswonger Sim Agent | 6,568 | **0** (760K cache_read, no spoofing) | Interceptor |

Conclusion: Max 5x/20x show zero model substitution. Pro tier shows spoofing under sustained high cache_read. Request intensity pattern (not just session length) appears to be the trigger — cnighswonger's Sim Agent reached 760K cache_read with zero spoofing due to natural 5-minute gaps between bursts.

**Update (cnighswonger 4/18):** Some "Haiku bursts" are CC's built-in Explore subagent, which runs on claude-haiku-4-5-20251001 by default — not server-side model substitution.

ArkNill's Haiku distribution: 10,067 / 41,602 = **24.2%** — entirely legitimate subagent calls (8x smaller request bodies, 8x lower cache_read, zero Haiku-only sessions).

## 5. fallback-percentage Invariance

| Source | Calls checked | Value | Variance |
|--------|--------------|-------|----------|
| ArkNill (Max 20x) | 37,363 | 0.5 | **Zero** (37,363 / 37,363) |
| cnighswonger (Max 5x) | 14,000+ | 0.5 | **Zero** |

Confirmed invariant across both Max 5x and Max 20x accounts.

## 6. Opus 4.7 Impact Assessment

### 6.1 cnighswonger Burn Rate Testing (gateway proxy, Max 5x)

| Metric | Phase 1 (thinking ON) | Phase 2 (thinking OFF) |
|--------|----------------------|------------------------|
| Calls | 230 | 93 |
| Burn rate | **12.9%/hr** | **3.9%/hr** |
| Q5h/call | 0.87% | 0.75% |
| Avg output/call | 522 | 1,174 |
| Reduction | — | **3.3x** |

Note: Sample sizes are small (230 + 93 calls). These are directional findings from controlled A/B conditions, not statistically robust estimates. The 3.3x reduction is consistent with the qualitative observation but the precise ratio has wide confidence intervals.

### 6.2 seanGSISG Per-Call Comparison (215K JSONL, Max 20x)

| Metric | 4.6 | 4.7 | Ratio | Status |
|--------|-----|-----|-------|--------|
| Avg output tokens | 413 | 1,079 | **2.61x** | ⚠ RETRACTED |
| Avg cache_read | 103,632 | 124,086 | 1.20x | ⚠ RETRACTED |
| Iter-call share | 57.9% | 78.8% | +20.9pp | ⚠ Possibly workload-driven |
| Quota per call (1x) | 110,782 | 136,380 | 1.23x | ⚠ RETRACTED |

**⚠ seanGSISG self-correction (4/21):** Per-call multipliers are confounded by workload variance. 4.6 avg input varies 2–93 tokens by day within the comparison window. The aggregate is an averaging artifact. Workload-matched comparison needed for definitive 4.7 quantification.

### 6.3 Iterations ↔ Quota Correlation (seanGSISG)

| Util bucket | Windows | Mean iter_output_fraction |
|-------------|---------|--------------------------|
| 0–25% | 3,850 | 51.6% |
| 25–50% | 1,346 | 51.9% |
| 50–80% | 552 | 65.7% |
| 80–100% | 137 | 81.0% |
| 100%+ | 96 | **84.8%** |

30.9pp gap between low-util and high-util buckets. Top-10 sliding peaks since March all sit at 96–99% iter output share; the one January peak is 0% (pre-iterations-prevalence). Within-month Pearson r is modest (Feb 0.22, Mar 0.04, Apr 0.23) because iter base-rate saturates inside a single recent month. The signal lives in the secular trend: 7.7% → 52.4% of calls across Feb–Apr.

Iterations compound with the cache_read weight change — they don't compete.

### 6.4 Thinking Token Analysis (seanGSISG)

**Prevalence:** Thinking blocks appear in 1.9–36.7% of calls (varies by month).

**Quota impact (estimated):**

| Month | Gap Tokens | Gap × 5x Hypothetical Weight | % of Total Quota |
|-------|-----------|-------------------------------|-----------------|
| Feb | 0.01M | 0.03M | 0.0% |
| Mar | 0.06M | 0.29M | 0.0% |
| Apr | 0.49M | 2.44M | 0.1% |

"5x Weight" column is a hypothetical worst-case multiplier from seanGSISG's sensitivity analysis — assuming thinking tokens could be weighted at 5x against quota (unconfirmed by Anthropic). Even under this aggressive assumption, visible thinking tokens account for only 0.0–0.1% of total quota — negligible. The invisible adaptive thinking overhead (Opus 4.7) is a separate, unmeasurable factor that dwarfs the visible component.

**Tool_use token estimation (Scripts v2, 111,415 blocks):**

| Stat | Tokens |
|------|--------|
| Mean | 171 |
| p50 | 44 |
| p90 | 221 |
| p99 | 3,041 |
| Max | 23,796 |

Original fixed estimate of 80 tokens was "a poor fit for both tails." Updated per-block estimation makes thinking-token gap verdict more robust, though Feb/Dec numbers remain sensitive.

### 6.5 seanGSISG 4.7 Self-Correction Detail (4/21)

The retracted per-call comparison was confounded by **workload variance within the 4.6 window itself**:

| Date | Model | Calls | Avg input | Avg output | Avg cache_read |
|------|-------|-------|-----------|------------|----------------|
| Apr 5 | 4.6 | 5,886 | 80 | 181 | 67,639 |
| Apr 8 | 4.6 | 3,794 | 93 | 367 | 114,031 |
| Apr 10 | 4.6 | 6,287 | 14 | 354 | 74,296 |
| Apr 11 | 4.6 | 4,793 | 6 | 948 | 233,210 |
| Apr 13 | 4.6 | 115 | 2 | 729 | 41,384 |
| Apr 15 | 4.6 | 156 | 7 | 995 | 53,459 |
| Apr 16 | 4.7 | 97 | 3 | 1,518 | 81,008 |
| Apr 19 | 4.7 | 424 | 3 | 871 | 71,994 |
| Apr 20 | 4.7 | 964 | 5 | 1,177 | 103,143 |
| Apr 21 | 4.7 | 1,130 | 4 | 1,036 | 165,196 |

Per-call input on 4.6 varies **2–93 tokens** within the comparison window. By Apr 10–15 (4.6 tail), metrics were already continuous with 4.7's range. The aggregated "2.61x output" and "0.10x input" are averaging artifacts, not model effects.

**What stands:** Scripts v2 fixes, iterations correlation (30.9pp), 179K counterfactual. **What doesn't:** 4.7 per-call multiplier claims.

### 6.6 Structural Assessment

4.7's cost increase is not a client-side fixable bug:

| Factor | Fixable? | Mechanism |
|--------|----------|-----------|
| Invisible adaptive thinking tokens | **No** | Model internal, not in API usage response |
| Tokenizer inflation (+35%) | **No** | Encoding change |
| Cache metering changes | **No** | Server-side pricing policy |
| Model pin bypass (#49503) | **No** | Server routing decision |
| DISABLE_ADAPTIVE_THINKING | Workaround | Degrades to 4.6-equivalent reasoning |

## 7. Data Source Details

### ArkNill Proxy DB (`~/.cc-relay/usage.db`)

| Table | Rows | Description |
|-------|------|-------------|
| requests | 45,884 | API call records (tokens, model, latency, ratelimit headers) |
| budget_events | 167,818 | B4/B5 budget tracking (100% are tool_result truncations) |
| cache_diagnostics | 18,163 | Per-request fingerprint, system blocks, tool ordering |
| microcompact_events | 5,500 | Microcompact clearing events |
| delegations | 137 | Multi-CLI delegation records |
| session_terminals | 119 | Session-to-terminal mapping |

### ArkNill Weekly Opus Breakdown

| Period | Calls | cache_read (M) | cache_create (M) | output (M) | Hit rate |
|--------|-------|---------------|-------------------|------------|----------|
| Apr 01–06 | 7,522 | 1,076.5 | 20.9 | 3.8 | 98.10% |
| Apr 07–13 | 12,907 | 1,768.7 | 31.4 | 7.9 | 98.26% |
| Apr 14–16 | 7,294 | 1,385.7 | 18.0 | 3.9 | 98.71% |
| Apr 17–22 | 3,813 | 660.3 | 15.5 | 3.1 | 97.71% |

### seanGSISG Scripts (v2, 4/21)

6 analysis scripts at `seanGSISG/claude-code-hidden-problem-analysis/scripts/`:
- `cache-read-weight-transition.mjs` — counterfactual (0x vs 1x) analysis
- `dual-window-simulation.mjs` — sliding window (15-min step) + tumbling
- `iterations-window-correlation.mjs` — iter_output_fraction by util bucket
- `opus-47-comparison.mjs` — 4.7 vs 4.6 per-call (⚠ confounded)
- `quota-composition-breakdown.mjs` — token composition by month
- `thinking-token-estimation.mjs` — per-block tool_use estimate (mean=171, p50=44)

### cnighswonger Session Comparison (Issue #4)

| Agent | Duration | Calls | Hit rate | Cold starts | Interceptor |
|-------|----------|-------|----------|-------------|-------------|
| Cache Agent | 11 days | 4,420 | 99.4% | 10 (1/442) | Yes |
| Code Agent | 21 days | 7,911 | 98.2% | 117 (1/68) | No |
| Sim Agent | 17 days | 6,568 | 97.9% | 73 (1/90) | No |
| E3B Agent | 6 days | 467 | 96.6% | 12 (1/39) | No |

## 8. Contributors

| Contributor | Dataset | Key contribution |
|-------------|---------|-----------------|
| **ArkNill** | 45K calls, Max 20x, mitmproxy | Foundational analysis (16 documents), dual sliding window discovery, GrowthBook flag analysis |
| **seanGSISG** | 215K calls, Max 20x, JSONL | Before-data (Dec 2025), counterfactual proof, reproducible scripts, iterations correlation, self-correcting 4.7 analysis |
| **cnighswonger** | 101K calls, Max 5x, interceptor | 4-session workload comparison, Jan 2026 data, 38.6x multiplier, DISABLE_ADAPTIVE_THINKING discovery, Explore=Haiku finding |
| **fgrosswig** | ~4K calls/day, Max 5x, gateway | Session Serializer, Haiku burst live capture, 24h forensics. Referenced but not participating in Issue #3. Note: data originates from private Gitea instance; not independently reproducible from public sources |

---

> This document cross-references data from three independent measurement systems.
> It is maintained alongside the 16 numbered analysis documents in this repository.
