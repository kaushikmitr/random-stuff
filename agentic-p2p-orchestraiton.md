Agentic Orchestration on llm-d: Cost-Aware P2P KV Pulls and KV Leases
Status: draft v4 
Builds on: llm-d/llm-d#2067 (P2P KV cache sharing), the p2p-source-producer plugin in llm-d-router 
Engine ask (§6 only): a lease in the vLLM (or other model server) OffloadingConnector eviction policy

1. Summary
llm-d can now pull KV blocks from a peer pod instead of recomputing them (#2067). The pull fires when the best peer holds at least minCachedTokenDelta more cached tokens than the scheduled pod. That threshold is a hand-set constant, per model, from a benchmark run.

Four changes:

Cost-aware pull decision. Replace the constant with a formula: pull when pulling is faster than recomputing, given the pod's current load.
Pressure-aware source selection. Today every consumer pulls from the single most-cached peer. Spread pulls across all peers with enough cache.
Auto-calibration. Measure the pull constants (floor, bandwidth) in the existing Helm pre-upgrade calibration Job. No manual sweeps.
KV leases for agent tool gaps. An agent calls a tool, disappears for seconds to minutes, and returns with the same growing prefix. Changes 1 and 2 already handle returns, migration, and fan-out. The one thing they cannot fix: eviction cannot tell an idle session from an agent mid-task. A lease marks a session's offloaded blocks "evict last until time T." This is advisory, by default lease renewed by every turn for 300-second, but the  optional harness hint can set this duration.

Changes 1 to 3 are edits to EPP plugins plus calibration steps. Change 4 needs one engine change (vLLM, sglang..). The design is deliberately reactive-first: the gateway acts on what it observes, agents owe it nothing, and the lease TTL hint is the only optional client input.


2. Background: the P2P pull today
From #2067: each pod runs the OffloadingConnector with a CPU tier and serves its blocks to peers over NIXL. The EPP builds a prefix index from KV events, so it knows which pods hold which blocks. After scheduling, the p2p-source-producer plugin compares the best-cached peer against the scheduled pod; if the peer leads by minCachedTokenDelta tokens, the router names that peer on the request and the engine pulls the blocks from it. A failed transfer falls back to recompute.

Two benchmark findings drive everything below:

The pull is a recovery path, not a placement strategy. Load-only routing scatters cache so thin no peer can serve. Affinity-first placement stays; the pull catches misses (evictions, cold replicas, spillover).
Pull cost is nearly flat; recompute cost grows with length. GLM-5.2 (wide-EP, H200): pull ~1.7–2.3 s at any length, recompute ~130–144 µs/token, crossover at 13,648 tokens. gpt-oss-120b (TP=1, H200): pull floor ~49 ms, pull wins at every measured length.

So the cost model is: pull = fixed fee + per-byte charge; recompute = per-token charge. Which wins depends on prefix length and, as §3 shows, on pod load.


3. Change 1: cost-aware pull decision
3.1 The problem
minCachedTokenDelta has three problems:

Load-blind. The 13.6K crossover was measured on an idle pod. On a busy pod, recompute shares the GPU with everything in flight, so the true crossover is lower. The fixed threshold skips profitable pulls exactly when the cluster is busiest, which is when the pull helps most (−45% TTFT p90 at concurrency 32 on agentic traces).
Deployment-specific. 2,048 for gpt-oss, 16,384 for GLM. New model or hardware means a new manual sweep.
It hides a known cost model. The benchmark reports already fit floor + bandwidth vs. per-token rate. The plugin just does not use it.
3.2 The rule
Let Δ = tokens the pull saves. Pull when pulling is faster:

t₀ + Δ · b / BW   <   Δ / R_eff

t₀ is the pull's fixed cost, b the KV bytes per token, BW the transfer bandwidth (all calibrated, §5), and R_eff the pod's current effective prefill rate: R_peak discounted by the load already in flight, from metrics the EPP already has. Solving for Δ gives a dynamic threshold:

Δ* = t₀ / (1/R_eff − b/BW)

Pull when Δ > Δ*. On an idle pod, Δ* matches the measured crossover. On a busy pod, R_eff drops, Δ* shrinks, and pulls fire earlier, which is correct because the recompute would queue behind other work. If the network is slower than recompute even at full load, Δ* is infinite and the plugin never pulls, which is also correct. Formulas for R_eff and the plugin change are in Appendix B.
3.3 Multimodal: count image and video tokens separately
In the engine, an image placeholder expands to hundreds or thousands of tokens; a video to tens of thousands. The EPP's tokenizer does not expand placeholders: it sees 1 token where the engine sees 2,000.

The cached-token counts from the precise index are already in expanded token space, so Δ is trustworthy. Everything derived from the EPP's own prompt count is not: total prompt tokens, the uncached-token load charge, the recompute estimate, and the approximate prompt-hash index. Off by up to 100× for video.

Fix: track text tokens and mm tokens as two separate ledgers, via a per-model expansion function in the token-producer. Two ledgers because the recompute rates differ: recomputing an mm token also re-runs the vision encoder, so the recompute side of the rule is itemized per ledger (Appendix B). Consequence: pulls are more profitable for multimodal prefixes, so a text-derived constant is most wrong exactly where the pull saves the most.

Overlap rule for v1: prefix caching is positional, so the same image after different text is a miss even though the encoder output is identical. Count an mm item as cached only when its blocks match positionally; use content hashes only to avoid transferring the same image or video to a pod twice. Position-free sharing of encoder outputs is deferred; it needs engine support.


4. Change 2: pressure-aware source selection
Today the plugin picks the single most-cached peer. Under a hot shared prefix, the whole fleet pulls from one pod; its NIC and CPU tier are an unmeasured choke point (the hot-set benchmark moved ~139M pulled tokens through exactly this shape).

Fix: among all peers whose lead exceeds Δ* (they are all good enough, since pull time barely depends on which peer serves), pick the one with the lowest current pull pressure. The EPP can estimate pressure from the pulls it has itself directed, with no engine changes; an engine-side gauge is a later upgrade (Appendix B).

One refinement: a pod with an in-flight pull counts as pressure on its source and as a prospective source itself. This is what makes fan-out self-organize (§6).


5. Change 3: auto-calibration
The pull's cost constants (t₀, BW) are deployment-specific, and measuring them has a trap: the first pull between a fresh pod pair pays a one-time ~6 s session cost, so measurement must use a warmed pair. That makes calibration a procedure, and procedures should be automated.

We already have the pattern: the τ calibration runs as a Helm pre-upgrade Job that measures the live engine and writes a ConfigMap, with zero EPP code changes. Extend the same Job with a pull phase: seed a prefix on one pod, pull it from another at two lengths, fit the line (intercept = t₀, slope = b/BW). Multimodal models get one more step to measure the mm recompute rate. The full procedure is in Appendix C.

Operator experience: helm upgrade --set calibration.enabled=true, and the thresholds are correct for their model, hardware, and network.


6. Change 4: KV leases for agent tool gaps
Agent traffic has three recurring moments, and changes 1 to 3 already handle all of them without any client cooperation:

A mid session prompt home by themselves. A return turn's prompt starts with the whole trajectory the home pod already cached, so the prefix-cache-scorer gives that pod a huge lead and the scheduler sends the turn back there. No session tracking exists or is needed: "home" is re-derived from the prompt's content on every turn, because the trajectory is the prefix.
A hot home migrates for free. The scheduler places the return turn on a cooler pod; the dynamic pull fetches the trajectory; the session re-homes through ordinary affinity. Proactive migration would only beat this by one pull floor.
Fan-out self-organizes. N siblings sharing a prefix spread via the load scorers, and pressure-aware source selection (§4) turns the resulting pulls into an organically growing set of sources. For moderate N this is within a pull floor of explicit coordination. An explicit broadcast tree for large N is deferred, gated on eval evidence.

The replication this creates is not the cache-scattering #2067 warns against. Scattering is a fragmented cache with no complete copy anywhere; this is complete copies placed where requests needed them, the mechanism the hot-set benchmark showed is worth 2.6× throughput. Copies are demand-proportional (one exists only because a request needed it there), self-cleaning (unused copies age out by LRU), and cheap (a 6-wide fan-out of a 43K prefix is under 2% of fleet CPU tier).

One gap remains that nothing reactive can close: eviction cannot tell an idle session from an agent mid-task. Under churn, the blocks of a session returning in 30 seconds are exactly as evictable as garbage. The fix is a lease.
6.1 The lease
A lease marks a session's offloaded blocks "evict last until time T." Three design properties:

Advisory and sheddable. A preference, never a guarantee. Under real memory pressure, leased blocks are evicted too (soonest-expiring first). The tier never refuses work to honor a lease.
Renewed by every turn, released by silence. The lease rides each request; every turn refreshes it. A session that stops turning stops renewing, and expiry is the release. No release message exists.
CPU tier only. GPU eviction demotes blocks to CPU rather than destroying them, so the lease only defends the CPU copy. The engine change stays inside the OffloadingConnector's eviction policy.

Two bounds keep leases from harming other tenants: a TTL cap, and a cap on the fraction of the tier that can be leased. Past the caps, new leases are silently ignored, which is safe because leases are advisory. The EPP leases only sessions whose cached tokens exceed the idle Δ*: the blocks that are expensive to lose. Wire format, eviction order, defaults, and metrics are in Appendix B.
6.2 Choosing T
Nobody knows T exactly, and the cost asymmetry means nobody has to. Too long: a dead session holds eviction priority (not memory) for at most one TTL. Too short: behavior falls back to today's, and the §3 pull recovers it for one pull floor. So: a generous default, leaning long.

Default: 300 seconds. Covers the common tool gap, renewed every turn. Deliberately does not cover long tails ("wait for human approval"): a 30-minute gap pays one pull at return, the fair price for that much tier time. The TTL covers the mode of the gap distribution, not its maximum.
One optional hint: x-llm-d-kv-lease-ttl: <seconds>. A harness that knows its tool's typical duration may state it, clamped to the cap. Absent: default. Malformed or over budget: ignored. Safe to accept because it costs nothing to honor, is bounded by the same caps as the default, and degrades to the default.

Optional refinement: set the fleet TTL from the observed inter-turn gap distribution; the EPP already sees every gap.
6.3 Prior art
vLLM already merged a KV lease: vllm-project/vllm#41383, the NIXL heartbeat lease for P/D, where the decode engine renews the TTL of a request's blocks on the prefill node while the request waits in decode's queue. That lease is request-scoped, inside one transfer window. Ours is session-scoped, between requests, when no request exists anywhere in the system, so the renewer must be the router rather than an engine heartbeat. Same TTL concept, same connector layer, different key and renewal source. llm-d is already wiring the request-scoped version into P/D (llm-d/llm-d#1522). The deferred prefetch verb has adjacent prior art too: the on_new_request() hook from #41383 already drives eager prefetch for queued requests (vllm-project/vllm#42086, LMCache #42321); those prefetch at enqueue, while the deferred verb would prefetch before a request exists.


7. Phasing
phase
contents
touches
dependency
1
dynamic Δ* (§3), pressure-aware source (§4), auto-calibration (§5)
p2p-source-producer, Helm Job
none
2
in-flight-pull awareness (§4); session observability
EPP
none
3
leases (§6)
EPP + vLLM connector
the lease change
deferred
pull-aware placement scoring; broadcast-tree group scheduling for large N; position-free mm encoder-output sharing
scheduler math; EPP + connector; engine
eval evidence; engine support


Each phase ships value alone.
8. Evaluation
Phase 1: re-run #2067's four-arm agentic-trace grid with a fifth arm (dynamic Δ*). Success: matches or beats the static threshold at every concurrency with no hand-set constant. Also sweep a deliberately mis-set static threshold to show the failure the dynamic rule removes.
Phase 2/3: an agentic workload profile for llm-d-benchmark with tool gaps and fan-out trees from recorded traces. Metrics: return-turn TTFT vs. gap length, leases on vs. off, at churn levels that actually evict; lease shed/expiry rates; last-sibling TTFT vs. fan-out width, to find the N where explicit group scheduling would start to pay. Any new hint or coordination mechanism enters only with an eval arm showing a gap the reactive baseline cannot close.

\

