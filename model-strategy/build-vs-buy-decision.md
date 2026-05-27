# Build vs Buy Decision

> **Audience.** Architects deciding whether to call a hosted model API, run their own GPUs, or use a managed inference layer in between. Tech leads whose CFO asked "why are we spending $200k/month on Anthropic when we could self-host?" — and who want a defensible answer. Anyone whose model strategy currently is "Anthropic for everything" and is wondering whether that's a strategy or a default. **Scope.** The *architectural* decisions: the four options in 2026 (hosted direct, managed inference, self-hosted, white-label inference); the criteria that drive the choice (cost at scale, latency, regulatory boundary, capability ceiling, fine-tune needs, vendor independence); the total cost of ownership comparison; the hybrid pattern as the common production end-state; escape paths when the choice was wrong; the "managed inference vs direct hosted" sub-decision (Bedrock vs Anthropic direct). Not the foundational frontier-vs-open-weights-vs-fine-tune choice (see [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md)). Not the per-tenant fine-tuning decision (see [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md), which is build-extreme). Not the routing patterns (see [model-routing-and-tiering.md](./model-routing-and-tiering.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

"Should we self-host?" is among the most expensively-answered questions in AI platform engineering. The cost of getting it wrong runs in either direction: a team that should have stayed on hosted spends 12 months building inference infrastructure that's slower and less reliable than what they had; a team that should have moved to self-hosted is six months from a budget crisis because their hosted spend is growing 80% quarter over quarter.

The framing the question is usually asked under is wrong. "Self-host or hosted" is a binary choice; the actual decision space is at least four options, and the right answer for most platforms is a *portfolio* — some workloads on hosted, some on self-hosted, some on managed inference layers between. The choice is per-workload, not per-platform.

The four options in 2026:

- **Hosted direct.** Call the model provider's API directly. Anthropic API, OpenAI API, Google AI API. Lowest operational overhead; provider owns inference; you pay per-token at retail rates.
- **Managed inference.** Use a cloud provider's managed AI service that hosts the model. AWS Bedrock, GCP Vertex AI, Azure OpenAI Service. Provider's hosted model on cloud infrastructure with the cloud provider's compliance posture; per-token pricing similar to direct (sometimes with volume discounts); cloud-native integration (IAM, VPC, observability).
- **White-label inference.** Specialized inference providers hosting open-weight models. Together AI, Fireworks, Replicate, Anyscale, Lepton. You pick the model; they handle the hosting; per-token pricing; multi-tenancy on their infrastructure.
- **Self-hosted.** Your own GPUs running open-weight models (Llama, Mistral, Qwen, DeepSeek). Maximum control; capital or operational cost in GPU infrastructure; you own the entire stack including model lifecycle.

Each has properties that match certain workloads and mismatch others. The architectural decision is per-workload selection, with a default leaning, and explicit criteria for when to deviate.

This document covers the decision criteria, the total cost comparison (including the hidden costs that surprise teams), the hybrid pattern that most production platforms end up with, and the escape paths when the original choice no longer fits.

This document is opinionated about four things:

1. **The default should be buy (hosted or managed inference), not build (self-hosted).** Self-hosting has substantial operational cost that's rarely justified at small or medium scale. The economics of hosted in 2026 are better than they look — provider economies of scale, batching, and ongoing model improvements deliver value the self-host route mostly misses.
2. **The "self-hosting is cheaper" intuition is usually wrong at the team's actual scale.** Per-token cost comparisons miss the GPU operational cost, the model lifecycle cost, the eval suite cost, and the team's capacity. At 1M tokens/day, self-hosting almost never wins. At 100M tokens/day, it sometimes does. Below 100M, the case is rare.
3. **Managed inference (Bedrock, Vertex, Azure OpenAI) is the default for regulated workloads.** The compliance posture inherited from the cloud provider (HIPAA-eligible, FedRAMP-authorized, BAA standard) is much easier to consume than building it yourself or negotiating with a hosted provider.
4. **Multi-vendor is a strategy, not a hedge.** Having Anthropic + OpenAI in production isn't "two vendors so we're not locked in"; it's two production dependencies with twice the operational surface. Multi-vendor is justified when each vendor's model genuinely beats the other for some workload, not because "what if Anthropic goes down."

Structure: (2) the four options in 2026; (3) the decision criteria; (4) the total cost of ownership comparison; (5) the hybrid pattern; (6) escape paths; (7) the managed-inference vs direct-hosted sub-decision; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The four options in 2026

Each has a canonical shape, a typical cost structure, and a typical operational profile.

### 2.1 Hosted direct

Direct API calls to the model provider.

**Examples.** `api.anthropic.com`, `api.openai.com`, `generativelanguage.googleapis.com`.

**What you operate.** Your application code, your API client.

**What the provider operates.** Everything else — model serving, model lifecycle, GPU infrastructure, safety filters, rate limits.

**Cost structure.** Per-token at provider's retail rate. Usually no volume commitment; pay as you go. Enterprise discounts available at high volume.

**Compliance.** Provider's BAA / DPA / DPF / etc. May meet your needs; may not (provider's compliance posture is fixed; you can't customize).

