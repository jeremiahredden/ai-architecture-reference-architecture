# GPU Strategy for Self-Hosted

> **Audience.** Architects considering whether to self-host AI inference. Tech leads whose CFO asked "can we just buy our own GPUs?" Anyone whose hosted-model bill is large enough that self-hosting starts to feel plausible. **Scope.** The *architectural* decisions: when self-hosting open-weight inference makes sense and when it doesn't; GPU choices (H100, MI300X, L40S, consumer cards); serving-framework choices (vLLM, TGI, SGLang, Triton); capacity planning; the cost-crossover point; the operational cost that often dominates the bill. Not the build-vs-buy decision overview (see [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md)). Not the engineering operations of GPU clusters (see sibling [reliability-engineering/capacity-planning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/capacity-planning.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Self-hosting open-weight inference is increasingly viable in 2026. Open-weight models (Llama, Mistral, Qwen) match hosted-frontier on many workloads; GPU pricing has improved; serving frameworks (vLLM, TGI) deliver excellent throughput.

The temptation: "we're spending $X on hosted; let's self-host and save."

The reality: self-hosting has costs hosted doesn't show:

- GPU procurement (capital or reserved-instance).
- Infrastructure operation (compute, networking, storage).
- Model lifecycle (training, eval, deployment, rollback).
- On-call (GPU failures; OOM; capacity).
- Drift monitoring.
- Compliance overhead.

Many self-hosting projects discover total cost is higher than hosted. The cases where self-host wins are real but require specific conditions.

This document covers the architectural decision and the operational reality.

This document is opinionated about four things:

1. **Self-hosting is rarely cheaper at small / medium scale.** Operational overhead dominates. Crossover at high volume.
2. **GPU choice matters more than people think.** H100 vs L40S vs MI300X have different cost / capability profiles.
3. **The serving framework is load-bearing.** vLLM continuous batching delivers 2-5x what naive serving does.
4. **Self-hosting commitment is multi-year.** Don't enter without operational capacity.

Structure: (2) when self-host wins; (3) when hosted wins; (4) GPU choices in 2026; (5) serving framework choices; (6) capacity planning; (7) the operational cost reality; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. When self-host wins

The cases where self-host is justified.

### 2.1 Very high volume

For workloads at 100M+ tokens/day:

- Hosted: per-token cost dominates ($X/day).
- Self-host: infrastructure amortized; per-token near-zero.

At scale, self-host wins.

Cross-link to [model-strategy/build-vs-buy-decision.md §4](../model-strategy/build-vs-buy-decision.md).

### 2.2 Strict latency requirements

Some workloads need sub-second TTFT:

- Voice assistants.
- Real-time editing.

Self-hosted small models (Llama 3 8B) can achieve P99 < 200ms.

Hosted models typically P99 > 1s.

### 2.3 Residency / regulatory constraints

For data that can't go to hosted providers:

- HIPAA-strict workloads (some interpretations).
- Sovereign-cloud requirements.
- Air-gapped environments.

Self-host is sometimes the only option.

Cross-link to [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md).

### 2.4 Fine-tune at scale

For per-tenant fine-tunes (cross-link to [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md)):

- Hosted: per-tenant fine-tune costs.
- Self-hosted: LoRA adapters on shared base; cheaper at scale.

For multi-tenant fine-tune offerings: self-host can be necessary.

### 2.5 Capability not available hosted

Some open-weight models meet specific needs that hosted doesn't:

- Specialized fine-tunes (Code Llama for code; Med-Llama for clinical).
- Newer open-weight (community releases).

For these: self-host is the only path.

### 2.6 Capacity / rate-limit constraints

If hosted provider can't grow capacity fast enough:

- Provider account limits.
- Provider regional constraints.

Self-host adds capacity independently.

### 2.7 The "we want to learn / experiment" reason

Self-hosting for learning:

- Team gets GPU + serving experience.
- Internal capability.
- Innovation possible.

But: significant investment for educational purposes alone.

---

## 3. When hosted wins

The default case.

### 3.1 Low / medium volume

For workloads at <50M tokens/day:

- Hosted: ~$5k-25k/month.
- Self-host: $20k-50k/month operational + capital.

