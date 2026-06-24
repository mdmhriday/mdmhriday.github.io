---
layout: page
title: Do Vision-Language Models Actually Use the Image?
description: A controlled perturbation study across 1,500 visually-necessary questions from MMStar
importance: 1
category: research
related_publications: false
---

Vision-language models score well on standard benchmarks. But scoring well does not tell you *how* the model arrived at its answer — whether it genuinely processed the image or whether it could have done just as well without it. This project runs a controlled experiment to find out where, and how much, a VLM actually depends on visual input.

---

## Setup

I evaluated **Qwen2.5-VL-7B-Instruct** on all 1,500 questions from [MMStar](https://github.com/MMStar-Benchmark/MMStar) — the only VLM benchmark constructed with systematic visual-necessity validation, meaning every question was verified to be unsolvable from text alone before inclusion. That property matters here: if questions were text-solvable, null conditions like removing the image entirely would give uninformative results.

Each question was answered under nine conditions spanning three categories:

<div class="table-responsive">
<table class="table table-sm table-bordered">
<thead class="thead-light">
<tr><th>Category</th><th>Condition</th><th>What changes</th></tr>
</thead>
<tbody>
<tr><td rowspan="3"><strong>Pipeline localisation</strong></td><td>C1 — Token omission</td><td>All image tokens removed</td></tr>
<tr><td>C2 — Encoder noise</td><td>Gaussian noise at pixel level before the encoder</td></tr>
<tr><td>C3 — Spatial shuffle</td><td>Projected embeddings permuted across spatial positions after the MLP bridge</td></tr>
<tr><td rowspan="5"><strong>Degradation geometry</strong></td><td>C4 (σ=2,5,10,20)</td><td>Gaussian blur at four severity levels</td></tr>
<tr><td>C5a–d (10–75%)</td><td>Progressive patch masking at four levels</td></tr>
<tr><td>C6 — Token reduction</td><td>Fewer image tokens, spatial coherence preserved</td></tr>
<tr><td colspan="2"> </td></tr>
<tr><td colspan="2"> </td></tr>
<tr><td><strong>Cross-modal comparison</strong></td><td>C7 — Text shuffle</td><td>Word tokens randomly shuffled; destroys syntax and semantics</td></tr>
</tbody>
</table>
</div>

The primary metric is **Total Variation Distance (TVD)** between the output distribution under each condition and the full-image baseline C0. TVD is bounded [0,1] and captures distributional shifts that accuracy alone misses — two conditions can produce identical accuracy while shifting the model's probability distribution very differently.

---

## Finding 1 — The Cliff

The model does not degrade gracefully under visual perturbation. It falls off a cliff at the first perturbation and then barely moves.

<div class="row mt-3">
<div class="col-sm-6">
{% include figure.liquid
   path="assets/img/vlm_blur_curve.png"
   class="img-fluid rounded z-depth-1"
   caption="Gaussian blur degradation. Accuracy collapses 23.5pp at σ=2, then drifts only 4.9pp further across all remaining levels." %}
</div>
<div class="col-sm-6">
{% include figure.liquid
   path="assets/img/vlm_mask_curve.png"
   class="img-fluid rounded z-depth-1"
   caption="Patch masking degradation. 10% masking costs 23.2pp. Going from 10% to 75% masking costs only 1.8pp more — a 12.9× asymmetry." %}
</div>
</div>

| Condition | Accuracy |
|---|---|
| C0 — Full image | **55.3%** |
| Blur σ=2 | 31.8% |
| Blur σ=5 | 27.7% |
| Blur σ=10 | 27.3% |
| Blur σ=20 | 26.9% |
| C1 — No image | 27.1% |

The model reaches near text-only floor at the first perturbation. The same pattern holds for masking: 10% masking costs 23.2pp, but going all the way to 75% masking costs only 1.8pp more. The cliff is consistent across both degradation types.

This has a practical implication for robustness evaluation. Most benchmarks apply graded corruptions and report accuracy at multiple severity levels, implicitly assuming a meaningful slope. When the model falls from 55% to 32% at the first step and then flatlines, the intermediate severity levels are measuring nothing distinct. For this model, binary presence/absence of uncorrupted visual input may be more diagnostic than graded severity.

---

## Finding 2 — What Level of the Pipeline Matters

The three localisation conditions — C1, C2, C3 — triangulate where in the visual processing pipeline the model's dependence actually lives. Each condition removes or disrupts visual information at a different stage:

- **C1** removes tokens entirely — zero visual input in the sequence
- **C2** corrupts activation statistics at the encoder source, but keeps tokens present
- **C3** destroys spatial arrangement of content after the MLP bridge, but preserves per-token statistics and token count

If the model depended primarily on token *presence*, C1 should produce the largest distributional shift. If it depended on encoder *statistics*, C2 should. If it depended on spatial *content*, C3 should.

<div class="row mt-3">
<div class="col-sm-8 offset-sm-2">
{% include figure.liquid
   path="assets/img/vlm_tvd_bar.png"
   class="img-fluid rounded z-depth-1"
   caption="TVD from baseline across the four key conditions. TVD(C2) > TVD(C1) > TVD(C3), with text shuffle (C7) as reference." %}
</div>
</div>

| Condition | TVD from C0 | Accuracy |
|---|---|---|
| C1 — Token omission | 0.633 | 27.1% |
| C2 — Encoder noise | **0.647** | 27.1% |
| C3 — Spatial shuffle | 0.564 | 29.4% |
| C7 — Text shuffle | 0.650 | 26.8% |

The ordering is TVD(C2) > TVD(C1) > TVD(C3). Two things stand out.

First, C1 and C2 produce identical accuracy — 27.1% — but different distributional shifts. Accuracy alone would say these conditions are equivalent. TVD reveals they are not: corrupting encoder output shifts the distribution more than removing tokens entirely. The model reacts differently to *no visual signal* versus *corrupted visual signal*, even when the end accuracy is the same.

Second, C3 produces the smallest distributional shift of the three. Destroying spatial arrangement — which object is where, what the layout of the image is — matters less than corrupting what comes out of the encoder. The model is more sensitive to encoder-level statistics than to spatial content specifically.

The category breakdown shows this is not uniform across task types:

| Benchmark | C0 | C1 (text-only) | TVD(C2) | TVD(C3) |
|---|---|---|---|---|
| SEEDBench_IMG | 52.6% | 23.8% | 0.826 | 0.812 |
| MMBench | 81.7% | 30.3% | 0.726 | 0.723 |
| ScienceQA | 52.2% | 40.6% | 0.646 | 0.665 |
| AI2D | 39.8% | 27.6% | 0.544 | 0.416 |
| MathVista | 45.5% | 23.2% | 0.472 | **0.223** |
| MMMU | 37.0% | 32.0% | 0.405 | 0.327 |

MathVista shows TVD(C3) = 0.223 — spatial arrangement of visual content barely affects the output distribution on mathematical reasoning tasks. SEEDBench and MMBench show TVD(C3) near 0.81 — for scene understanding and visual grounding tasks, spatial content matters substantially. Visual dependence is a joint property of model and task, not a fixed model-level constant.

---

## Finding 3 — Spatial Coherence and Token Count Are Independent

C5 (masking) and C6 (token reduction) allow a direct comparison: C5 destroys spatial coherence while preserving token count; C6 reduces token count while preserving spatial coherence. If they produce similar results, the dimensions are not independent. If they diverge, they are.

| Condition | What it disrupts | Accuracy |
|---|---|---|
| C5d — 75% masked | Spatial coherence (count preserved) | 30.3% |
| C6 — Token reduction | Token count (coherence preserved) | 30.8% |
| C1 — No image | Both | 27.1% |

At matched severity, the two conditions produce nearly identical accuracy (30.3% vs 30.8%). This suggests spatial coherence and token count are not independently weighted by the model — disrupting either one produces a similar floor, close to but above the no-image baseline. The model treats them as roughly equivalent degradation of the visual signal rather than separable components.

---

## Summary

Three things the experiment establishes:

**1.** The model's visual processing has no graceful degradation regime. It either receives uncorrupted visual input and uses it, or it falls to near text-only performance immediately. Robustness evaluations based on graded severity curves will miss this structure.

**2.** The model is more sensitive to encoder activation statistics than to spatial content. Distributional shifts are largest when encoder output is corrupted (C2), smaller when spatial arrangement is destroyed (C3), with token omission (C1) in between. Accuracy cannot detect this distinction — C1 and C2 produce identical accuracy but different TVDs.

**3.** Spatial coherence and token count produce equivalent degradation at matched severity levels, suggesting the model does not independently weight these two dimensions of visual information.

---

## Technical Notes

**Dataset:** MMStar (1,500 questions, 6 visual reasoning categories, 4-option multiple choice). Answer labels replaced with neutral format (Option A–D) throughout all conditions to prevent lexical prior leakage.

**Model:** Qwen2.5-VL-7B-Instruct, 8-bit quantised, inference only. Chain-of-thought disabled; direct-answer mode throughout.

**Metric:** TVD = ½ Σ|P(y\|C0) − P(y\|Cc)| over the 4-option output distribution. Computed per item; values reported as means across 1,500 items.

**C3 implementation note:** Post-projection embeddings permuted jointly with their M-RoPE positional encodings for Qwen2.5-VL, ensuring position-content pairing is preserved within each shuffled token rather than creating a positional contradiction.