**Latency.** Typically lowest network overhead (single hop to provider's nearest endpoint).

**Operational overhead.** Lowest. Authentication, rate-limit handling, retry / fallback logic — all on your side, but no infrastructure.

**Best for.** Small-to-medium volume; standard use cases; teams without GPU ops capacity; non-regulated or compatible-regulated workloads.

### 2.2 Managed inference

The cloud provider hosts the model; you call through the cloud's API surface.

**Examples.**

- **AWS Bedrock.** Hosts Anthropic Claude, Meta Llama, AI21 Jamba, Cohere, Mistral, Stability, Amazon Nova. Integrated with IAM, VPC, CloudWatch, KMS, AWS BAA.
- **GCP Vertex AI.** Hosts Google Gemini, Anthropic Claude, Meta Llama, Mistral. Integrated with GCP IAM, VPC SC, Cloud Audit Logs.
- **Azure OpenAI Service.** Hosts GPT-4o, GPT-4, GPT-3.5. Integrated with Azure AD, Azure VNet, Azure Monitor.

**What you operate.** Your application code; cloud provider's IAM and networking config.

**What the cloud provider operates.** The inference endpoint; integration with cloud-native services. The model itself is the model provider's; the operational layer is the cloud provider's.

**Cost structure.** Per-token; similar to hosted direct (sometimes slightly higher; sometimes with cloud volume discounts).

**Compliance.** Inherits cloud provider's compliance (HIPAA, FedRAMP, ISO, SOC 2). Usually broader than the model provider's direct offering.

**Latency.** Slight additional hop through cloud's API surface; usually negligible.

**Operational overhead.** Low. Cloud-native integration reduces friction for teams already on the cloud.

**Best for.** Regulated workloads where cloud provider's compliance posture matters; teams deeply on AWS / GCP / Azure; workloads that benefit from cloud-native integration (IAM, VPC, KMS).

### 2.3 White-label inference

Specialized inference providers hosting open-weight models.

**Examples.** Together AI, Fireworks AI, Replicate, Anyscale, Lepton, Perplexity API (some models), DeepInfra.

**What you operate.** Your application code; selecting which model on the provider.

**What the provider operates.** Inference infrastructure; model serving; sometimes fine-tuning service.

**Cost structure.** Per-token; usually cheaper than hosted direct for open-weight models (you're paying for inference, not for proprietary model development).

**Compliance.** Variable. Some have HIPAA / SOC 2; many don't. Verify before regulated use.

**Latency.** Variable; depends on provider's infrastructure. Often comparable to hosted direct; sometimes faster (lighter SaaS layer).

**Operational overhead.** Low. Similar to hosted direct.

**Best for.** Open-weight models you don't want to self-host. Cost-sensitive workloads where the open-weight model meets quality requirements. Experimentation and rapid iteration.

### 2.4 Self-hosted

Your own GPU infrastructure running open-weight models.

**Examples of stack.** vLLM, TGI, Triton, SGLang on Kubernetes / Slurm / AWS EKS / GCP GKE.

**Examples of model.** Llama 3 / 4, Mistral, Qwen 2.5 / 3, DeepSeek, Phi, Gemma.

**What you operate.** Everything — GPU procurement / lease, inference server, model lifecycle, model updates, scaling, monitoring, eval, security.

**Cost structure.** Capital (or reserved-instance commit) for GPUs; operational for infrastructure team; per-call cost is near-zero at high utilization but high at low utilization.

**Compliance.** Whatever you build; can be the strongest posture (you control everything) or the weakest (you're responsible for everything).

**Latency.** Best-case lowest (no provider hop, no network to external API); worst-case worst (cold start, scaling delay, hardware issues).

**Operational overhead.** Highest. GPU ops, model lifecycle, eval, security — all on your team.

**Best for.** Very high volume; strict residency requirements; capability requirements only open-weight models meet; regulatory environments where external dependencies are forbidden.

### 2.5 The decision summary

| Option | Operational overhead | Cost (low volume) | Cost (high volume) | Compliance flex | Best for |
| --- | --- | --- | --- | --- | --- |
| Hosted direct | Lowest | Lowest | Highest | Provider's posture | Default; small-medium volume |
| Managed inference | Low | Low-medium | Medium-high | Cloud provider's posture | Regulated; cloud-native teams |
| White-label inference | Low | Low | Medium | Variable | Open-weight at cost; experimentation |
| Self-hosted | Highest | Highest | Lowest | Whatever you build | Very high volume; strict requirements |

The crossover points between options vary by workload, but the general shape: hosted is cheapest until volume is high, then self-hosted becomes economic — if the team has GPU ops capacity.

---

## 3. The decision criteria

The criteria that should drive the choice, in rough order of binding-ness.

### 3.1 Regulatory boundary

Often the binding constraint. Some regulatory postures force certain options:

- **FedRAMP High / IL5 / IL6.** Self-hosted in authorized infrastructure (AWS GovCloud, Azure Government, dedicated). Managed inference in cloud-native FedRAMP-authorized services (Bedrock in GovCloud, when models are available). Hosted direct is rarely viable for high regulatory tiers.
- **HIPAA.** Most providers have BAAs; both hosted direct and managed inference are viable. Self-hosted is fine but operationally heavy for HIPAA.
- **GDPR + EU residency.** Hosted direct (EU regional endpoints from major providers), managed inference (cloud provider's EU regions), white-label (verify provider's EU posture), or self-hosted in EU.
- **Industry-specific (FINRA, MiFID, PCI-DSS).** Often dictates managed inference or self-hosted; verify with compliance.

**Architectural implication.** If the workload is regulated, the regulatory posture narrows the option set first. Then evaluate other criteria within the narrowed set.

### 3.2 Capability requirements

Some workloads require frontier model capability that only hosted-direct or managed-inference Claude / GPT / Gemini provides. Some workloads work fine with open-weight models that white-label or self-hosted can run.

**Test.** Run the workload's eval suite against candidate models. Frontier wins by some margin; open-weight is usually within X% of frontier for many workloads in 2026.

**Architectural implication.** If frontier capability is required, the option set is hosted direct or managed inference. If open-weight is sufficient, all four options are viable; cost and operational criteria become more binding.

### 3.3 Cost at scale

The economic crossover point depends on the workload and the team's GPU ops capacity. Rough heuristics for 2026:

- **< 10M tokens/day.** Hosted direct or managed inference almost always wins. Self-host's fixed costs aren't amortized.
- **10M-100M tokens/day.** Mixed. Self-host of open-weight can win if quality is sufficient and team has GPU ops; otherwise hosted.
- **100M-1B tokens/day.** Self-host of open-weight often wins for the high-volume portion; frontier kept on hosted for quality-sensitive workloads.
- **> 1B tokens/day.** Self-host is usually right for the bulk; specialized workloads stay on hosted.

These are heuristics; specific calculations vary by model, region, GPU pricing, team's existing infrastructure.

### 3.4 Latency requirements

Hosted has provider-side network and queueing; self-host can be tuned for ultra-low latency.

- **> 1 second budget.** Any option works.
- **500ms-1s budget.** Hosted direct or managed inference; self-host viable.
- **< 500ms budget.** Self-host of small model or specialized white-label (Groq, Cerebras, etc., which run smaller models at extreme throughput).

### 3.5 Capability ceiling

What's the best a workload can do with the chosen option?

- Hosted direct: frontier capability, usually best on capability ceiling.
- Managed inference: same model as hosted direct (often); same capability ceiling.
- White-label: limited to open-weight model capability (high, but below frontier on many benchmarks).
- Self-hosted: same as white-label for open-weight; can do custom fine-tuning to lift specific capabilities.

For workloads where capability ceiling matters (clinical, legal, complex reasoning), frontier-via-hosted is usually correct.

### 3.6 Fine-tune needs

If the workload requires fine-tuning:

- Hosted direct: some providers offer fine-tuning (OpenAI, planned for Anthropic). Per-tenant fine-tunes have hosted-provider cost.
- Managed inference: some support fine-tuning (Bedrock model customization, Azure OpenAI fine-tuning).
- White-label: many support fine-tuning (Together, Fireworks).
- Self-hosted: full control over fine-tuning; LoRA, full fine-tune, distillation — all available.

Cross-link to [per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md) for the per-tenant fine-tune decision.

### 3.7 Vendor independence

How much does it matter not to be locked to one vendor?

- **Highly locked-in workloads.** Application code, prompts, eval suite all tuned to a specific model. Migration is expensive.
- **Loosely-coupled workloads.** Prompts portable across models; eval suite tests output quality not specific quirks. Migration is a release, not a project.

The "we'll multi-vendor for resilience" instinct is often more expensive than the resilience it buys. Most production AI workloads are single-vendor; the operational simplicity wins.

### 3.8 Team capacity

GPU ops, model lifecycle, eval discipline — these are skills. Teams without them shouldn't take on self-hosting without acquiring them first.

**Test.** Does the team have:
- GPU infrastructure operational experience?
- Model lifecycle discipline (training, evaluation, deployment, rollback, deprecation)?
- Eval suite construction and maintenance capability?
- On-call rotation that can troubleshoot a GPU node failure?

If "no" to any: self-hosting is a build-up; budget for the buildup before committing.

---

## 4. The total cost of ownership comparison

Per-token cost is the visible cost; it's not the total cost. The hidden costs are where wrong decisions are made.

### 4.1 Hosted direct: visible and hidden

**Visible:** per-token retail rate × volume.

**Hidden:**

- Rate-limit management (Redis-backed budget, see [integration-architecture/backpressure-and-queueing.md](../integration-architecture/backpressure-and-queueing.md)).
- Retry / fallback logic.
- Multi-region routing if applicable.
- Cost-tracking / FinOps tooling.

**Hidden cost magnitude.** Modest. ~$2k-$5k/month of engineering capacity for a medium-sized AI platform.

### 4.2 Managed inference: visible and hidden

**Visible:** per-token rate × volume (often comparable to hosted direct).

**Hidden:**

- Cloud-native integration setup (IAM, VPC, observability).
- Potential cost premium over hosted direct.
- Cloud-specific operational concerns (region availability, quota management).

**Hidden cost magnitude.** Modest. Often net-zero or net-positive (cloud-native integration saves time vs separately wiring up hosted direct).

### 4.3 White-label inference: visible and hidden

**Visible:** per-token rate × volume (often lower than hosted direct for open-weight models).

**Hidden:**

- Vendor reliability variability (some providers have less robust infrastructure than hyperscalers).
- Compliance verification per vendor.
- Model availability and version-pinning (some white-label providers update models without notice).

**Hidden cost magnitude.** Modest if the workload is non-critical. Significant if reliability is critical and the provider's track record is short.

### 4.4 Self-hosted: visible and hidden

**Visible:** GPU capital or lease + cloud egress + minor per-call infrastructure.

**Hidden:**

- GPU operational FTE (~0.5-2 FTE depending on scale).
- Model lifecycle FTE (training, evaluation, deployment, deprecation; ~0.5-1 FTE).
- Eval suite construction and maintenance (~0.25-0.5 FTE).
- On-call rotation cost.
- Cluster scaling / capacity planning overhead.
- Cold-start cost (GPU stays warm even when idle).
- Model upgrade cost (every new model version requires re-eval, re-deploy).

**Hidden cost magnitude.** Large. ~1-3 FTE for a serious self-hosted deployment ($250-750k/year loaded). GPU cost can be $50k-$500k/year depending on scale. Hidden costs often exceed visible costs.

### 4.5 The break-even analysis

Per-token cost makes self-host look cheaper at scale; total cost narrows the gap considerably. A realistic comparison:

| Volume (tokens/day) | Hosted direct cost/year | Self-hosted cost/year (visible + hidden) |
| --- | --- | --- |
| 1M | $36k | $500k+ (no break-even) |
| 10M | $360k | $600k+ (no break-even unless team already has GPU ops) |
| 100M | $3.6M | $1.5M (break-even crossed; self-host advantage) |
| 1B | $36M | $6M (significant self-host advantage) |

(Rough numbers using $0.01/1k input tokens; adjust to your provider's pricing. Volume threshold for break-even varies considerably with quality requirements, team capacity, and infrastructure utilization.)

### 4.6 The "GPUs are getting cheaper" trend

GPU economics improve over time. Cloud GPU pricing trends down ~15-30% per year on like-for-like; new GPU generations are more efficient. The self-host crossover is shifting toward lower volumes over time.

But provider economics also improve. Anthropic and OpenAI have reduced per-token pricing repeatedly. The break-even isn't stable; it moves with both sides.

### 4.7 The "growth trajectory" factor

A workload at 5M tokens/day today, growing 20% month-over-month, will be at 50M+ in a year. The decision should account for the trajectory, not the current state.

For high-growth workloads, "we'll move to self-host when we hit X" is a real decision; the team must be building capacity for it now.

---

## 5. The hybrid pattern

Most production platforms end up with a hybrid. The pattern:

### 5.1 The typical end-state

```
Workload                              Choice                           Why
─────────────────────────────────────────────────────────────────────────────────
Customer chat (medium volume)         Hosted direct                    Capability + simplicity
Clinical agent (high stakes, low vol) Managed inference (Bedrock)      Compliance posture
Document classification (high vol)    Self-hosted open-weight          Cost dominates
Embedding (high vol)                  Self-hosted (BGE / instructor)   Cost; open-weight matches frontier
Per-tenant billing-code (one tenant)  Self-hosted fine-tuned           Workload requires fine-tune
Bulk analytics (batch)                White-label or managed batch     Cost; batch tolerance
Internal copilot (low volume)         Hosted direct                    Volume too low for anything else
```

Five different options across the portfolio. Each chosen for the workload's properties.

### 5.2 The routing layer

The routing layer (see [model-routing-and-tiering.md](./model-routing-and-tiering.md)) sits in front, sending each request to the appropriate model / provider. The routing logic is workload-aware.

### 5.3 The catalogue as binding contract

The catalogue (see [model-catalogue-and-registry.md](./model-catalogue-and-registry.md)) tracks each option's entry. Compliance, cost, latency, and ownership are documented per entry. The catalogue's compliance and operational fields are different per option:

- Hosted direct: provider's BAA reference, Anthropic / OpenAI specifics.
- Managed inference: cloud provider's BAA, IAM role, VPC endpoint.
- White-label: provider's status (HIPAA? SOC 2? GDPR? — verify and record).
- Self-hosted: cluster reference, GPU SKU, ops owner.

### 5.4 The "decline to hybrid" case

Some teams are better served by sticking with one option:

- Single hosted provider when volume is small and stable.
- Single managed inference provider when deeply on one cloud.
- Single self-hosted setup when team is GPU-native and workload is uniform.

Hybrid has cost (more vendor management, more operational surface). Justify it; don't default to it.

### 5.5 The mature hybrid pattern

A mature pattern observed across regulated SaaS:

- **Default model:** hosted direct (Anthropic or managed Bedrock for Anthropic).
- **High-volume / cost-sensitive:** self-hosted open-weight.
- **Specialized:** per-workload selection (white-label for batch, self-host for fine-tune).
- **Regulated tenants:** managed inference for the cloud provider's compliance posture.
- **Multi-region:** mix per region depending on availability matrix.

This pattern emerges organically; the architecture's job is to support it cleanly (routing, catalogue, observability).

---

## 6. Escape paths

When the original choice is wrong or the situation changes, migration paths exist. They vary in cost.

### 6.1 Hosted direct → managed inference

Common path. Driven by:

- Need for cloud-native integration (IAM, VPC, observability).
- Need for cloud-provider's compliance posture.
- Volume discount on cloud's volume commit.

**Migration mechanics.** Often straightforward — the model API is similar (same Anthropic Claude API surface on Bedrock and direct). Migration is largely re-pointing endpoints and re-doing IAM.

**Migration cost.** Low. ~1-2 weeks for a typical workload.

### 6.2 Hosted direct → self-hosted

Common at high volume. Driven by:

- Cost dominates at scale.
- Need for capability only available on open-weight (fine-tune, custom).
- Strict residency.

**Migration mechanics.** Larger lift. Different model; eval suite re-runs; prompt adjustments often needed; operational infrastructure stand-up.

**Migration cost.** High. Months of work; ongoing operational commitment.

**Escape valve.** Migrate workload-by-workload, not whole-platform-at-once. Start with workloads where open-weight meets quality bar.

### 6.3 Self-hosted → hosted direct (or managed inference)

Less common but happens. Driven by:

- GPU ops capacity strained.
- Open-weight model fell behind frontier; capability gap widening.
- Strategic shift away from infrastructure ownership.

**Migration mechanics.** Different model; eval re-run; prompt port; operational simplification.

**Migration cost.** Medium. The hard work is the eval and prompt port; the operational side is "shut down the GPUs" which is simpler than standing them up.

### 6.4 White-label → managed inference or self-hosted

White-label is sometimes a stepping stone. A team using Together for cost discovers either:

- The model is so important they want managed-inference for compliance / SLA, or
- The volume is so high they should self-host.

**Migration mechanics.** Same model (often); endpoint change; eval re-run for any subtle inference differences.

**Migration cost.** Low to medium.

### 6.5 The portfolio rebalance

Sometimes the migration isn't between options for a workload but a rebalance of the portfolio: some workloads move one way, some the other.

**Pattern.** Annual or semi-annual model strategy review. Workload-by-workload, is the current choice still right? Are the criteria the same? Has the cost shifted?

**Pattern.** Trigger-based review. New compliance requirement, major provider change, significant cost growth — any can trigger a workload-specific review.

### 6.6 The hard escape: provider exit

A model provider goes bankrupt, gets acquired and shuts down, or removes a model you depend on. Real risk for white-label providers; less likely but possible for frontier providers.

**Mitigation.** Multi-region (within one provider) is cheap insurance. Multi-vendor for critical workloads (one provider + a tested fallback in the catalogue) is more expensive but worth it for high-stakes paths.

**Pattern.** Critical workloads have a documented escape plan: which alternative model would be used, what eval suite verifies it, what migration timeline.

---

## 7. The managed-inference vs direct-hosted sub-decision

When both managed inference and direct hosted offer the same model (e.g., Claude via Bedrock vs Claude via Anthropic direct), how to choose?

### 7.1 Managed inference advantages

- **Cloud-native compliance.** AWS BAA covers Bedrock; Azure BAA covers Azure OpenAI. Often simpler than negotiating with the model provider directly.
- **Cloud-native IAM.** No separate API key management; IAM roles control access.
- **Cloud-native networking.** VPC endpoints; data doesn't leave the cloud's network.
- **Cloud-native observability.** CloudWatch, Cloud Audit Logs, Azure Monitor capture API calls.
- **Cloud-native pricing.** Volume discounts via cloud commit; consolidated billing.

### 7.2 Direct hosted advantages

- **Direct relationship with model provider.** Faster access to new models (often launch on hosted direct first, then propagate to managed).
- **Provider-specific features.** Some provider features (e.g., Anthropic's prompt caching specifics, OpenAI's structured outputs) launch on direct first.
- **Lower latency (sometimes).** Fewer hops; some workloads see lower P99.
- **Pricing parity or advantage.** Sometimes cheaper than managed (no cloud markup).

### 7.3 The decision matrix

| Criterion | Managed inference | Direct hosted |
| --- | --- | --- |
| Cloud-native team | Strong fit | Friction |
| New-model speed | Lags | Leading |
| Compliance posture | Inherits cloud's | Provider's specific |
| Pricing | Often parity; sometimes premium | Often parity; sometimes cheaper |
| Operational simplicity | Cloud IAM/networking | Separate API keys / VPC config |
| Vendor flexibility | Easier to swap models within cloud | Easier to swap clouds |

### 7.4 The recommendation patterns

- **Default for cloud-native teams:** managed inference. Simpler integration; compliance inherits.
- **Default for non-cloud teams or multi-cloud:** direct hosted. Avoids cloud lock-in; consistent API across deployment environments.
- **Default for regulated:** managed inference. Cloud's compliance posture is usually easier to justify to auditors.
- **Default for early-adopter workloads:** direct hosted. New models, new features, new model versions.

### 7.5 Both, sometimes

Some teams use both. The same model via Bedrock for the regulated workload; via Anthropic direct for the non-regulated workload that benefits from the new feature.

This requires:
- Two catalogue entries for the same underlying model.
- Routing logic that picks the right endpoint per workload.
- Two cost streams to reconcile.

Justified when the trade-offs differ by workload. Avoid when the trade-offs are similar (don't multi-source for the sake of it).

---

## 8. Worked Meridian example

Meridian's portfolio illustrates the hybrid pattern and the decision history.

### 8.1 The starting state (2024)

Single option: OpenAI hosted direct, GPT-4o. The Patient API AI-Assist used it for chat. All compliance via OpenAI's standard BAA. Monthly spend: $12k. Team size: 3 engineers, no GPU ops.

The choice was right for the stage. Operational simplicity; capability sufficient; cost manageable.

### 8.2 The Care Coordinator decision (2025)

When Meridian built the Care Coordinator agent, the model decision was reconsidered:

- Volume projection: 200k agent tasks/month, ~6 LLM calls each = 1.2M calls/month, ~5000 tokens average each = 6B tokens/month. At GPT-4o's rate: ~$120k/month projected.
- Quality requirement: clinical reasoning; high stakes; needed best available.
- Compliance requirement: BAA-covered; HIPAA-eligible; clinical evidence-base auditable.

Options evaluated:
- OpenAI direct: known; cost projection $120k/month.
- Anthropic direct: better at long-context reasoning per Meridian's eval; cost projection $90k/month at Sonnet rate.
- Managed inference (Bedrock): same Anthropic model; AWS BAA covers; cost similar to direct.
- Self-hosted Llama: cheaper but quality gap on clinical reasoning (eval showed 89% vs 95% for Sonnet).

Decision: managed inference via Bedrock, with Anthropic Claude Sonnet.

Why:
- Capability: Anthropic won the eval.
- Compliance: AWS BAA + AWS HIPAA-eligible Bedrock — Meridian's compliance team had existing AWS audit; cleaner than Anthropic direct.
- Cost: $90k/month projected; acceptable for the value.
- Operational: Meridian was already on AWS; IAM, VPC, CloudWatch all consistent with existing patterns.

### 8.3 The ingestion pipeline decision (2025)

Document ingestion was a different workload:

- Volume: 40k documents/day × ~3000 tokens = ~120M tokens/day.
- Quality requirement: classification accuracy 90%+; structured output reliable.
- Compliance: BAA-covered.

Options:
- Hosted Sonnet: $12k/day, ~$360k/month. Too expensive for the value.
- Hosted Haiku: ~$2k/day, ~$60k/month. Acceptable but still significant.
- Self-hosted Llama 3 70B: GPU cost ~$8k/month + ops ~$25k/month = ~$33k/month.
- White-label (Together) Llama 3 70B: ~$25k/month. Cheaper than self-host on the visible side.

Decision: self-hosted Llama 3 70B on AWS HIPAA-eligible infrastructure.

Why:
- Cost: half of Hosted Haiku at high volume; comparable to white-label but with full control.
- Compliance: HIPAA infrastructure controlled by Meridian's security team; no third-party-in-data-path.
- Capability: Llama 3 70B's eval result was 92% (sufficient).
- Operational: Meridian had grown to ~8 engineers including some GPU experience; the buildup was justifiable.

Trade-off: 8-week buildup of GPU infrastructure, eval suite, ops capacity. The ROI came in month 3.

### 8.4 The embeddings decision

Embeddings for retrieval were the third decision point:

- Volume: 120M documents to embed initially + ongoing 40k/day = ~120M embeddings/year initially, ~15M/year ongoing.
- Quality requirement: retrieval@10 of 0.85+.

Options:
- OpenAI text-embedding-3-large: $0.13 per 1M tokens, ~$50/month ongoing.
- Self-hosted BGE-large or instructor-xl: GPU cost ~$3k/month + ops shared with Llama cluster.

Decision: self-hosted BGE-large on the same HIPAA cluster.

Why:
- Volume: at this scale, hosted was cheaper on visible cost.
- But: PHI in embeddings was a sensitive surface; keeping embedding in-house was lower-risk.
- And: the Llama cluster existed; adding BGE was incremental ops cost.

Decision driven primarily by compliance / control, not cost.

### 8.5 The Canadian deployment (2026)

Per [data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md), Atlantic Maple required `ca-central-1`:

- Cohere Command R+ via Cohere direct API (hosted direct) for primary chat.
- No self-hosted in Canada (small volume; didn't justify ops).
- Cohere embed-multilingual for Canadian embeddings.

Decision: hosted direct for Cohere in Canada. Small volume; no need for managed inference or self-host complexity.

### 8.6 The current portfolio (mid-2026)

```
Workload                          Option              Provider/Model
─────────────────────────────────────────────────────────────────────
Care Coordinator (US)              Managed inference   AWS Bedrock + Anthropic Sonnet
Care Coordinator (Canada)          Hosted direct       Cohere Command R+
Patient API chat (US)              Managed inference   AWS Bedrock + Anthropic Sonnet
Patient API chat (Canada)          Hosted direct       Cohere Command R+
Document ingestion / classify      Self-hosted         Llama 3 70B on HIPAA cluster
Document embedding                 Self-hosted         BGE-large on HIPAA cluster
Billing-code workflow              Self-hosted (LoRA)  Llama 3 8B fine-tuned on HIPAA cluster
Analytics warehouse copilot        Managed inference   AWS Bedrock + Anthropic Sonnet
Internal IT copilot                Hosted direct       Anthropic Haiku
```

Five distinct options in production. Each chosen for the workload.

### 8.7 The cost picture

Total monthly AI spend: ~$200k.

- Managed inference (Bedrock): ~$120k.
- Self-hosted GPU infrastructure: ~$45k (visible) + ~$25k (ops FTE allocation) = ~$70k.
- Hosted direct (Cohere, Anthropic Haiku): ~$10k.

If everything were hosted direct: estimated $400-500k/month. Self-hosted portfolio saves ~$200-300k/month.

If everything were self-hosted: would require ~3 FTE in GPU ops + comparable infrastructure cost ($120k+/month) + significant quality regression on clinical workloads. Not justified.

The hybrid is the cost-optimal answer.

### 8.8 The lessons

- The decision was made per workload, not platform-wide.
- The hybrid emerged over 18 months as workloads matured.
- The catalogue (cross-link to [model-catalogue-and-registry.md](./model-catalogue-and-registry.md)) tracks all options consistently.
- Annual review evaluates whether the portfolio is still right.
- The Q3 2025 self-host buildup was the most consequential decision; took 8 weeks but pays continuously.

---

## 9. Anti-patterns

### 9.1 The "we'll self-host to save money" intuition without analysis

**Pattern.** CFO sees hosted bill; "let's self-host." Engineering builds infrastructure; 12 months in, total cost is higher than hosted because of operational overhead.

**Corrective.** Total cost analysis (§4) before committing. Visible vs hidden; FTE costs; eval suite costs; ops capacity.

### 9.2 The "multi-vendor for resilience" hedge that costs more than it saves

**Pattern.** Two providers in production "for resilience." Twice the operational surface; twice the contracts; twice the cost-tracking; the alleged resilience benefit rarely materializes because real outages are correlated or short.

**Corrective.** Single-vendor for most workloads; multi-vendor only when each vendor is genuinely better for some workload, or when business continuity contractually requires multi-vendor.

### 9.3 The "we'll just use Anthropic for everything" default

**Pattern.** First workload picks Anthropic; pattern persists across all workloads. Embedding workloads use Anthropic embedding (when OpenAI / Cohere / open-weight would be better). Bulk classification uses Anthropic Sonnet (when Haiku or open-weight would be cheaper).

**Corrective.** Per-workload selection. Anthropic for what it's best at; other options for what they're best at.

### 9.4 The self-host buildup without eval discipline

**Pattern.** Stand up GPU infrastructure; deploy Llama; ship to production; discover the eval gap weeks later. Too late to easily migrate back.

**Corrective.** Eval suite built before infrastructure. Verify the open-weight model meets quality bar in pre-production; commit to the option only after eval passes.

### 9.5 The cloud lock-in that wasn't a choice

**Pattern.** Choose managed inference for "cloud-native integration." Become entirely dependent on the cloud's ecosystem (IAM, VPC endpoints, KMS). Migration to another cloud requires substantial re-architecture.

**Corrective.** Cloud lock-in is sometimes the right choice; make it explicitly. Document what's cloud-specific; understand the migration cost.

### 9.6 The white-label provider that disappeared

**Pattern.** Use a small white-label provider for cost. Provider gets acquired; service shuts down; emergency migration with weeks of notice.

**Corrective.** Critical workloads have documented escape plans. Smaller white-label providers used for non-critical workloads; the largest hyperscaler-backed ones (Bedrock, Vertex, Azure OpenAI) for critical.

### 9.7 The "we'll add managed inference later" deferral

**Pattern.** Start with hosted direct. Regulated customer arrives; the BAA + IAM + VPC requirements force managed-inference migration. Migration is during the customer onboarding pressure.

**Corrective.** If managed inference is on the roadmap, set it up before the regulated customer arrives. Migration in-flight is harder than parallel deployment.

### 9.8 The "self-hosted GPUs sit idle 70% of the time" inefficiency

**Pattern.** Build GPU cluster for peak load. Peak is 30% of capacity; 70% sits idle. The economics that justified self-hosting assumed high utilization; reality is low utilization; net cost is higher than hosted.

**Corrective.** Right-size for sustained load; burst to hosted for peaks; or use white-label for the bursty portion. Self-host is most economic at high sustained utilization.

---

## 10. Findings (sprint-assignable)

**ARCH-BVB-001 (P0). No build-vs-buy decision framework documented.** Each workload picks ad-hoc; portfolio drifts. Adopt criteria framework (§3); apply per workload; document choices in catalogue. Owner: AI platform + architecture.

**ARCH-BVB-002 (P0). Self-host decision made without total cost analysis.** Per-token comparison ignores hidden costs (ops FTE, eval, lifecycle). Require total cost analysis (§4) for any self-host proposal. Owner: AI platform + FinOps.

**ARCH-BVB-003 (P0). All workloads on single provider by default.** Per-workload optimization untapped; cost or quality left on table. Per-workload selection enabled by routing layer + catalogue; review annually. Owner: AI platform.

**ARCH-BVB-004 (P1). No eval suite for candidate models on a workload.** Self-host or white-label adoption proceeds without quality validation. Eval suite built before infrastructure; quality bar verified pre-deploy. Owner: AI platform + feature teams.

**ARCH-BVB-005 (P1). Managed inference not evaluated for regulated workloads.** Regulated workloads default to hosted-direct; compliance posture harder than necessary. Default regulated workloads to managed inference (cloud provider's compliance posture). Owner: AI platform + compliance.

**ARCH-BVB-006 (P1). White-label provider used without compliance verification.** Vendor's HIPAA / SOC 2 / GDPR posture not documented; assumed not verified. Per-vendor compliance verification before production use; documented in catalogue. Owner: security + compliance.

**ARCH-BVB-007 (P1). No documented escape plan for critical workloads.** Provider exit or capability deprecation is improvised. Per-critical-workload escape plan: alternative model, eval verification, migration timeline. Owner: AI platform + product.

**ARCH-BVB-008 (P1). Multi-vendor for "resilience" without documented benefit.** Operational cost incurred for unclear value. Either document the resilience justification (specific scenarios; cost-benefit) or consolidate to single vendor. Owner: AI platform + architecture.

**ARCH-BVB-009 (P1). GPU infrastructure utilization not tracked.** Self-host economics assumed high utilization; actual utilization not measured. Track GPU utilization; if < 50% sustained, reconsider option. Owner: AI platform + observability.

**ARCH-BVB-010 (P2). Cloud lock-in from managed inference not documented.** Migration cost off-roadmap. Document cloud-specific dependencies; understand cross-cloud migration cost. Owner: architecture.

**ARCH-BVB-011 (P2). No annual portfolio review.** Choices made at workload launch never revisited. Annual review per workload: criteria still apply? cost changed? new option available? Owner: AI platform.

**ARCH-BVB-012 (P2). No growth-trajectory consideration.** Workload at 5M tokens/day chosen for current scale; will be 50M in a year. Decision should consider 12-month trajectory; revisit when crossing thresholds. Owner: AI platform + product.

**ARCH-BVB-013 (P2). Self-host buildup has no operational on-call.** First production incident with self-hosted GPU is improvised. On-call rotation with GPU experience; runbook; pre-production failure tests. Owner: AI platform + SRE.

**ARCH-BVB-014 (P2). White-label model version updates not pinned.** Provider updates model; behavior changes; production issues. Pin to specific model version where supported; subscribe to provider release notes. Owner: AI platform.

**ARCH-BVB-015 (P2). Managed inference vs direct-hosted decision is ad-hoc.** Some workloads on Bedrock, some on Anthropic direct; no documented rationale. Apply §7 decision matrix; document per workload. Owner: AI platform.

**ARCH-BVB-016 (P3). Burst-to-hosted not implemented for self-hosted workloads.** Self-host can't handle peaks; degraded under load. Burst-to-hosted (or burst-to-white-label) for peak overflow; routing layer handles. Owner: AI platform.

**ARCH-BVB-017 (P3). FTE cost of self-hosting not budgeted.** Team takes on GPU ops without capacity; gets stretched. Document FTE per self-host workload; budget against headcount. Owner: engineering management.

**ARCH-BVB-018 (P3). Provider deprecation tracking not linked to build-vs-buy.** Provider deprecates model; team can't easily switch options. Tracking integrated; deprecation triggers build-vs-buy review for affected workloads. Owner: AI platform.

---

## 11. Adoption sequencing checklist

For a team designing or reviewing build-vs-buy choices, in order:

- [ ] **Inventory current workloads.** What does each workload do? Volume? Quality requirement? Compliance? Latency? Growth trajectory?
- [ ] **For each workload, apply criteria framework (§3).** Document the binding criterion and the natural option.
- [ ] **For workloads where multiple options work, do total cost analysis (§4).** Visible + hidden cost per option.
- [ ] **For self-host candidates, build eval suite first.** Verify quality bar pre-infrastructure.
- [ ] **For regulated workloads, default to managed inference (§7).** Override only with documented rationale.
- [ ] **For white-label candidates, verify compliance posture.** Document in catalogue.
- [ ] **For multi-vendor candidates, document the resilience case.** Or consolidate.
- [ ] **Build routing layer that supports per-workload selection.** Cross-link to model-routing-and-tiering.md.
- [ ] **Populate catalogue with each option as a distinct entry.** Cross-link to model-catalogue-and-registry.md.
- [ ] **For critical workloads, document escape plan.** Alternative model + eval + migration timeline.
- [ ] **Track GPU utilization for self-host options.** Alert if < 50% sustained.
- [ ] **Track per-workload cost.** Surface to FinOps; quarterly review.
- [ ] **Annual portfolio review.** Workload-by-workload re-evaluation.
- [ ] **Document FTE allocation for self-hosted options.** Visible in capacity planning.
- [ ] **Pre-production chaos test self-hosted setup.** Worker failure, GPU OOM, model upgrade.

---

## 12. References

**In this folder.**
- [frontier-vs-open-weights-vs-fine-tune.md](./frontier-vs-open-weights-vs-fine-tune.md) — foundational decision; build-vs-buy decision sits on top.
- [model-routing-and-tiering.md](./model-routing-and-tiering.md) — routing layer that enables per-workload selection.
- [model-catalogue-and-registry.md](./model-catalogue-and-registry.md) — catalogue tracks every option per workload.
- [model-migration-playbook.md](./model-migration-playbook.md) *(coming)* — migration mechanics for the escape paths.
- [capability-vs-cost-vs-latency-tradeoffs.md](./capability-vs-cost-vs-latency-tradeoffs.md) *(coming)* — non-functional comparison.

**Elsewhere in this repo.**
- [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — the extreme case of build (per-tenant fine-tune).
- [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md) — residency requirements that constrain build-vs-buy.
- [integration-architecture/backpressure-and-queueing.md](../integration-architecture/backpressure-and-queueing.md) — rate-limit handling; hosted vs self-hosted have different patterns.
- [reference-systems/meridian-care-coordinator.md](../reference-systems/meridian-care-coordinator.md) — worked example whose build-vs-buy is described here.
- [reference-systems/analytics-warehouse-copilot.md](../reference-systems/analytics-warehouse-copilot.md) — different workload, different decision.

**Sibling repos.**
- [ai-engineering-reference-architecture / model-lifecycle / model-registry.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/model-lifecycle/model-registry.md) — engineering-side registry across all options.
- [ai-engineering-reference-architecture / cost-and-finops / cost-attribution.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/cost-attribution.md) — per-workload, per-option cost attribution.
- [ai-engineering-reference-architecture / cost-and-finops / tier-routing-for-cost.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/cost-and-finops/tier-routing-for-cost.md) — cost-driven routing across options.
- [ai-engineering-reference-architecture / eval-engineering / eval-engineering-playbook.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/eval-engineering/eval-engineering-playbook.md) — eval suites that validate option quality.

**External.**
- AWS Bedrock pricing and model documentation.
- GCP Vertex AI pricing and model documentation.
- Azure OpenAI Service pricing and model documentation.
- Anthropic / OpenAI / Cohere direct pricing.
- Together AI, Fireworks AI, Replicate, Anyscale, Lepton pricing.
- GPU cloud pricing (AWS, GCP, Azure, Lambda Labs, CoreWeave) for self-host TCO.
- vLLM, TGI, Triton documentation for self-host inference.
