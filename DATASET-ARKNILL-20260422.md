# ArkNill Primary Dataset Report (2026-04-22)

> Status: Published
> Primary source: `~/.cc-relay/usage.db` (mitmproxy TLS intercept → SQLite)
> Supplementary: seanGSISG (Issue #3, 215K calls), cnighswonger (Issue #3/#4, 101K calls)

## Methodology

This document uses ArkNill's proxy dataset as the **authoritative primary source**. Each dataset has different characteristics — different collection method, time range, plan tier, and granularity. Rather than treating them as equals, this document extracts findings from our own data first, then uses the other two to **validate, extend, or challenge** those findings.

| Role | Dataset | Strength | Limitation |
|------|---------|----------|------------|
| **Primary** | ArkNill (45.8K, Max 20x) | TLS-level capture, ratelimit headers, request bodies, cache diagnostics, budget events | Apr-only (no before-data), single plan tier |
| **Temporal extension** | seanGSISG (215K, Max 20x) | 5 months (Dec–Apr), counterfactual proof, reproducible scripts | JSONL-only (no headers, no request bodies) |
| **Tier comparison** | cnighswonger (101K, Max 5x) | Different plan tier, interceptor on/off, workload-type controlled, Jan data | Interceptor modifies requests (not raw capture) |

---

## 1. ArkNill Dataset Profile

### 1.1 Scale

| Metric | Value |
|--------|-------|
| Total requests captured | 45,884 |
| Claude model requests | 41,602 |
| Opus requests | 31,535 (75.8%) |
| Haiku requests | 10,067 (24.2%) |
| Date range | 2026-04-01 08:55 – 2026-04-22 02:08 (21 days) |
| Unique sessions | 320 |
| Requests with ratelimit headers | 37,363 (89.8%) |
| Total visible tokens | 5.22 billion |

### 1.2 Token Composition

| Category | Tokens | Share |
|----------|--------|-------|
| cache_read | 5,059,406,625 | **96.96%** |
| cache_creation | 99,890,766 | 1.91% |
| input_tokens | 36,544,950 | 0.70% |
| output_tokens | 22,200,581 | 0.43% |

cache_read dominates at 96.96%. Per-week Opus-only breakdown: W1 97.7%, W2 97.8%, W3 98.4%, W4 97.6%.

**Cross-check note on cache_read share gap:** seanGSISG reports 89.8–95.2% vs ArkNill's 96.96%. The gap is explained by: (1) seanGSISG's dataset spans 5 months including December, which had higher cold-start ratios and lower cache_read share; (2) ArkNill's Apr-only dataset is inherently biased toward warm sessions (21 days into active proxy use); (3) cnighswonger's 97–99% on multi-agent sessions shows the same Apr-period elevated rate. The figures are not contradictory — they reflect different measurement windows.

### 1.3 Weekly Breakdown

| Week | Total | Opus | Haiku | cache_read (M) | cache_create (M) | output (M) | Hit rate | Opus avg CR | Opus avg output |
|------|-------|------|-------|---------------|-----------------|------------|----------|------------|----------------|
| Apr 01–06 | 10,242 | 7,522 | 2,720 | 1,137.6 | 26.5 | 5.0 | 97.72% | 143,120 | 509 |
| Apr 07–13 | 17,344 | 12,907 | 4,437 | 1,847.5 | 37.5 | 9.7 | 98.01% | 137,036 | 613 |
| Apr 14–16 | 8,516 | 7,294 | 1,222 | 1,414.4 | 20.4 | 4.4 | 98.58% | 189,975 | 528 |
| Apr 17–22 | 5,501 | 3,813 | 1,688 | 660.3 | 15.5 | 3.1 | 97.71% | 166,318 | 607 |

W3 (Apr 14–16) shows peak cache_hit_rate (98.58%) and highest Opus avg cache_read (189,975) — this was the period of deepest sustained sessions before 4.7 disruption. W4 (Apr 17–22) shows reduced volume as activity shifted to 4.7 investigation and multi-provider work.

### 1.4 Auxiliary Tables

| Table | Rows | Unique to ArkNill |
|-------|------|-------------------|
| budget_events (B4/B5) | 167,818 | ✅ No other dataset captures these (see §5) |
| cache_diagnostics | 18,163 | ✅ Per-request fingerprint, system blocks, tool ordering |
| microcompact_events | 5,500 | ✅ Microcompact clearing records |
| intercept_events | 0 | (Not active in current config) |
| delegations | 137 | ✅ Multi-CLI delegation (Codex/Gemini) |

---

## 2. Q5h Utilization Analysis (ArkNill Primary)

**Methodology note on per-1% utilization:** ArkNill's "1.5–2.1M" range (in [02_RATELIMIT-HEADERS.md §3](02_RATELIMIT-HEADERS.md)) was calculated by dividing total visible tokens by observed Q5h percentage within individual 5h windows — not by regressing across consecutive request deltas. The range reflects window-to-window variance. Direct DB replication requires the same windowed methodology; naive `SUM(cache_read) / SUM(q5h_delta)` across individual requests produces incorrect results due to window boundary resets and non-monotonic utilization paths.

### 2.1 Daily Q5h Peak (Opus, from ratelimit headers)

| Date | Opus calls | Max Q5h% | Avg Q5h% | Max Q7d% |
|------|-----------|----------|----------|----------|
| Apr 04 | 1,187 | 39.0% | 16.2% | 16.0% |
| Apr 05 | 1,191 | 26.0% | 11.2% | 25.0% |
| Apr 06 | 1,412 | 37.0% | 13.3% | 36.0% |
| Apr 07 | 2,259 | 38.0% | 16.4% | 52.0% |
| Apr 08 | 2,850 | **92.0%** | 33.1% | 76.0% |
| Apr 09 | 3,247 | 79.0% | 25.6% | **96.0%** |
| Apr 10 | 1,556 | 59.0% | 22.1% | 99.0% |
| Apr 11 | 102 | 27.0% | 24.4% | 21.0% |
| Apr 13 | 2,891 | 69.0% | 30.3% | 44.0% |
| Apr 14 | 2,044 | 30.0% | 12.4% | 59.0% |
| Apr 15 | 4,025 | 60.0% | 25.4% | 79.0% |
| Apr 16 | 1,224 | 52.0% | 22.3% | 89.0% |
| Apr 17 | 694 | 38.0% | 14.9% | 8.0% |
| Apr 19 | 286 | 22.0% | 10.7% | 12.0% |
| Apr 20 | 1,158 | 25.0% | 10.0% | 21.0% |
| Apr 21 | 1,324 | 40.0% | 15.1% | 33.0% |
| Apr 22 | 351 | 38.0% | 26.3% | 36.0% |

Peak: Apr 08 at **92.0% Q5h** with 2,850 Opus calls. Q7d peaked at **99.0%** on Apr 10.

Note: Q7d reset on Apr 17 (8.0%) correlates with Anthropic's rate limit expansion for Opus 4.7 rollout (confirmed by cnighswonger, gateway Issue #6).

### 2.2 Q5h Distribution (Opus, 27,802 requests with Q5h headers)

| Q5h Bucket | Calls | Avg cache_read | Avg output | % of total |
|------------|-------|----------------|------------|------------|
| 0–25% | 18,267 | 152,695 | 549 | 65.9% |
| 25–50% | 7,711 | 165,030 | 613 | 27.7% |
| 50–80% | 1,655 | 129,876 | 653 | 6.0% |
| 80–100% | 169 | 113,976 | 803 | 0.6% |
| 100%+ | 0 | — | — | 0.0% |

Max 20x budget is larger than Max 5x (exact ratio undisclosed by Anthropic; price is 2x but token budget scaling may differ), explaining zero 100%+ events despite higher absolute token volumes.

**Cross-check (cnighswonger, Max 5x):** 33 windows >80% and 21 windows >100% in April alone. Comparable cache patterns under the smaller Max 5x budget produce rate-limiting that Max 20x absorbs.

### 2.3 Q7d Distribution (Opus, from ratelimit headers)

| Q7d Bucket | Calls | Avg cache_read |
|------------|-------|----------------|
| 0–25% | 6,804 | 144,713 |
| 25–50% | 7,677 | 138,431 |
| 50–80% | 9,598 | 159,330 |
| 80–100% | 3,756 | 193,474 |

Q7d reached 99.0% on Apr 10 (see §2.1), then reset to 8.0% on Apr 17 — correlated with Anthropic's rate limit expansion for Opus 4.7 rollout. Unlike Q5h, Q7d spending is more evenly distributed across buckets, with the 50–80% bucket being the largest (34.5%). The 80–100% bucket shows the highest avg cache_read (193K), suggesting heavy sessions push Q7d harder as accumulated context grows over days.

---

## 3. Cache Behavior Analysis (ArkNill Primary)

### 3.1 Cache Hit Rate: 98.06%

| Metric | ArkNill | Validation |
|--------|---------|------------|
| Overall hit rate | **98.06%** | — |
| Override env hit rate | 97.08% (24,694 turns post-Apr 10) | cnighswonger interceptor-on: 99.4% (higher — interceptor does more) |
| Stock env hit rate | 96.00% (4,357 turns post-Apr 10) | cnighswonger no-interceptor: 96.6–98.2% (consistent) |

Note: "turns" = conversation turns from JSONL analysis ([15_ENV-BREAKDOWN.md](15_ENV-BREAKDOWN.md)), not API requests from proxy DB. The total 285K+ turns referenced in Issue #4 includes all sessions across the full proxy observation period; the 97.08%/96.00% comparison uses the post-Apr-10 subset for controlled comparison.
| Override – stock delta | **+1.08pp** | cnighswonger delta: +1–3pp (consistent) |

### 3.2 Cache Diagnostics (unique to ArkNill)

From 18,163 cache diagnostic records:

| Metric | Value | Implication |
|--------|-------|-------------|
| Tools reordered | **15.1%** of requests | CC non-deterministically reorders tools between turns |
| TTL injected | **27.7%** of requests | Override environment injects 1h TTL on applicable requests |

**No cross-check available.** seanGSISG and cnighswonger's JSONL-based analysis cannot measure per-request tool ordering or TTL injection. This is a unique contribution of the mitmproxy capture approach.

### 3.3 Cache Rebuild Over Session Lifetime (ArkNill, 121 sessions ≥50 Opus turns)

| Session quartile | cache_creation / (cache_read + cache_creation) |
|-----------------|------------------------------------------------|
| Q1 (start) | **3.93%** |
| Q4 (end) | **1.01%** |

~4x drop. Sessions that survive long enough show improving cache efficiency.

**Cross-check (cnighswonger Issue #4, weekly):** Cache Agent (interceptor) 99.2% → 99.9%, Sim Agent (no interceptor) 93.1% → 99.1%. Directionally consistent — the trend toward stabilization is reproducible across datasets.

---

## 4. Model Substitution (ArkNill Primary)

### 4.1 Zero Mismatches on Max 20x

- 36,956 requests cross-checked as of Apr 19 (see [13_PROXY-DATA.md §2](13_PROXY-DATA.md)); current count: **41,306** requests with both `model` and `raw_usage.model` present
- Method: request `model` field vs `raw_usage.model` in API response JSON
- **Zero mismatches** across all checked requests

### 4.2 Haiku Profile (subagent, not spoofing)

| Metric | Haiku | Opus | Ratio |
|--------|-------|------|-------|
| Avg cache_read | 19,336 | 154,265 | **8.0x smaller** |
| Avg request_body_bytes | 68,827 | 525,351 | **7.6x smaller** |
| Avg output_tokens | 426 | 568 | 0.75x |
| Share of total calls | 24.23% | 75.77% | — |

Haiku calls have 8x smaller request bodies and 8x less cache_read — consistent with CC's Explore subagent (designed as "fast agent" for codebase search, not full reasoning).

**Cross-check (cnighswonger 4/18):** Confirmed from Sim Agent JSONL transcripts — Explore subagent runs on `claude-haiku-4-5-20251001` by default. Agent 1 (general-purpose) = Opus, Agent 2 (Explore) = Haiku. This is CC design, not server-side model substitution.

**Cross-check (cnighswonger Issue #4):** Sim Agent reached 760K cache_read with zero Haiku spoofing. fgrosswig saw Haiku at 587K on Pro tier. Spoofing is tier-dependent (Pro yes, Max no) and possibly intensity-dependent (sustained burst vs intermittent).

---

## 5. Budget Events (ArkNill Exclusive)

### 5.1 Budget Events (B4 — Tool Result Truncation)

| Metric | Value |
|--------|-------|
| Total events | 167,818 |
| Tool type | 100% `tool_result` |
| Truncated | **167,818 (100.0%)** |
| Avg content_chars after truncation | 24 |
| Unique sessions affected | All active sessions |

Every captured budget event is a tool_result that was truncated before being sent to the API. The average remaining content (24 chars) indicates aggressive truncation — effectively stripping tool output to minimal stubs. This is CC's B4 budget mechanism limiting how much tool output is included in the prompt.

### 5.2 Microcompact Events (B5 — Content Clearing)

| Metric | Value |
|--------|-------|
| Total events | 5,500 |
| Sessions affected | 37 |
| Avg tool_results cleared per event | 3.4 |
| Max cleared in single event | 22 |
| Avg total tool_results in context | 129.3 |
| Avg message count at time of event | 255 |

Microcompact events occur in longer sessions (avg 255 messages at trigger) and clear a small number of tool results per event (avg 3.4 out of 129.3 present). The max of 22 cleared in a single event suggests occasional aggressive pruning in very large contexts.

**Context:** Both B4 and B5 events are invisible in JSONL — they occur at request construction level before the API call. The mitmproxy intercept captures them because it sees the full request body including truncated/cleared content. No other dataset in this cross-validation has this granularity.

---

## 6. fallback-percentage (ArkNill Primary)

| Metric | Value |
|--------|-------|
| Requests with header | 37,363 |
| Value = 0.5 | **37,363 (100.0%)** |
| Variance | **Zero** |

**Cross-check (cnighswonger, Max 5x):** 14,000+ calls, 0.5, zero variance. Confirmed invariant across Max 5x and Max 20x.

---

## 7. Temporal Gaps Filled by Supplementary Datasets

ArkNill's dataset starts April 1. The following findings depend entirely on supplementary data:

### 7.1 Before-Data (seanGSISG, Dec 2025 – Mar 2026)

| Finding | Source | Value | ArkNill can verify? |
|---------|--------|-------|---------------------|
| cache_read weight was 0x before ~Mar 6 | seanGSISG counterfactual | 0 days >180M under 0x, 18 under 1x | ❌ No (proxy started Apr 4) |
| Max 5h window under 0x formula | seanGSISG | 21.2M = 11.8% | ❌ No |
| CacheRead per 1% (Feb) | seanGSISG | 1.62–1.72M | ✅ Apr data consistent (1.5–2.1M) |
| Iterations grew 7.7% → 52.4% | seanGSISG | Feb–Apr trend | ❌ No (Apr-only, already saturated) |

### 7.2 January Baseline (cnighswonger, unique)

| Finding | Value |
|---------|-------|
| Jan 2026 calls | 474 |
| Max window under 0x | 1.2% |
| Max window under 1x | 13.8% |
| Multiplier | >20x |

Even at lowest volume, the weight change produces >20x multiplier. **Only cnighswonger has this data point.**

### 7.3 Tier Comparison (cnighswonger, Max 5x vs ArkNill Max 20x)

| Metric | ArkNill (Max 20x) | cnighswonger (Max 5x) | Implication |
|--------|--------------------|-----------------------|-------------|
| Q5h peak (Apr) | 92.0% | 284.9% | Comparable cache patterns under different budgets |
| Windows >100% | 0 | 21 (Apr) | Max 20x absorbs cost that rate-limits Max 5x |
| cache_read per 1% | 1.5–2.1M | 1.67–1.77M | Budget scale differs, per-1% is consistent |
| Cache hit rate | 98.06% | 96.6–99.4% | Overlapping range |
| Model substitution | 0 / 41,306 | 0 / 14,000+ | Both clean |

**Key insight:** Comparable cache-dominated token profiles produce dramatically different user experiences across tiers. Max 20x absorbs the cost; Max 5x hits the wall. Note: these are different users with different workloads — the comparison is structural (same cache mechanism, different budget headroom), not workload-matched.

---

## 8. Opus 4.7 Assessment (Supplementary Data)

ArkNill's proxy DB does not contain Opus 4.7 traffic (ZBook remains on 4.6 per advisory decision). All 4.7 findings come from supplementary sources:

| Finding | Source | Data | Status |
|---------|--------|------|--------|
| 2.4x Q5h burn rate (averaged) | cnighswonger | 230 calls, Phase 1 (thinking ON) = 12.9%/hr | ✅ Directional (small n) |
| 3.3x reduction with DISABLE_ADAPTIVE_THINKING | cnighswonger | Phase 2 = 3.9%/hr, 93 calls | ✅ Directional (small n) |
| Per-call output 2.61x | seanGSISG | 2,615 4.7 calls vs 32,299 4.6 calls | ⚠ Retracted (workload confounded) |
| Per-call quota 1.23x | seanGSISG | Same comparison | ⚠ Retracted |
| Iter-call share +20.9pp | seanGSISG | 57.9% → 78.8% | ⚠ Possibly workload-driven |
| Model pin bypass | cnighswonger | settings.json ignored on v2.1.111 | ✅ Stands (filed #49503) |

**ArkNill's position:** 4.7 cost increase is a pricing/architecture decision, not a client-fixable bug. Workload-matched comparison would require running identical tasks on 4.6 and 4.7 back-to-back — feasible with our proxy but not prioritized (the result wouldn't change the structural conclusion).

---

## 9. What Only ArkNill Can Measure

| Capability | Why unique |
|------------|-----------|
| **Per-request ratelimit headers** | mitmproxy captures response headers; JSONL doesn't contain them |
| **Request body bytes** | Exact payload size per request; not in JSONL |
| **Cache diagnostics** (fingerprint, tool order, system blocks) | Requires intercepting the full request before it hits the API |
| **Budget events** (B4/B5 truncation) | Occurs at request construction level; invisible in API response |
| **Microcompact events** | Same — request-level instrumentation |
| **Multi-CLI delegation** | CC + Codex + Gemini routing; single-vendor datasets can't see this |
| **Real-time Q5h/Q7d from headers** | Per-request utilization tracking; JSONL only has post-hoc session summaries |

---

## 10. What ArkNill Cannot Measure (Depends on Others)

| Gap | Filled by |
|-----|-----------|
| Pre-April data (transition detection) | seanGSISG (Dec 2025 – Mar 2026) |
| January baseline | cnighswonger (474 calls) |
| Interceptor on/off controlled comparison | cnighswonger (4 sessions) |
| Workload-type effect on cold starts | cnighswonger (research 1/442 vs coding 1/39) |
| Opus 4.7 burn rate | cnighswonger (Phase 1/2) |
| Tier difference (Max 5x behavior) | cnighswonger |
| Iterations growth trend (Feb→Apr) | seanGSISG (secular 7.7% → 52.4%) |
| Counterfactual proof (0x vs 1x) | seanGSISG (21.2M = 11.8% under 0x) |

---

> This document is the ArkNill-primary companion to CROSS-VALIDATION-20260422.md.
> CROSS-VALIDATION treats all datasets as equal peers; this document uses ArkNill as the anchor.
