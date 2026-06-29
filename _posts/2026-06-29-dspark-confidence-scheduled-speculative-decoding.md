---
title: "DSpark: Confidence-Scheduled Speculative Decoding"
description: "An illustrated intuition ladder for DSpark: why speculative decoding is exact, why accepted prefixes behave like survival processes, and how DSpark combines semi-autoregressive drafting with load-aware verification."
toc: true
badges: true
comments: true
sticky_rank: 1
author: Abhishek Sharma
image: images/dspark_confidence_scheduled_speculative_decoding.svg
categories: [llm, inference, speculative-decoding, systems, deepseek]
---

> **TL;DR.** Speculative decoding is not “let a small model guess and hope.” It is an exact coupling protocol: cheap computation proposes a future, the target model certifies the reusable probability mass, and residual sampling repairs the first failure. DSpark improves this protocol in two places: a **semi-autoregressive drafter** repairs suffix coherence, and a **confidence scheduler** spends target verification only where the expected prefix survival justifies the hardware cost.

<style>
.dspark-post {
  --ink: #172033;
  --muted: #5b6b80;
  --line: #dce5f0;
  --blue: #2f80ed;
  --blue-soft: #e8f2ff;
  --orange: #f2994a;
  --orange-soft: #fff0df;
  --green: #27ae60;
  --green-soft: #e5f7ec;
  --red: #eb5757;
  --red-soft: #ffe7e7;
  --purple: #7b61ff;
  --purple-soft: #eeeaff;
  color: var(--ink);
}
.dspark-post p { line-height: 1.72; }
.dspark-post h2 { margin-top: 2.7rem; border-top: 1px solid var(--line); padding-top: 2rem; }
.dspark-post h3 { margin-top: 2rem; }
.dspark-post .lede { font-size: 1.08rem; color: var(--muted); }
.dspark-post .viz {
  border: 1px solid var(--line);
  border-radius: 22px;
  padding: 1.2rem;
  margin: 1.4rem 0;
  background: linear-gradient(135deg, #ffffff, #f8fbff);
  box-shadow: 0 12px 30px rgba(31, 49, 75, 0.07);
  overflow: hidden;
}
.dspark-post .caption { color: var(--muted); font-size: .92rem; margin-top: .6rem; }
.dspark-post .grid-2 { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 1rem; }
.dspark-post .grid-3 { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: .9rem; }
@media (max-width: 760px) { .dspark-post .grid-2, .dspark-post .grid-3 { grid-template-columns: 1fr; } }
.dspark-post .card { border: 1px solid var(--line); border-radius: 18px; padding: 1rem; background: white; }
.dspark-post .small { font-size: .93rem; color: var(--muted); }
.dspark-post .pill { display: inline-block; border-radius: 999px; padding: .22rem .6rem; background: var(--blue-soft); color: #1255aa; font-weight: 700; font-size: .82rem; }
.dspark-post .token-row { display: flex; align-items: center; gap: .5rem; flex-wrap: wrap; }
.dspark-post .tok { min-width: 2.2rem; height: 2.2rem; border-radius: 10px; display: inline-flex; align-items: center; justify-content: center; font-weight: 800; background: var(--blue-soft); color: #1255aa; border: 1px solid #cfe3ff; }
.dspark-post .tok.good { background: var(--green-soft); color: #177245; border-color: #bdeccd; }
.dspark-post .tok.bad { background: var(--red-soft); color: #b42318; border-color: #ffd0d0; }
.dspark-post .tok.warn { background: #fff7d6; color: #946200; border-color: #ffe79b; }
.dspark-post .tok.ghost { background: #f2f4f7; color: #98a2b3; border-color: #e4e7ec; text-decoration: line-through; }
.dspark-post .arrow { color: #8a99aa; font-weight: 800; }
.dspark-post .pipeline { display: flex; gap: .8rem; align-items: center; flex-wrap: wrap; }
.dspark-post .stage { border-radius: 18px; border: 1px solid var(--line); padding: .9rem 1rem; background: #fff; min-width: 145px; text-align: center; }
.dspark-post .stage strong { display:block; }
.dspark-post .bar-wrap { display:flex; align-items:center; gap:.55rem; margin:.42rem 0; }
.dspark-post .bar-label { width: 7.2rem; font-weight: 700; font-size: .9rem; }
.dspark-post .bar { height: 1.05rem; border-radius: 999px; background: #eef2f7; overflow:hidden; flex:1; }
.dspark-post .bar > span { display:block; height:100%; border-radius:999px; background: linear-gradient(90deg, var(--blue), var(--purple)); }
.dspark-post .sand { display:grid; grid-template-columns: 90px repeat(4, 1fr); gap:.45rem; align-items:end; }
.dspark-post .sand div { text-align:center; font-size:.85rem; }
.dspark-post .sand .label { text-align:right; font-weight:800; color:var(--muted); }
.dspark-post .pile { display:flex; align-items:end; justify-content:center; height:68px; }
.dspark-post .pile span { display:block; width:36px; border-radius:8px 8px 0 0; }
.dspark-post .draft-pile { background: var(--blue); opacity:.75; }
.dspark-post .target-pile { background: var(--orange); opacity:.85; }
.dspark-post .shared-pile { background: var(--green); opacity:.85; }
.dspark-post .survival { display:flex; align-items:flex-end; gap:.8rem; min-height:160px; }
.dspark-post .stone { width:74px; border-radius:16px 16px 8px 8px; background: linear-gradient(#d7ecff, #88c7ff); border:1px solid #b6dcff; display:flex; align-items:center; justify-content:center; font-weight:800; color:#0b4b89; animation: bob 2.8s ease-in-out infinite; }
.dspark-post .stone:nth-child(2) { animation-delay: .15s; }
.dspark-post .stone:nth-child(3) { animation-delay: .30s; }
.dspark-post .stone:nth-child(4) { animation-delay: .45s; }
.dspark-post .stone:nth-child(5) { animation-delay: .60s; }
@keyframes bob { 0%,100%{ transform: translateY(0);} 50%{ transform: translateY(-7px);} }
.dspark-post .moving-token { animation: slideToken 4s ease-in-out infinite; }
@keyframes slideToken { 0%{ transform: translateX(0); opacity:.35;} 30%{ opacity:1;} 70%{ transform: translateX(260px); opacity:1;} 100%{ transform: translateX(260px); opacity:.35;} }
.dspark-post .blink { animation: blink 1.4s ease-in-out infinite; }
@keyframes blink { 0%, 100%{ opacity: 1;} 50%{ opacity: .35;} }
.dspark-post .formula { background: #fbfcff; border: 1px solid var(--line); border-radius: 18px; padding: 1rem; overflow-x:auto; }
.dspark-post .note { border-left: 5px solid var(--blue); background: var(--blue-soft); padding: 1rem; border-radius: 14px; }
.dspark-post .warning { border-left: 5px solid var(--orange); background: var(--orange-soft); padding: 1rem; border-radius: 14px; }
.dspark-post .goodbox { border-left: 5px solid var(--green); background: var(--green-soft); padding: 1rem; border-radius: 14px; }
.dspark-post table { font-size: .95rem; }
</style>

<div class="dspark-post" markdown="1">

## n01. The map of the paper

DSpark starts from two observations about speculative decoding.

First, **parallel drafters are fast** because they propose many future tokens in one pass. But they often lose coherence later in the block because position 5 did not condition on the actual sampled positions 1–4.

Second, **verification is not free**. Under heavy serving load, every low-confidence draft token sent to the target model consumes batch capacity that could have served another user.

So DSpark adds two narrow fixes:

<div class="viz pipeline">
  <div class="stage"><span class="pill">bulk</span><strong>Parallel backbone</strong><span class="small">draft many positions once</span></div>
  <span class="arrow">→</span>
  <div class="stage"><span class="pill">coordination</span><strong>Sequential head</strong><span class="small">repair token dependencies</span></div>
  <span class="arrow">→</span>
  <div class="stage"><span class="pill">budget</span><strong>Confidence scheduler</strong><span class="small">verify only useful prefix</span></div>
  <span class="arrow">→</span>
  <div class="stage"><span class="pill">exactness</span><strong>Target verifier</strong><span class="small">preserve target law</span></div>
</div>

The rest of the post builds the intuition ladder in the following order: serving objective, verification waste, exact speculative sampling, survival length, latency equation, drafter architectures, DSpark’s design, training, scheduling, and production results.

---

## n02. Serving objective: latency versus throughput

A production serving system has two goals:

| Metric | Question | Restaurant analogy |
|---|---|---|
| **Per-user tok/s** | How fast does one user see tokens? | How quickly one table gets food |
| **Tokens/s/GPU** | How many total tokens does the GPU produce? | How many plates the kitchen outputs |

These are not the same. A system can make one VIP user very fast while leaving the GPU under-utilized. Or it can fully utilize the GPU while every individual user waits too long.

<div class="viz">
  <svg viewBox="0 0 760 300" width="100%" role="img" aria-label="Pareto frontier sketch">
    <rect x="0" y="0" width="760" height="300" rx="22" fill="#fff"/>
    <line x1="70" y1="245" x2="710" y2="245" stroke="#9aa8b5" stroke-width="2"/>
    <line x1="70" y1="245" x2="70" y2="35" stroke="#9aa8b5" stroke-width="2"/>
    <text x="330" y="282" font-size="16" fill="#5b6b80">aggregate tokens/s/GPU</text>
    <text x="18" y="145" font-size="16" fill="#5b6b80" transform="rotate(-90 18,145)">per-user tok/s</text>
    <path d="M95 220 C180 190, 245 175, 315 165 C390 154, 470 129, 555 85" fill="none" stroke="#b7c2cf" stroke-width="7" stroke-linecap="round"/>
    <path d="M120 195 C220 155, 320 135, 420 112 C520 90, 600 68, 675 45" fill="none" stroke="#2f80ed" stroke-width="7" stroke-linecap="round"/>
    <text x="220" y="198" font-size="18" font-weight="700" fill="#7b8794">old frontier</text>
    <text x="500" y="82" font-size="18" font-weight="700" fill="#2f80ed">DSpark frontier</text>
  </svg>
  <div class="caption">A serving improvement is strongest when the whole frontier moves outward: same throughput with lower latency, or same latency with higher throughput.</div>
</div>

For a rough model, if a GPU serves `N` concurrent users and produces aggregate rate `R`, then one user sees roughly:

$$
r \approx \frac{\eta(N) R_{\max}}{N}
$$

where $\eta(N)$ captures batching efficiency, scheduler overhead, memory pressure, and verification waste.

DSpark’s goal is not merely “better draft tokens.” It tries to improve this serving efficiency curve.

---

## n06. Verification waste under load

Speculative decoding often sounds like:

> “Draft more tokens, then verify more tokens.”

The hidden assumption is that verification is free. It is not.

If five users each propose four draft tokens, the target verifier sees twenty candidate positions instead of five ordinary decode positions.

<div class="viz grid-2">
  <div class="card">
    <strong>Ordinary decode batch</strong>
    <div class="token-row" style="margin-top:.8rem">
      <span class="tok">T</span><span class="tok">T</span><span class="tok">T</span><span class="tok">T</span><span class="tok">T</span>
    </div>
    <p class="small">Five users, five target positions.</p>
  </div>
  <div class="card">
    <strong>Speculative verification batch</strong>
    <div class="token-row" style="margin-top:.8rem">
      <span class="tok">a</span><span class="tok">b</span><span class="tok warn">c</span><span class="tok bad">d</span>
      <span class="tok">e</span><span class="tok">f</span><span class="tok warn">g</span><span class="tok bad">h</span>
    </div>
    <p class="small">Long suffixes may be paid for even when they almost never survive.</p>
  </div>
</div>

Under light load, the GPU may have spare capacity. Wasting a few verification slots barely matters.

Under heavy load, the same extra token has opportunity cost: it delays another request, increases queueing, or lowers total tokens/s/GPU.

<div class="warning">
  <strong>Key systems shift.</strong> The question changes from “is this token individually plausible?” to “is verifying this token the best use of scarce target-model capacity right now?”
</div>

That is why DSpark needs a scheduler, not only a better drafter.

---

## n03. Speculative decoding invariant: propose cheaply, certify expensively

Ordinary autoregressive decoding performs one target-model step per token:

<div class="viz token-row">
  <span class="tok">context</span><span class="arrow">Target →</span><span class="tok good">x₁</span><span class="arrow">Target →</span><span class="tok good">x₂</span><span class="arrow">Target →</span><span class="tok good">x₃</span>
</div>

Speculative decoding asks a cheap draft process to propose a possible future:

<div class="viz">
  <div class="token-row">
    <span class="tok">c</span><span class="arrow">draft →</span><span class="tok">E</span><span class="tok">F</span><span class="tok">G</span><span class="tok">H</span>
  </div>
  <div class="token-row" style="margin-top:.8rem">
    <span class="tok ghost"> </span><span class="arrow">target scores in one pass →</span><span class="tok good">p₁</span><span class="tok good">p₂</span><span class="tok good">p₃</span><span class="tok good">p₄</span>
  </div>
</div>

The target cannot generate all future positions independently. But once the draft supplies the hypothetical sequence, the target can evaluate:

$$
p_t(E \mid c),\quad
p_t(F \mid cE),\quad
p_t(G \mid cEF),\quad
p_t(H \mid cEFG)
$$

in a single parallel verification pass.

### One-token exactness

Let the draft distribution be $q(x)$ and the target distribution be $p(x)$. The draft samples $X \sim q$ and accepts proposal $x$ with probability:

$$
a(x)=\min\left(1,\frac{p(x)}{q(x)}\right)
$$

The probability that token $x$ is proposed and accepted is:

$$
q(x)a(x)=q(x)\min\left(1,\frac{p(x)}{q(x)}\right)=\min(q(x),p(x))
$$

This is the shared probability mass.

<div class="viz">
  <div class="sand">
    <div></div><div>A</div><div>B</div><div>C</div><div>D</div>
    <div class="label">Draft</div>
    <div class="pile"><span class="draft-pile" style="height:58px"></span></div>
    <div class="pile"><span class="draft-pile" style="height:34px"></span></div>
    <div class="pile"><span class="draft-pile" style="height:20px"></span></div>
    <div class="pile"><span class="draft-pile" style="height:45px"></span></div>
    <div class="label">Target</div>
    <div class="pile"><span class="target-pile" style="height:44px"></span></div>
    <div class="pile"><span class="target-pile" style="height:60px"></span></div>
    <div class="pile"><span class="target-pile" style="height:36px"></span></div>
    <div class="pile"><span class="target-pile" style="height:22px"></span></div>
    <div class="label">Shared</div>
    <div class="pile"><span class="shared-pile" style="height:44px"></span></div>
    <div class="pile"><span class="shared-pile" style="height:34px"></span></div>
    <div class="pile"><span class="shared-pile" style="height:20px"></span></div>
    <div class="pile"><span class="shared-pile" style="height:22px"></span></div>
  </div>
  <div class="caption">Accepted mass is the overlap: $\min(p(x),q(x))$.</div>
</div>

The unmatched target mass must be supplied by a residual correction distribution:

$$
r(x) \propto (p(x)-q(x))_+
$$

This is why “fall back to the target distribution” is imprecise. If rejection sampled directly from $p$, the algorithm would double-count target mass already covered by accepted draft proposals.

Example:

$$
q=(0.6,0.3,0.1),\qquad p=(0.4,0.4,0.2)
$$

Accepted mass is $(0.4,0.3,0.1)$, total $0.8$. The residual mass is $(0,0.1,0.1)$, so after normalization $r=(0,0.5,0.5)$. The final distribution is:

$$
(0.4,0.3,0.1)+0.2(0,0.5,0.5)=(0.4,0.4,0.2)=p
$$

### Coupling view

Speculative decoding constructs a coupling of draft and target samples. The acceptance probability is:

$$
\alpha=\sum_x \min(p(x),q(x))=1-\operatorname{TV}(p,q)
$$

So a draft model is good when its probability mass overlaps the target’s mass, not merely when it shares the same argmax.

### Why only the longest accepted prefix survives

Suppose the draft proposes:

<div class="viz token-row">
  <span class="tok good">E ✓</span><span class="arrow">→</span><span class="tok good">F ✓</span><span class="arrow">→</span><span class="tok bad">G ✗</span><span class="arrow">→</span><span class="tok ghost">H</span>
</div>

After $G$ is rejected, the correct next token is some $G^\star$. The real context becomes $c,E,F,G^\star$, not $c,E,F,G$.

The old `H` was generated on the wrong branch:

<div class="viz">
<pre style="margin:0; font-size:1rem; line-height:1.45">context ── E ── F ── G  ── H    old branch
                    \
                     G* ── ?    actual branch</pre>
</div>

The causal invariant is:

> Reuse is valid only inside the causal region whose assumptions remain unchanged.

For a linear autoregressive chain, that valid region is exactly the longest accepted prefix.

---

## n04. Prefix survival and accepted length

Accepted prefix length is a survival process.

Imagine stepping stones:

<div class="viz survival">
  <div class="stone" style="height:62px">90%</div>
  <div class="stone" style="height:54px">80%</div>
  <div class="stone" style="height:48px">70%</div>
  <div class="stone" style="height:64px">95%</div>
  <div class="stone" style="height:58px">90%</div>
</div>

To stand on stone 3, stone 1 and stone 2 must also have survived:

$$
P(\text{reach stone 3})=0.9\times0.8\times0.7=0.504
$$

The same is true for draft tokens. If token acceptances are $c_1,c_2,c_3,c_4$, then:

$$
P(\text{accept token 4})=c_1c_2c_3c_4
$$

And the expected number of output tokens from a round with four draft tokens is:

$$
L=1+c_1+c_1c_2+c_1c_2c_3+c_1c_2c_3c_4
$$

The leading `1` is the guaranteed output token: either the first accepted draft token or a correction token. If all drafts survive, the target often contributes a bonus token.

<div class="note">
  <strong>Why early tokens matter so much.</strong> Improving $c_1$ helps every downstream prefix. Improving $c_4$ helps only the length-4 prefix.
</div>

This explains why DSpark pays special attention to early-token quality and why the training objective weights early positions more heavily.

---

## n05. Latency equation: three levers

Speculative decoding happens in rounds:

<div class="viz pipeline">
  <div class="stage"><strong>draft block</strong><span class="small">cost $T_{draft}$</span></div>
  <span class="arrow">→</span>
  <div class="stage"><strong>verify prefix</strong><span class="small">cost $T_{verify}$</span></div>
  <span class="arrow">→</span>
  <div class="stage"><strong>emit tokens</strong><span class="small">accepted length $\tau$</span></div>
</div>

So latency per token is an efficiency ratio:

$$
L=\frac{T_{draft}+T_{verify}}{\tau}
$$

There are only three levers:

| Lever | How to improve it | DSpark component |
|---|---|---|
| Lower $T_{draft}$ | Avoid heavy autoregressive draft loops | parallel backbone |
| Lower effective $T_{verify}$ | Avoid low-value verification slots | confidence scheduler |
| Increase $\tau$ | Improve accepted prefix length | sequential head + better training |

The important lesson is that optimizing one variable can hurt another. A very accurate draft that costs almost as much as the target may lose. A tiny draft that is fast but rarely accepted also loses.

The real objective is the whole ratio.

---

## n07. Autoregressive drafters: strong but serial

An autoregressive drafter proposes:

<div class="viz token-row">
  <span class="tok">E</span><span class="arrow">→</span><span class="tok">F</span><span class="arrow">→</span><span class="tok">G</span><span class="arrow">→</span><span class="tok">H</span><span class="arrow">→</span><span class="tok">I</span>
</div>

Each token depends on the previous draft token. This gives strong coherence, because position 4 knows what positions 1–3 actually became.

But the critical path has length $\gamma$. If one draft step takes 1 ms, seven draft tokens cost roughly 7 ms.

Autoregressive drafting improves $\tau$ but hurts $T_{draft}$.

Parallel drafting does the opposite: it lowers $T_{draft}$ by producing all positions at once, but may reduce $\tau$ because later positions do not know the realized earlier tokens.

The design question becomes:

> Can we preserve enough autoregressive dependency modeling without paying full autoregressive latency?

DSpark’s answer is: put the heavy computation in a parallel backbone, then add a tiny sequential correction head.

---

## n08. Parallel drafters and DFlash

A parallel drafter predicts all positions simultaneously:

<div class="viz">
  <svg viewBox="0 0 760 220" width="100%">
    <rect width="760" height="220" rx="18" fill="#fff"/>
    <rect x="290" y="30" width="180" height="52" rx="18" fill="#e8f2ff" stroke="#b8d8ff"/>
    <text x="380" y="64" text-anchor="middle" font-size="18" font-weight="800" fill="#1255aa">Context</text>
    <line x1="380" y1="82" x2="160" y2="140" stroke="#8aa0b6" stroke-width="3"/>
    <line x1="380" y1="82" x2="305" y2="140" stroke="#8aa0b6" stroke-width="3"/>
    <line x1="380" y1="82" x2="455" y2="140" stroke="#8aa0b6" stroke-width="3"/>
    <line x1="380" y1="82" x2="600" y2="140" stroke="#8aa0b6" stroke-width="3"/>
    <g font-size="24" font-weight="800" fill="#1255aa" text-anchor="middle">
      <rect x="132" y="140" width="56" height="46" rx="12" fill="#e8f2ff"/><text x="160" y="171">E</text>
      <rect x="277" y="140" width="56" height="46" rx="12" fill="#e8f2ff"/><text x="305" y="171">F</text>
      <rect x="427" y="140" width="56" height="46" rx="12" fill="#e8f2ff"/><text x="455" y="171">G</text>
      <rect x="572" y="140" width="56" height="46" rx="12" fill="#e8f2ff"/><text x="600" y="171">H</text>
    </g>
  </svg>
</div>

DFlash takes this idea further. Since the sequential loop has been removed, it can spend the saved latency on a deeper backbone. It also injects rich target-model context through KV features, so the draft model does not have to rediscover everything the target already understood.

The trade is:

| Architecture | Strength | Weakness |
|---|---|---|
| Autoregressive | strong token-to-token coherence | sequential critical path |
| Parallel / DFlash | very low draft latency, deeper backbone possible | weak intra-block dependency |

The meta-principle:

> Eliminating a dependency does not simply reduce latency. It creates a computational budget that can be reinvested into better representations.

---

## n09. Context injection and shared heads

DFlash/DSpark borrows two things from the target model:

1. **Internal context**: target hidden states summarize what the prompt means.
2. **Output geometry**: the frozen target LM head maps hidden states into the same vocabulary coordinate system.

The paper describes target-layer context injection as:

$$
H_{ctx}=\operatorname{RMSNorm}\left(W_c[H_{l_1};H_{l_2};\dots;H_{l_m}]\right)
$$

Then draft attention sees both target context and draft states:

$$
K=[K_{target-context};K_{draft}],\qquad V=[V_{target-context};V_{draft}]
$$

For the prompt “The capital of France is”, the target’s hidden state may already encode:

<div class="viz token-row">
  <span class="tok">France</span><span class="arrow">+</span><span class="tok">capital question</span><span class="arrow">+</span><span class="tok good">city expected</span><span class="arrow">≈</span><span class="tok good">Paris</span>
</div>

The draft model receives this compressed understanding instead of reconstructing it from scratch.

Shared LM head means the draft’s hidden state is interpreted using the target’s vocabulary geometry. It does not guarantee identical probabilities, but it improves alignment at the interface.

> A cheap component need not duplicate an expensive component if it can reuse the expensive component’s intermediate representation.

---

## n10. Multi-modal collision

Parallel prediction can preserve correct token marginals while destroying the joint sequence.

Suppose two continuations are valid:

<div class="viz">
<pre style="margin:0; line-height:1.55">Path A: of ─────► course
Path B: no ─────► problem</pre>
</div>

The true joint distribution could be:

$$
P(\text{of course})=0.5,\qquad P(\text{no problem})=0.5
$$

with cross-combinations having probability zero.

But an independent parallel predictor sees correct marginals:

$$
P(X_1=\text{of})=P(X_1=\text{no})=0.5
$$

$$
P(X_2=\text{course})=P(X_2=\text{problem})=0.5
$$

and then samples as if:

$$
\hat P(X_1,X_2)=P(X_1)P(X_2)
$$

Now it invents:

<div class="viz grid-2">
  <div class="card"><span class="tok good">of</span> <span class="tok good">course</span><p class="small">valid</p></div>
  <div class="card"><span class="tok good">no</span> <span class="tok good">problem</span><p class="small">valid</p></div>
  <div class="card"><span class="tok bad">of</span> <span class="tok bad">problem</span><p class="small">mode collision</p></div>
  <div class="card"><span class="tok bad">no</span> <span class="tok bad">course</span><p class="small">mode collision</p></div>
</div>

This is not a low-capacity problem. Even a perfect independent predictor can learn every position’s marginal and still fail to preserve which tokens belong to the same world.

The missing ingredient is branch identity.

> Dependencies are not merely computational obstacles. Some dependencies are communication channels.

---

## n11. Conditional acceptance curves

Overall acceptance hides survival effects. Conditional acceptance asks:

> How often is position $k$ accepted, **given that all previous positions survived**?

For position $k$:

$$
P(A_k \mid A_1,\ldots,A_{k-1})=
\frac{\#\text{ times position }k\text{ accepted}}
{\#\text{ times positions }1..k-1\text{ survived}}
$$

Example: out of 100 blocks, position 1 survives 80 times and position 2 survives 60 times. Naively position 2 acceptance is 60%. Conditionally it is:

$$
P(A_2\mid A_1)=60/80=75\%
$$

This matters because later positions may look bad only because earlier gates killed them. Conditional curves separate:

- strong initial prediction;
- suffix coherence decay;
- whether the architecture maintains dependencies across the block.

This is why the DSpark paper’s position-wise analysis is so useful: it shows DFlash’s strong early-token capacity, Eagle3’s sequential coherence, and DSpark’s attempt to combine both.

---

## n12. DSpark design thesis

DSpark adds one small fix for each bottleneck:

<div class="viz grid-2">
  <div class="card">
    <span class="pill">problem</span>
    <h4>Parallel suffix decay</h4>
    <p class="small">Later draft tokens do not condition on actual earlier draft tokens.</p>
    <strong>Fix: tiny sequential head</strong>
  </div>
  <div class="card">
    <span class="pill">problem</span>
    <h4>Verification waste</h4>
    <p class="small">Long low-confidence suffixes occupy target-model batch slots.</p>
    <strong>Fix: confidence scheduler</strong>
  </div>
</div>

Given prompt tokens `ABC`, the target first produces anchor `D`. Then DSpark drafts from that anchor:

<div class="viz">
  <div class="token-row"><span class="tok good">ABC</span><span class="arrow">Target →</span><span class="tok good">D</span><span class="arrow">anchor</span></div>
  <div class="token-row" style="margin-top:.7rem"><span class="tok">Parallel</span><span class="arrow">→</span><span class="tok">E</span><span class="tok">G</span><span class="tok">J</span><span class="tok">K</span></div>
  <div class="token-row" style="margin-top:.7rem"><span class="tok warn">Sequential head</span><span class="arrow">→</span><span class="tok good">E</span><span class="tok good">F</span><span class="tok good">H</span><span class="tok warn">I</span></div>
  <div class="token-row" style="margin-top:.7rem"><span class="tok">Scheduler</span><span class="arrow">→</span><span class="tok good">E</span><span class="tok good">F</span><span class="tok good">H</span><span class="tok ghost">I</span></div>
</div>

The backbone writes quickly. The head proofreads line by line. The scheduler tears off the low-confidence suffix before sending it to the target editor.

The deeper engineering principle:

> Preserve the high-performance path. Repair only the failure mode.

---

## n13. Parallel backbone stage

DSpark separates expensive global prediction from cheap sequential correction.

<div class="viz">
<pre style="margin:0; line-height:1.55">Anchor + masks ── heavy parallel backbone ──► h₁ h₂ h₃ h₄ h₅ h₆ h₇
                                              U₁ U₂ U₃ U₄ U₅ U₆ U₇
                                                │  │  │  │  │  │  │
                                                └── lightweight sequential head ──► x₁ → x₂ → ... → x₇</pre>
</div>

For a conventional autoregressive drafter:

$$
T_{draft}\approx \sum_{k=1}^{\gamma}T_{heavy}(k)
$$

For DSpark:

$$
T_{draft}\approx T_{heavy-parallel}+\gamma T_{tiny-head}
$$

This is beneficial only while:

$$
\gamma T_{tiny-head}\ll T_{heavy-parallel}
$$

So the design rule is:

> Put intelligence in the parallel path; put only dependency repair in the serial path.

---

## n14. Markov sequential head

The Markov head adds a low-rank transition bias based only on the previous sampled token:

$$
p_k(v\mid x_{k-1})=\operatorname{softmax}\left(U_k(v)+B(x_{k-1},v)\right)
$$

If the previous token is `of`, the head can boost `course` and suppress `problem`.

<div class="viz grid-2">
  <div class="card">
    <strong>Backbone logits</strong>
    <div class="bar-wrap"><span class="bar-label">course</span><span class="bar"><span style="width:82%"></span></span></div>
    <div class="bar-wrap"><span class="bar-label">problem</span><span class="bar"><span style="width:80%"></span></span></div>
  </div>
  <div class="card">
    <strong>After Markov bias: previous = of</strong>
    <div class="bar-wrap"><span class="bar-label">course</span><span class="bar"><span style="width:96%"></span></span></div>
    <div class="bar-wrap"><span class="bar-label">problem</span><span class="bar"><span style="width:45%"></span></span></div>
  </div>
</div>

A full transition matrix over a 100k vocabulary would be huge. DSpark uses a low-rank factorization:

$$
B=W_1W_2
$$

with $W_1\in\mathbb{R}^{|V|\times r}$ and $W_2\in\mathbb{R}^{r\times |V|}$. For a previous token $x_{k-1}$:

$$
B(x_{k-1},:)=W_1[x_{k-1},:]W_2
$$

This gives a cheap vocabulary-wide transition correction.

The Markov head does not understand the whole sentence. It repairs the dominant local dependency that parallel drafting removed.

---

## n15. RNN sequential head

The RNN head keeps a small running state instead of only the previous token:

$$
s_k=f(s_{k-1},E(x_{k-1}),h_k)
$$

$$
z_k=U_k+W_os_k
$$

This helps when the right next token depends on longer draft history.

Example:

> “The keys to the cabinet …”

A Markov head sees only `cabinet` and may prefer `is`. An RNN state can remember the earlier plural subject `keys` and boost `are`.

The tradeoff is deployment cost: recurrent state reads/writes, less fusion, stricter ordering, and batching complexity.

So the production question is not:

> Which head has the highest acceptance?

It is:

> Which head gives the best end-to-end latency and operational reliability?

In the paper’s production setup, DSpark uses the Markov head.

---

## n17. Confidence head

The confidence head predicts conditional survival:

$$
c_k=P(A_k=1\mid A_1=\cdots=A_{k-1}=1)
$$

It is not generating better tokens. It is deciding whether spending target verification on this position is likely to be useful.

<div class="viz token-row">
  <span class="tok good">x₁<br/>.97</span>
  <span class="tok good">x₂<br/>.94</span>
  <span class="tok warn">x₃<br/>.72</span>
  <span class="tok bad">x₄<br/>.20</span>
  <span class="tok ghost">x₅<br/>.69</span>
</div>

A later token can have decent conditional confidence but poor cumulative value if an earlier position is weak.

If:

$$
c_1=0.97,\quad c_2=0.94,\quad c_3=0.88,\quad c_4=0.20,\quad c_5=0.72
$$

then the chance of reaching and accepting token 5 is:

$$
0.97\cdot0.94\cdot0.88\cdot0.20\cdot0.72\approx0.12
$$

The confidence head becomes useful only when connected to action:

$$
\text{prediction}\rightarrow\text{expected utility}\rightarrow\text{resource decision}
$$

---

## n16. Total variation label

The paper supervises confidence using the analytical expected acceptance rate:

$$
c_k^*=1-\frac12\lVert p_d^{(k)}-p_t^{(k)}\rVert_1
$$

This is the distribution overlap:

$$
c_k^*=\sum_v\min(p_d(v),p_t(v))
$$

Why? Because speculative acceptance for a sampled draft token $x\sim p_d$ is:

$$
\alpha(x)=\min\left(1,\frac{p_t(x)}{p_d(x)}\right)
$$

Taking expectation over the draft distribution:

$$
\mathbb{E}_{x\sim p_d}[\alpha(x)]
=\sum_x p_d(x)\min\left(1,\frac{p_t(x)}{p_d(x)}\right)
=\sum_x\min(p_d(x),p_t(x))
$$

This is better than an argmax-match label. Two distributions can choose different top tokens but still overlap heavily:

$$
p_d=(0.40,0.35,0.25),\qquad p_t=(0.35,0.40,0.25)
$$

The argmax differs, but the overlap is:

$$
0.35+0.35+0.25=0.95
$$

Speculative decoding cares about reusable probability mass, not exact top-token agreement.

---

## n22. Losslessness and non-anticipation

A scheduler must not use information caused by a sampled token to decide whether that same token should be included.

Valid order:

<div class="viz token-row">
  <span class="tok good">past info</span><span class="arrow">→</span><span class="tok good">decide budget</span><span class="arrow">→</span><span class="tok good">sample/verify</span>
</div>

Invalid order:

<div class="viz token-row">
  <span class="tok bad">sample future</span><span class="arrow">→</span><span class="tok bad">inspect future confidence</span><span class="arrow">→</span><span class="tok bad">decide whether to keep earlier token</span>
</div>

Suppose the target samples:

$$
P(A)=0.7,\qquad P(B)=0.3
$$

If `A` tends to lead to confident futures and `B` tends to lead to uncertain futures, a future-peeking scheduler would keep more `A` paths. The visible output might become 85/15 instead of 70/30.

That changes the target distribution.

Early stopping in the ideal scheduler prevents this: once the scheduler decides the next extension is not worth admitting, it stops and does not inspect later tokens to reconsider earlier decisions.

---

## n18. Training objective

DSpark trains three capabilities:

1. produce plausible tokens;
2. match the target distribution;
3. predict whether draft tokens will survive verification.

The combined objective is:

$$
\mathcal{L}=\alpha_{ce}\mathcal{L}_{ce}+\alpha_{tv}\mathcal{L}_{tv}+\alpha_{conf}\mathcal{L}_{conf}
$$

The paper reports default weights:

$$
\alpha_{ce}=0.1,\qquad \alpha_{tv}=0.9,\qquad \alpha_{conf}=1.0
$$

The relatively large TV term reflects the speculative decoding objective: interface compatibility with the target matters more than standalone next-token accuracy.

Position weighting emphasizes early tokens:

$$
w_k=\exp\left(-\frac{k-1}{\gamma}\right)
$$

because errors at early positions destroy more downstream opportunity.

Expected accepted length is:

$$
\mathbb{E}[L]=\sum_{j=1}^{\gamma}\prod_{k=1}^{j}c_k
$$

An early $c_i$ appears in more terms of this sum. That is the mathematical version of “the first stone supports the whole bridge.”

---

## n19. Sequential temperature scaling

The confidence head can rank tokens well but still be overconfident.

Example:

| Position | raw confidence | empirical survival |
|---|---:|---:|
| 1 | 0.98 | 0.94 |
| 2 | 0.92 | 0.82 |
| 3 | 0.80 | 0.65 |

Ranking is fine. Calibration is not.

For scheduling, calibration matters because prefix survival multiplies:

$$
S_j=\prod_{k=1}^{j}c_k
$$

Raw scores:

$$
0.95\cdot0.90\cdot0.85\cdot0.80\approx0.58
$$

Calibrated scores:

$$
0.90\cdot0.82\cdot0.75\cdot0.68\approx0.34
$$

That difference can flip whether verifying token 4 is worthwhile.

Temperature scaling changes a confidence logit $z_k$ via:

$$
\hat c_k=\sigma\left(\frac{z_k}{T_k}\right)
$$

DSpark uses **sequential** temperature scaling: calibrate position 1, freeze it, calibrate cumulative survival through position 2, freeze it, and continue left to right.

> Scores choose an ordering; calibrated probabilities determine how much resource to commit.

---

## n20. Scheduler objective

For a batch, the scheduler chooses how many draft tokens to verify per request.

The throughput objective is:

$$
\Theta=\tau^*\times \operatorname{SPS}(B)
$$

where:

- $\tau^*$ is expected accepted tokens per verification round;
- $\operatorname{SPS}(B)$ is the profiled number of target verification steps per second at batch size $B$.

Adding another verification token has two effects:

<div class="viz grid-2">
  <div class="card"><strong>Benefit</strong><p>Expected accepted tokens may rise.</p></div>
  <div class="card"><strong>Cost</strong><p>Verification batch grows, so SPS may fall.</p></div>
</div>

Example:

| Choice | expected accepted tokens | SPS | throughput |
|---|---:|---:|---:|
| verify 3 | 2.4 | 100 | 240 |
| verify 4, high survival | 3.2 | 95 | 304 |
| verify 4, low survival | 2.6 | 80 | 208 |

The scheduler should add the fourth token only in the high-survival case.

---

## n21. Greedy prefix allocation

Each request offers prefix extensions with decreasing cumulative survival:

<div class="viz grid-2">
  <div class="card"><strong>Request A</strong><div class="token-row"><span class="tok good">A1 .90</span><span class="arrow">→</span><span class="tok warn">A2 .70</span></div></div>
  <div class="card"><strong>Request B</strong><div class="token-row"><span class="tok good">B1 .80</span><span class="arrow">→</span><span class="tok bad">B2 .30</span></div></div>
</div>

Sort all candidate prefix extensions globally:

<div class="viz token-row">
  <span class="tok good">A1 .90</span><span class="arrow">→</span><span class="tok good">B1 .80</span><span class="arrow">→</span><span class="tok warn">A2 .70</span><span class="arrow">→</span><span class="tok bad">B2 .30</span>
</div>

This automatically respects prerequisites. A2 cannot appear before A1 because cumulative survival is non-increasing along each chain.

The paper’s scheduler uses this monotonic structure to turn a combinatorial allocation problem into “sort candidates, then scan the greedy admission path while checking throughput.”

---

## n23. Offline evaluation setup

Offline evaluation disables the scheduler and forces all drafters to propose a fixed block size.

Why? Because we want to isolate raw draft quality.

<div class="viz pipeline">
  <div class="stage"><strong>Context</strong></div>
  <span class="arrow">→</span>
  <div class="stage"><strong>Draft fixed K</strong></div>
  <span class="arrow">→</span>
  <div class="stage"><strong>Target verifies</strong></div>
  <span class="arrow">→</span>
  <div class="stage"><strong>Measure τ</strong></div>
</div>

This is like a wind-tunnel test. Turn off the pilot so the aircraft design is measured under the same conditions.

The DSpark paper evaluates across math, code, and chat, comparing against Eagle3 and DFlash in the same training framework.

---

## n24. Main results and mechanism

The reported offline result is that DSpark improves accepted length over both a strong autoregressive baseline and a strong parallel baseline.

The mechanism is easier to see position-wise:

<div class="viz">
  <div class="bar-wrap"><span class="bar-label">DFlash p1</span><span class="bar"><span style="width:92%"></span></span></div>
  <div class="bar-wrap"><span class="bar-label">DFlash suffix</span><span class="bar"><span style="width:45%"></span></span></div>
  <div class="bar-wrap"><span class="bar-label">Eagle3 p1</span><span class="bar"><span style="width:72%"></span></span></div>
  <div class="bar-wrap"><span class="bar-label">Eagle3 suffix</span><span class="bar"><span style="width:74%"></span></span></div>
  <div class="bar-wrap"><span class="bar-label">DSpark p1</span><span class="bar"><span style="width:92%"></span></span></div>
  <div class="bar-wrap"><span class="bar-label">DSpark suffix</span><span class="bar"><span style="width:78%"></span></span></div>
</div>

DFlash has strong first-token quality because the parallel backbone can be deeper under the same latency budget. Eagle3 has better suffix coherence because it is autoregressive. DSpark tries to keep the first and recover the second.

The broader principle:

> Performance along a chain is dominated by both the strength of its entry point and the propagation of consistency afterward.

---

## n25. A little autoregression goes a long way

Adding more parallel capacity does not fully solve a wrong factorization.

A 5-layer DFlash can produce stronger independent guesses, but it still mostly answers:

$$
P(x_k\mid \text{context})
$$

DSpark’s sequential head moves closer to:

$$
P(x_k\mid \text{context},x_{<k})
$$

As proposal length grows, that missing dependency matters more.

<div class="note">
  <strong>Analogy.</strong> Adding lanes does not fix a bad intersection. Sometimes one traffic officer gives more value than another lane.
</div>

Correct inductive bias can dominate brute-force capacity.

---

## n26. Production training optimizations

Training the drafter requires target-model supervision. The paper describes two production optimizations.

### Hidden-state communication

Instead of moving full vocabulary logits across workers, move the hidden states before the LM head and compute selected logits locally.

If vocabulary size is about 100,000 and hidden dimension is about 4,096, the raw size ratio is roughly 24×.

<div class="viz pipeline">
  <div class="stage"><strong>Target worker</strong><span class="small">hidden vector h</span></div>
  <span class="arrow">→</span>
  <div class="stage"><strong>Draft worker</strong><span class="small">LM head locally</span></div>
</div>

### Anchor-bounded sequence packing

Do not train at every token position. Sample anchor positions and train draft blocks around those anchors.

For a 1,000-token document and block length 4, training everywhere would create roughly $1000\times4$ draft predictions. Sampling three anchors creates only $3\times4=12$ draft predictions.

<div class="viz">
<pre style="margin:0; line-height:1.55">Full document:  t1 t2 t3 ... t1000
Anchors:                     [100 → 101..104]   [500 → 501..504]   [800 → 801..804]
Packed batch:      [anchor block 1] [anchor block 2] [anchor block 3]</pre>
</div>

Attention indices keep packed blocks logically independent even though they share one physical batch.

Meta-principle:

> Move the smallest sufficient representation, and compute only where supervision exists.

---

## n27. Async scheduler in practice

The ideal scheduler chooses capacity from current confidence. Production serving complicates this because CUDA graph replay and zero-overhead scheduling need the next batch size before the current step finishes.

If the GPU waits for the scheduler, latency is lost:

<div class="viz token-row">
  <span class="tok">GPU step t</span><span class="arrow">→</span><span class="tok bad">wait</span><span class="arrow">→</span><span class="tok">GPU step t+1</span>
</div>

DSpark uses an asynchronous approximation:

<div class="viz token-row">
  <span class="tok good">confidence t−2</span><span class="arrow">→</span><span class="tok good">choose capacity K for t+1</span><span class="arrow">→</span><span class="tok good">current scores fill K slots</span>
</div>

Old information decides **how many seats** the bus has. Current information decides **which passengers** get those seats.

This does two useful things:

1. hides scheduling latency behind GPU work;
2. prevents capacity from expanding after seeing current sampled consequences.

Delay becomes a causal barrier.

---

## n28. Variable-length verification without wasting kernels

The scheduler produces different verification lengths per request:

<div class="viz grid-3">
  <div class="card"><strong>Request A</strong><div class="token-row"><span class="tok good">A1</span><span class="tok good">A2</span><span class="tok good">A3</span></div></div>
  <div class="card"><strong>Request B</strong><div class="token-row"><span class="tok good">B1</span></div></div>
  <div class="card"><strong>Request C</strong><div class="token-row"><span class="tok good">C1</span><span class="tok good">C2</span></div></div>
</div>

Naively padding every request to the same length wastes compute:

<div class="viz token-row">
  <span class="tok good">A1</span><span class="tok good">A2</span><span class="tok good">A3</span>
  <span class="tok good">B1</span><span class="tok ghost">pad</span><span class="tok ghost">pad</span>
  <span class="tok good">C1</span><span class="tok good">C2</span><span class="tok ghost">pad</span>
</div>

The production solution described in the paper is to flatten all verified tokens into one physical batch while carrying marker tensors that preserve logical sequence structure inside sparse attention kernels.

<div class="viz token-row">
  <span class="tok good moving-token">A1</span><span class="tok good">A2</span><span class="tok good">A3</span><span class="tok good">B1</span><span class="tok good">C1</span><span class="tok good">C2</span>
</div>

Logical dependency is tracked by metadata, not by padding the physical execution shape.

This is a common systems trick: decouple the physical layout that hardware likes from the logical structure the algorithm needs.

---

## n29. Live traffic Pareto shift

The paper reports DSpark-5 deployed against the prior MTP-1 production baseline in DeepSeek-V4-Flash and DeepSeek-V4-Pro preview serving.

At matched practical throughput levels, DSpark improves per-user generation speed by:

- **60%–85%** on V4-Flash;
- **57%–78%** on V4-Pro.

At moderate SLA anchors, it improves aggregate throughput by about:

- **51%** at 80 tok/s/user on V4-Flash;
- **52%** at 35 tok/s/user on V4-Pro.

At strict SLA anchors, the baseline approaches a low-concurrency boundary. The large relative gains there are best interpreted as “DSpark keeps the system in a usable operating regime,” not as a stable multiplicative speedup.

<div class="viz">
  <div class="bar-wrap"><span class="bar-label">Flash SLA 80</span><span class="bar"><span style="width:51%"></span></span><span>+51% throughput</span></div>
  <div class="bar-wrap"><span class="bar-label">Pro SLA 35</span><span class="bar"><span style="width:52%"></span></span><span>+52% throughput</span></div>
  <div class="bar-wrap"><span class="bar-label">Flash matched</span><span class="bar"><span style="width:85%"></span></span><span>+60–85% TPS/user</span></div>
  <div class="bar-wrap"><span class="bar-label">Pro matched</span><span class="bar"><span style="width:78%"></span></span><span>+57–78% TPS/user</span></div>
</div>

The key production behavior is adaptive:

- light/moderate load: use spare compute to verify longer prefixes;
- heavy load: restrict verification to protect batch capacity.

Adaptive systems convert spare capacity into latency improvement without committing that extra cost under congestion.

---

## n30. Limitations and future direction

DSpark adapts verification after drafting. It may still generate a full $\gamma$-token block even when the scheduler later keeps only one or two tokens.

<div class="viz token-row">
  <span class="tok">drafted</span><span class="arrow">→</span><span class="tok good">A</span><span class="tok good">B</span><span class="tok ghost">C</span><span class="tok ghost">D</span><span class="tok ghost">E</span>
</div>

So DSpark removes target-side waste first:

<div class="viz token-row">
  <span class="tok">fixed drafting</span><span class="arrow">→</span><span class="tok good">adaptive verification</span>
</div>

The next step is likely:

<div class="viz token-row">
  <span class="tok good">adaptive drafting</span><span class="arrow">→</span><span class="tok good">adaptive verification</span>
</div>

Hard requests could early-exit from the draft backbone or choose a smaller draft length before paying for all positions.

This follows the broader pattern:

> Optimization removes waste one stage at a time. Move the decision closer to where the wasted work begins.

---

## Final compression

Speculative decoding is exact because accepted draft mass plus residual correction mass equals the target distribution.

Accepted length is a survival process because each later token exists only if every earlier token survived.

Parallel drafting creates speed by shortening the critical path, but it loses branch identity and causes suffix decay.

DSpark keeps the parallel backbone, adds a tiny sequential head for coherence, and uses calibrated confidence to spend verification where prefix survival justifies the hardware cost.

The transferable principle is:

<div class="goodbox">
  <strong>Speculate aggressively inside the system; enforce the invariant at commitment; allocate scarce verification compute by expected useful work, not by candidate availability.</strong>
</div>

</div>
