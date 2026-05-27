# Cost and Performance Architecture

## What this folder is

The architectural decisions that drive the cost line and the latency line of an AI system. The material here is what I put in front of a leadership review when the question is: *the AI feature works, the launch is in six weeks, and finance wants a number that they can plan around — what is the cost architecture, and what are the levers that will keep it under control as usage grows?*

## The organizing principle

AI cost in 2026 is not a procurement problem and not a budget problem. It is an *engineering* problem with strong architectural levers: model tier choice, caching, routing, batching, context size, retrieval depth, streaming, prompt-compression, and request-shaping. Teams that treat cost as something to negotiate with the vendor after launch find that the cost line is essentially fixed by the architecture they shipped; teams that treat cost as an architectural concern during design find that they have 5–10x of headroom they can deploy as needed.

The same is true of latency. AI latency is not solely a function of model speed; it is a function of context length (affects time-to-first-token), retrieval depth (affects time-to-context-ready), streaming choice (affects perceived latency), tool-call architecture (affects total agent latency), and concurrency design (affects p99 under load). Most AI systems are too slow because of architectural choices, not because of the model itself.

So the patterns here treat cost and performance as a *joint architectural budget*, with explicit per-feature targets and the architectural levers that move them.

## Planned documents

- **[token-economics.md](./token-economics.md)** — The cost-per-call breakdown: input tokens, output tokens, cached tokens, tool-call tokens, embedding calls, reranker calls. The model-tier cost curves (Haiku-class vs Sonnet-class vs Opus-class, open-weight self-hosted, fine-tune). Cost-per-conversation, cost-per-user, cost-per-tenant, and cost-per-feature accounting. The cost-budget-as-SLO pattern that catches a runaway feature before it shows up in the monthly invoice.
- **[caching-tiers.md](./caching-tiers.md)** — The four tiers of AI caching and when each is right: prompt prefix cache (most providers, free or near-free, high hit rate on system prompt), response cache (exact-match, useful for FAQ-shaped workloads), semantic cache (embedding-similarity match, useful for repeated-question workloads), and retrieval cache (caches retrieval results between requests). The cache-coherency patterns and the freshness-vs-cost trade-offs.
- **tier-routing-for-cost.md** *(coming)* — The architectural pattern of routing cheap-first and escalating to more capable models only on signal (low confidence, structured-output failure, complex query class). Cost savings typically 40–70% on a tiered workload. The router-architecture choices, calibration discipline, and the eval pattern that validates that routing does not degrade quality.
- **[gpu-strategy-for-self-hosted.md](./gpu-strategy-for-self-hosted.md)** — When self-hosting open-weight inference makes sense and when it does not. GPU choices (H100, MI300X, L40S, consumer cards for inference), serving-framework choices (vLLM, TGI, SGLang, Triton), capacity planning, the cost-crossover point where self-hosting beats hosted, and the operational cost (people, on-call, drift) that often dominates the bill.
- **[latency-budgets-and-streaming.md](./latency-budgets-and-streaming.md)** — The latency-budget discipline: time-to-first-token, time-to-useful-output, total response latency, and the budget allocations across system / retrieval / model / tools / answer. When streaming is required (user-facing chat), when it is not (background processing), and the streaming-implementation patterns that survive proxies, gateways, CDNs, and SSE-versus-WebSocket choices.
- **[throughput-and-concurrency.md](./throughput-and-concurrency.md)** — Concurrency-per-instance, request-batching patterns, the model-provider rate-limit shape (RPM, TPM, both), the patterns for scaling across multiple provider tenancies (Bedrock cross-account, Azure OpenAI multi-deployment), and the queue-and-fairness disciplines for multi-tenant systems.
- **[cost-incident-playbook.md](./cost-incident-playbook.md)** — The architectural and operational moves when a cost incident is in progress (one feature is consuming 10x its expected budget). Detection (cost-spike alerts), mitigation (rate-limit, route-to-cheaper-tier, kill switch), root cause (prompt regression, retrieval bloat, agent loop, abuse), and prevention (cost-budget-as-circuit-breaker). Integration with the engineering sibling's `cost-and-finops/` folder.

## How to use this section

**If you are sizing a new AI feature**, read `token-economics.md` first. The per-call cost shape and the per-user / per-tenant aggregation pattern are the two numbers leadership will ask about, and the architecture you choose determines them.

**If your AI bill is growing faster than usage**, the diagnosis is usually in `caching-tiers.md` or `tier-routing-for-cost.md` — most teams under-cache and under-route, leaving 50%+ of cost on the table.

**If your AI feature is perceived as slow**, `latency-budgets-and-streaming.md` is the diagnostic. The fix is usually streaming + retrieval-budget-trim + a small prompt-compression pass; rarely is it model swap.

## What this section is not

- **A cost-optimization listicle.** "10 ways to reduce your AI bill" exists in abundance elsewhere. This folder addresses the *architectural* levers; the operational levers (cost monitoring, alerting, FinOps process) are in the engineering sibling's `cost-and-finops/`.
- **A GPU-purchasing guide.** Hardware procurement decisions for self-hosted inference touch infrastructure procurement, supplier contracts, and capacity planning that are outside this folder's scope. The architectural decision about whether to self-host at all is here; the hardware spec is elsewhere.