Hosted wins.

### 3.2 Need for frontier-class capability

Frontier models (Claude Sonnet, GPT-4o) > open-weight on many benchmarks.

For workloads where frontier capability is required: hosted is the only viable.

Cross-link to [model-strategy/capability-vs-cost-vs-latency-tradeoffs.md](../model-strategy/capability-vs-cost-vs-latency-tradeoffs.md).

### 3.3 No GPU ops capacity

Teams without GPU operations experience:

- Hiring GPU experts is hard.
- Building internal expertise takes time.

Hosted: provider handles operations.

### 3.4 Rapidly evolving requirements

Workload requirements change:

- Hosted: provider improves model continuously.
- Self-host: re-train, re-deploy per change.

For changing workloads: hosted is more responsive.

### 3.5 Multi-cloud / multi-region complexity

For multi-region deployments:

- Hosted: provider's regional coverage.
- Self-host: each region needs own GPU pool.

Self-host doesn't scale across regions well.

### 3.6 Compliance posture matched by hosted

For most regulated workloads:

- Hosted (with BAA, DPA, etc.) is acceptable.
- Self-host's compliance burden is yours alone.

Verify; don't assume self-host has compliance advantages.

### 3.7 The "we don't have a use case yet" exploration phase

Teams exploring AI:

- Use hosted; iterate quickly.
- Self-host only after needs solidify.

---

## 4. GPU choices in 2026

The hardware landscape.

### 4.1 The data-center GPUs

**NVIDIA H100 (80GB):**

- Performance: top-tier for AI inference.
- Cost: ~$25k-30k purchase; ~$3-5/hour cloud.
- For: large models (70B+); high-throughput.

**NVIDIA H200 (141GB):**

- More memory than H100; better for very-long-context.
- Cost: ~$30k+ purchase; ~$4-6/hour cloud.

**AMD MI300X (192GB):**

- Compelling alternative to H100; lots of memory.
- Cost: similar to H100; varies by supplier.
- Software: ROCm; less mature than CUDA.

**NVIDIA L40S (48GB):**

- Mid-tier; good for smaller models (up to 30B).
- Cost: ~$8-10k; ~$1-2/hour.

### 4.2 The consumer / lower-end GPUs

**RTX 4090 (24GB):**

- Consumer card; not data-center-class but usable.
- Cost: ~$2k purchase.
- For: 7-8B models in inference.

**Other consumer cards:**

- Generally not for production AI inference.
- Power, cooling, reliability concerns.

### 4.3 The specialized inference accelerators

**Groq, Cerebras, Sambanova:**

- Custom silicon optimized for inference.
- Sub-100ms TTFT possible.
- Limited model selection (whatever they've ported).
- For: ultra-low-latency specific needs.

### 4.4 The choice per workload

```
Workload                  Recommended GPU
─────────────────────────────────────────────────
Llama 3 70B inference     1x H100 80GB (typical); H200 if context long
Llama 3 8B inference      L40S; some H100 if mixing with larger
Embedding (BGE-large)     L40S or T4 (lower)
Voice / sub-100ms         Groq / Cerebras
```

Match GPU to model size and workload.

### 4.5 The "we have an A100" reality

Many teams have A100s (previous generation):

- Still capable for many models.
- Cost-effective if already purchased.
- For new purchases: H100/H200 is better value.

### 4.6 The buying-vs-leasing decision

**Buy:**

- 3-year amortization typical.
- Capital allocation.
- Lock-in to hardware.

**Lease (cloud):**

- Pay-as-you-go.
- Flexibility.
- Per-hour rates add up.

For 1-2 year commitments: lease.
For 3+ years sustained use: buy or commit.

### 4.7 The "we got an early Blackwell" pattern

Early access to newer hardware:

- NVIDIA Blackwell (2025-2026 generation).
- Better performance per dollar.
- Limited availability.

Early adopters benefit; most teams wait for general availability.

---

## 5. Serving framework choices

The software that runs on the GPU.

### 5.1 vLLM

**What it is.** Open-source LLM inference server.

**Key feature.** Continuous batching: multiple requests batched on GPU simultaneously; new requests join in-flight.

**Performance.** 2-5x throughput vs naive batching.

**Adoption.** Most popular self-hosted option in 2026.

**Limitations.** Model support is broad but not universal; tuning for specific models.

### 5.2 TGI (Text Generation Inference)

**What it is.** HuggingFace's inference server.

**Performance.** Similar to vLLM.

**Adoption.** Common in HuggingFace ecosystem; integration with HF Models.

**Limitations.** Less feature-rich than vLLM for some use cases.

### 5.3 SGLang

**What it is.** Inference framework with structured-output focus.

**Key feature.** Constrained decoding; tool-use formats.

**Adoption.** Growing; niche for structured-output workloads.

### 5.4 Triton (NVIDIA)

**What it is.** NVIDIA's general inference server.

**Adoption.** Mature; for non-LLM models often.

**For LLMs.** Used with custom backends.

### 5.5 The choice per workload

```
Workload                           Framework
──────────────────────────────────────────────────
General LLM inference              vLLM (default)
HuggingFace ecosystem-bound        TGI
Structured-output focus            SGLang
Mixed-model inference              Triton
```

Most teams use vLLM unless a specific reason for another.

### 5.6 The "we wrote our own serving" trap

Teams sometimes build custom serving:

- Significant engineering investment.
- vLLM has eaten most use cases.
- Custom rarely matches vLLM's continuous batching.

**Corrective.** Use vLLM unless specific gap.

### 5.7 The framework operational considerations

Each framework has:

- Configuration complexity.
- Model-loading mechanics.
- Auto-scaling behaviors.
- Logging / observability.

Pre-test in staging.

### 5.8 The framework upgrade path

Frameworks evolve:

- vLLM major versions every quarter.
- Breaking changes possible.

Track; upgrade with discipline.

---

## 6. Capacity planning

How much hardware.

### 6.1 The throughput model

For a given GPU + serving framework + model:

```
throughput = tokens_per_minute (model-specific)
```

E.g., Llama 3 70B on H100 80GB with vLLM: ~25,000 tokens/min per request, with continuous batching boosting to ~150,000 tokens/min aggregate.

Per GPU, per model: measure.

### 6.2 The peak vs sustained capacity

```
Peak hourly demand: 5M tokens/hour
Steady-state demand: 2M tokens/hour
Provisioning for: 1.5x peak = 7.5M tokens/hour → 50 GPU-hours
```

Provision for peak with margin.

### 6.3 The auto-scaling

GPU auto-scaling:

- Cold start: 5-15 minutes.
- Pre-warm: 4-12 GPUs always warm.
- Burst: additional 2-3 GPUs per peak event.

Cross-link to [ai-engineering-reference-architecture / reliability-engineering / capacity-planning.md §7](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/capacity-planning.md).

### 6.4 The capacity-vs-cost trade-off

Over-provisioning: idle GPUs cost.

Under-provisioning: queueing; degraded latency.

Aim for ~60-70% sustained utilization.

### 6.5 The model-loading optimization

Model weights are large (Llama 3 70B: ~140GB):

- First load: 10-30 min.
- Subsequent: cache hit.

Architecture:

- Pre-cache weights on GPU nodes.
- Auto-scaler pre-loads weights on new nodes.

### 6.6 The multi-model capacity

For workloads with multiple models:

- Dedicated GPU per model.
- Or: time-share if memory allows.

Per-workload throughput requirements determine.

### 6.7 The cross-region capacity

For multi-region:

- Each region needs own GPU pool.
- No cross-region GPU sharing typically.

Cross-link to [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md).

### 6.8 The "we need 20% headroom" target

Like provider rate-limit headroom:

- 20% headroom comfortable.
- 10% concerning.
- 5% at-risk.

Capacity-headroom alerts.

### 6.9 The dynamic resize

For changing workload:

- Quarterly capacity review.
- Add / remove GPUs based on trend.
- Avoid significantly-over or significantly-under provisioning.

---

## 7. The operational cost reality

What teams underestimate.

### 7.1 The personnel cost

GPU operations need experience:

- GPU ops engineer: $200k-300k loaded.
- 0.5-1 FTE per serious GPU pool.

For self-host: budget personnel.

### 7.2 The on-call

GPU failures:

- Hardware (rare but happens).
- Driver / framework issues.
- Capacity emergencies.

24/7 on-call rotation.

### 7.3 The model lifecycle ops

Per model:

- Training (if applicable).
- Eval per version.
- Deployment.
- Rollback.
- Deprecation.

Multi-week per cycle.

### 7.4 The framework operations

vLLM (or chosen framework):

- Version upgrades.
- Configuration tuning.
- Troubleshooting.

Ongoing.

### 7.5 The capacity ops

- Auto-scaling configuration.
- Headroom monitoring.
- Quarterly resize.

Daily attention.

### 7.6 The total operational cost

For a "moderate self-host" (4-8 GPUs):

- Personnel: $150-300k/year (0.5-1 FTE).
- Operational infrastructure: $50-100k/year.
- GPU costs: $50-200k/year.
- Engineering setup: ~$100k initial.

Total: $250-700k/year ongoing.

### 7.7 The hosted-equivalent cost

For comparable workload hosted:

- Per-call rate × volume.
- For mid-volume: $100-300k/year.

The "self-host saves money" intuition is wrong for many workloads; total cost similar or higher.

### 7.8 The "we underestimated" pattern

Common failure:

- Initial estimate: $50k/year.
- Actual after year 1: $300k/year (personnel + ops).
- "Why didn't we know?"

Realistic estimation is hard.

### 7.9 The "we want to self-host for control" justification

Beyond cost:

- Data isolation.
- Custom fine-tuning.
- Latency requirements.

For these: self-host's value beyond direct cost.

---

## 8. Worked Meridian example

Meridian's self-host decisions.

### 8.1 The decision per workload

```
Workload                   Hosted vs Self-host    Rationale
──────────────────────────────────────────────────────────────────
Care Coordinator (US)      Managed inference     Capability, compliance, modest volume
Patient API chat (US)      Managed inference     Capability, compliance
Patient API chat (CA)      Hosted direct (Cohere) Residency
Document ingestion         Self-hosted Llama 70B Volume; cost crossover
Document embedding         Self-hosted BGE        Cost + data isolation
Clinical decision support  Managed inference     Safety-critical; capability
Analytics warehouse copilot Managed inference    Diverse queries
Internal IT copilot        Hosted direct (Haiku)  Low volume
Billing-code (specialty)   Self-hosted Llama 8B  Specialized fine-tune
```

Mixed; ~25% of workloads self-hosted.

### 8.2 The document-ingestion self-host

Highest-volume workload; cost-driven decision.

Setup:
- 8 H100 80GB GPUs on AWS HIPAA infrastructure.
- vLLM serving framework.
- Llama 3 70B base.
- Auto-scaling: 4-12 GPUs.

Cost:
- Infrastructure: $40k/month.
- Operations (allocated): ~$20k/month (0.4 FTE).
- Total: ~$60k/month vs hosted alternative $360k/month.
- Net: ~$300k/month saved.

ROI: massive on this workload.

### 8.3 The embedding self-host

Lower-effort self-host (shared cluster):

- Same 8 GPU cluster.
- BGE-large embedding model alongside Llama.
- Same vLLM framework.

Cost: minimal incremental ($1k/month) for the embedding workload.

### 8.4 The billing-code fine-tune self-host

Per-tenant fine-tune (LoRA on Llama 3 8B):

- LoRA file ~100MB per tenant.
- Inference: shared base model; switch LoRA per request.

Setup additional: ~$15k/month for fine-tune training capability + LoRA storage.

ROI: $20k/year saved on the specialty practice workload; justified.

### 8.5 The Q3 2025 GPU shortage

AWS GPU instances had reduced availability for a period:

- Auto-scaling couldn't add GPUs.
- Queue grew.
- Mitigated by: longer pre-warm pool; spillover to hosted Sonnet for excess.

Lessons:
- Cloud GPU availability isn't 100%.
- Plan for spillover.

### 8.6 The "we tried H200" experiment

Q1 2026: tested H200 (more memory) for long-context workloads:

- TTFT for 100k-token inputs: 4s on H100 vs 2.5s on H200.
- Cost: 30% premium per hour.
- Use case: rare (some Patient record reviews).

Decision: not justified for current workload mix. Re-evaluate annually.

### 8.7 The cross-region challenge

For Canadian customers (Atlantic Maple):

- Considered: self-hosted Llama in `ca-central-1`.
- AWS GPU availability in ca-central-1 limited.
- Cost: ~50% premium.
- Decision: used Cohere Command R+ instead.

Self-hosting in CA wasn't operationally feasible.

### 8.8 The infrastructure overview

```
Production:
  US-East-1:
    8x H100 80GB GPUs.
    vLLM serving Llama 3 70B + BGE-large + LoRA adapters.
    Auto-scaling 4-12.
    Network / storage: standard AWS.

CA-Central-1:
  Not self-hosted; Cohere managed.
  
Development:
  2x L40S GPUs (dev environment).
```

Per-region.

### 8.9 The operational cost

- 8-GPU production cluster: $40k/month (3-year reserved).
- 2-GPU dev: $4k/month.
- Engineering: 0.5 FTE allocated to GPU ops.

Total: ~$70k/month for the self-hosted portion. Saves ~$300k/month vs hosted alternative on document ingestion alone.

### 8.10 The lessons

- Self-host only the workloads justifying it.
- vLLM is essential for throughput.
- Auto-scaling needs pre-warm pool.
- Multi-region self-host is hard; consider hosted alternatives.
- Total cost includes operations; budget personnel.

---

## 9. Anti-patterns

### 9.1 The "we'll save money by self-hosting" assumption

**Pattern.** Calculate hosted cost; compare to GPU rental; conclude self-host saves.

**Corrective.** Include operations / personnel / lifecycle. Per §7.6.

### 9.2 The GPU-buying without serving-framework choice

**Pattern.** Buy GPUs; then figure out serving software.

**Corrective.** Serving framework first per §5; informs GPU choice.

### 9.3 The "we'll figure out ops as we go" deferral

**Pattern.** Self-host launched; ops capability built reactively.

**Corrective.** Operations from day one per §7.

### 9.4 The "consumer GPU is cheaper" trap

**Pattern.** RTX 4090 used in production; reliability issues; power / cooling problems.

**Corrective.** Data-center GPUs for production per §4.1.

### 9.5 The no-pre-warm cold-start

**Pattern.** Reactive auto-scaling; 10-min cold start; users see degraded latency.

**Corrective.** Pre-warm pool per §6.3.

### 9.6 The "we wrote our own serving" reinvention

**Pattern.** Custom serving instead of vLLM. Significant engineering; rarely matches vLLM throughput.

**Corrective.** Use vLLM per §5.6.

### 9.7 The single-region self-host for global workload

**Pattern.** Self-hosted in one region; cross-region latency.

**Corrective.** Multi-region or hybrid per §6.7.

### 9.8 The under-provisioning

**Pattern.** Sized for steady-state; peak demand exceeds; queue grows.

**Corrective.** Peak + 1.5x margin per §6.2.

### 9.9 The over-provisioning

**Pattern.** Bought capacity for 2-3x expected; sustained 20% utilization.

**Corrective.** Right-sized per §6.4.

### 9.10 The "GPUs are commodity" assumption

**Pattern.** Capacity assumed always available. GPU shortage during a demand spike; mitigation absent.

**Corrective.** Spillover to hosted per §8.5.

---

## 10. Findings (sprint-assignable)

**ARCH-GPU-001 (P0). Self-host decision made without total cost analysis.** Operations cost ignored. Per §7.6. Owner: AI platform + engineering management.

**ARCH-GPU-002 (P0). Self-host launched without operations capacity.** Will fail or strain team. Per §7. Owner: engineering management.

**ARCH-GPU-003 (P0). vLLM (or equivalent) not used; naive serving.** Sub-optimal throughput. Per §5. Owner: AI platform.

**ARCH-GPU-004 (P1). No pre-warm pool for GPUs.** Cold-start latency. Per §6.3 and [capacity-planning.md §7](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/capacity-planning.md). Owner: AI platform.

**ARCH-GPU-005 (P1). Capacity headroom monitoring absent.** Surprises during spikes. Per §6.8. Owner: AI platform + SRE.

**ARCH-GPU-006 (P1). Personnel cost not budgeted.** Self-host underfunded operationally. Per §7.1. Owner: engineering management.

**ARCH-GPU-007 (P1). GPU choice not workload-aligned.** Wrong tier for the workload. Per §4.4. Owner: AI platform.

**ARCH-GPU-008 (P1). Multi-region self-host attempted without infrastructure.** Won't scale. Per §6.7. Owner: AI platform.

**ARCH-GPU-009 (P2). Spillover to hosted not planned.** GPU shortage = workload down. Per §8.5. Owner: AI platform.

**ARCH-GPU-010 (P2). Auto-scaling without sufficient cold-start mitigation.** Reactive scaling too slow. Per §6.3. Owner: AI platform.

**ARCH-GPU-011 (P2). Model lifecycle absent.** Self-hosted model never updated. Per §7.3. Owner: AI platform.

**ARCH-GPU-012 (P2). Framework upgrade path undefined.** vLLM upgrade is disruption. Per §5.8. Owner: AI platform.

**ARCH-GPU-013 (P2). Cross-region GPU sharing attempted.** Doesn't work; latency. Per §6.7. Owner: AI platform.

**ARCH-GPU-014 (P2). Model-loading optimization absent.** First call after cold-start has elevated latency. Per §6.5. Owner: AI platform.

**ARCH-GPU-015 (P3). H200 / newer GPU evaluation not done.** May be missing performance / cost wins. Per §8.6. Owner: AI platform.

**ARCH-GPU-016 (P3). Annual GPU strategy review absent.** Drift; over- or under-provisioning. Owner: AI platform.

**ARCH-GPU-017 (P3). Custom GPU procurement when leasing was right.** Capital tied up. Per §4.6. Owner: engineering management + finance.

**ARCH-GPU-018 (P3). Buy decision based on 3-year amortization for changing workload.** Over-commit. Per §4.6. Owner: engineering management.

---

## 11. Adoption sequencing checklist

- [ ] **Total cost analysis (including operations) per §7.6.**
- [ ] **Operations capacity hiring before launch.**
- [ ] **vLLM (or equivalent) deployment (§5).**
- [ ] **GPU choice aligned with workload (§4.4).**
- [ ] **Pre-warm pool (§6.3).**
- [ ] **Capacity headroom monitoring (§6.8).**
- [ ] **Auto-scaling with appropriate triggers.**
- [ ] **Spillover-to-hosted plan (§8.5).**
- [ ] **Model lifecycle plan (§7.3).**
- [ ] **Multi-region: separate self-host or hybrid (§6.7).**
- [ ] **Annual GPU strategy review.**

---

## 12. References

**In this folder.**
- [token-economics.md](./token-economics.md) — self-host economics.
- [throughput-and-concurrency.md](./throughput-and-concurrency.md) — concurrency on GPU.
- [latency-budgets-and-streaming.md](./latency-budgets-and-streaming.md) — latency implications.
- [caching-tiers.md](./caching-tiers.md) — caching reduces GPU load.

**Elsewhere in this repo.**
- [model-strategy/build-vs-buy-decision.md](../model-strategy/build-vs-buy-decision.md) — broader decision.
- [model-strategy/frontier-vs-open-weights-vs-fine-tune.md](../model-strategy/frontier-vs-open-weights-vs-fine-tune.md) — model selection.
- [multi-tenancy-and-isolation/per-tenant-fine-tuning.md](../multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — LoRA adapters.
- [multi-tenancy-and-isolation/data-residency-patterns.md](../multi-tenancy-and-isolation/data-residency-patterns.md) — per-region GPU.

**Sibling repos.**
- [ai-engineering-reference-architecture / reliability-engineering / capacity-planning.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/capacity-planning.md) — engineering capacity.

**External.**
- vLLM documentation.
- TGI documentation.
- NVIDIA GPU documentation (H100, H200, etc.).
- AMD MI300X documentation.
- AWS / GCP / Azure GPU instance documentation.
