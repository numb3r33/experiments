---
title: "DSpark: Confidence-Scheduled Speculative Decoding"
description: "An illustrated intuition ladder for DSpark: why speculative decoding is exact, why accepted prefixes behave like survival processes, and how DSpark combines semi-autoregressive drafting with load-aware verification."
layout: post
toc: true
badges: true
comments: true
sticky_rank: 1
author: Abhishek Sharma
image: images/dspark_confidence_scheduled_speculative_decoding.svg
categories: [llm, inference, speculative-decoding, systems, deepseek]
---

> **TL;DR.** Speculative decoding is not "let a small model guess and hope." It is an exact coupling protocol: cheap computation proposes a future, the target model certifies the reusable probability mass, and residual sampling repairs the first failure. DSpark improves this protocol in two places: a **semi-autoregressive drafter** repairs suffix coherence, and a **confidence scheduler** spends target verification only where the expected prefix survival justifies the hardware cost.

<style>
.dspark-post{
  --ink:#22303f; --muted:#65748a; --faint:#8b97a8;
  --line:#e4eaf1; --bg-soft:#f7f9fc;
  --sky:#dcebfb;     --sky-edge:#a9cdf2;   --sky-ink:#1d5fa6;
  --lav:#e9e4fb;     --lav-edge:#c4b9f3;   --lav-ink:#6442cf;
  --mint:#d8f0de;    --mint-edge:#a8e7ad;  --mint-ink:#1f7a45;
  --amber:#fdeecb;   --amber-edge:#f1d089; --amber-ink:#a9760f;
  --pink:#fbe0ec;    --pink-edge:#f3b3d0;  --pink-ink:#b83673;
  --coral:#ffe1e1;   --coral-edge:#f4b1b1; --coral-ink:#c0392b;
  --ghost:#eef1f5;   --ghost-edge:#dde3ea; --ghost-ink:#9aa6b3;
  color:var(--ink);
  font-family:-apple-system,BlinkMacSystemFont,"Helvetica Neue",Helvetica,Arial,sans-serif;
  font-size:1.06rem; line-height:1.75; -webkit-font-smoothing:antialiased;
}
.dspark-post p, .dspark-post ul, .dspark-post ol, .dspark-post table,
.dspark-post blockquote, .dspark-post h2, .dspark-post h3, .dspark-post h4{
  max-width:680px; margin-left:auto; margin-right:auto;
}
.dspark-post h2{ font-size:1.6rem; font-weight:800; letter-spacing:-.01em;
  margin:3.4rem auto 1.1rem; padding-top:2.2rem; border-top:1px solid var(--line); }
.dspark-post h2:first-of-type{ border-top:0; padding-top:.4rem; margin-top:1.2rem; }
.dspark-post h3{ font-size:1.22rem; font-weight:800; margin:2.1rem auto .6rem; }
.dspark-post h4{ font-size:1.02rem; font-weight:800; margin:1.3rem auto .3rem; }
.dspark-post p{ margin:1.05rem auto; }
.dspark-post strong{ font-weight:700; }
.dspark-post blockquote{ border-left:4px solid var(--sky-edge); background:var(--bg-soft);
  padding:.6rem 1.1rem; border-radius:0 12px 12px 0; color:#3a4a5c; font-style:normal; }
.dspark-post .lede{ font-size:1.16rem; color:var(--muted); }

/* Figures: narrow text, wide illustration break-out */
.dspark-post figure.fig{ width:min(94vw,1000px); margin:2.4rem auto; position:relative;
  left:50%; transform:translateX(-50%); text-align:center; }
.dspark-post figure.fig svg, .dspark-post figure.fig img{ width:100%; height:auto; display:block; }
.dspark-post .panel{ width:min(94vw,900px); margin:2rem auto; position:relative; left:50%;
  transform:translateX(-50%); background:var(--bg-soft); border:1px solid var(--line);
  border-radius:20px; padding:1.5rem 1.7rem; }
.dspark-post figure.fig figcaption, .dspark-post .cap{ max-width:680px; margin:.85rem auto 0;
  color:var(--muted); font-size:.93rem; line-height:1.55; text-align:center; }

/* Callouts */
.dspark-post .callout{ max-width:680px; margin:1.8rem auto; border-radius:14px;
  padding:.9rem 1.2rem; border-left:5px solid var(--sky-edge); background:var(--sky); color:#13456f; }
.dspark-post .callout.warn{ border-left-color:var(--amber-edge); background:var(--amber); color:#7a5408; }
.dspark-post .callout.good{ border-left-color:var(--mint-edge); background:var(--mint); color:#16613a; }
.dspark-post .callout.key{ border-left-color:var(--pink-edge); background:var(--pink); color:#8e2a59; }
.dspark-post .callout p{ margin:.25rem 0; max-width:none; }

/* Token chips */
.dspark-post .tokens{ display:flex; gap:.5rem; flex-wrap:wrap; justify-content:center; align-items:center; }
.dspark-post .chip{ min-width:2.4rem; height:2.4rem; padding:0 .55rem; border-radius:11px;
  display:inline-flex; align-items:center; justify-content:center; flex-direction:column;
  font-weight:800; font-size:1rem; background:var(--sky); color:var(--sky-ink); border:1.5px solid var(--sky-edge); }
.dspark-post .chip small{ font-weight:700; font-size:.62rem; opacity:.85; line-height:1; }
.dspark-post .chip.good{ background:var(--mint); color:var(--mint-ink); border-color:var(--mint-edge); }
.dspark-post .chip.bad{ background:var(--coral); color:var(--coral-ink); border-color:var(--coral-edge); }
.dspark-post .chip.warn{ background:var(--amber); color:var(--amber-ink); border-color:var(--amber-edge); }
.dspark-post .chip.lav{ background:var(--lav); color:var(--lav-ink); border-color:var(--lav-edge); }
.dspark-post .chip.pink{ background:var(--pink); color:var(--pink-ink); border-color:var(--pink-edge); }
.dspark-post .chip.ghost{ background:var(--ghost); color:var(--ghost-ink); border-color:var(--ghost-edge); text-decoration:line-through; }
.dspark-post .chip.lg{ min-width:auto; padding:0 .8rem; }
.dspark-post .arrow{ color:var(--faint); font-weight:800; font-size:1.15rem; }

/* Flow nodes */
.dspark-post .flow{ display:flex; flex-wrap:wrap; gap:.55rem; align-items:center; justify-content:center; }
.dspark-post .node{ border-radius:14px; border:1.5px solid var(--line); background:#fff;
  padding:.65rem .85rem; min-width:124px; text-align:center; }
.dspark-post .node .t{ font-weight:800; display:block; font-size:.96rem; }
.dspark-post .node .s{ font-size:.78rem; color:var(--muted); display:block; margin-top:.1rem; }
.dspark-post .node.sky{ background:var(--sky); border-color:var(--sky-edge); color:var(--sky-ink); }
.dspark-post .node.lav{ background:var(--lav); border-color:var(--lav-edge); color:var(--lav-ink); }
.dspark-post .node.amber{ background:var(--amber); border-color:var(--amber-edge); color:var(--amber-ink); }
.dspark-post .node.pink{ background:var(--pink); border-color:var(--pink-edge); color:var(--pink-ink); }
.dspark-post .node.mint{ background:var(--mint); border-color:var(--mint-edge); color:var(--mint-ink); }
.dspark-post .node .s{ color:inherit; opacity:.78; }

/* Card grids */
.dspark-post .cols2{ display:grid; grid-template-columns:1fr 1fr; gap:1rem; max-width:760px; margin:1.5rem auto; }
.dspark-post .cols3{ display:grid; grid-template-columns:1fr 1fr 1fr; gap:.8rem; max-width:820px; margin:1.5rem auto; }
@media(max-width:640px){ .dspark-post .cols2, .dspark-post .cols3{ grid-template-columns:1fr; } }
.dspark-post .card{ border:1.5px solid var(--line); border-radius:16px; padding:1rem 1.1rem; background:#fff; }
.dspark-post .card h4{ margin:.1rem 0 .45rem; }
.dspark-post .card.sky{ background:var(--sky); border-color:var(--sky-edge); }
.dspark-post .card.pink{ background:var(--pink); border-color:var(--pink-edge); }
.dspark-post .card.mint{ background:var(--mint); border-color:var(--mint-edge); }
.dspark-post .card .small{ font-size:.9rem; color:var(--muted); }
.dspark-post .tag{ display:inline-block; border-radius:999px; padding:.13rem .6rem; font-size:.72rem;
  font-weight:800; text-transform:uppercase; letter-spacing:.03em;
  background:var(--sky); color:var(--sky-ink); border:1px solid var(--sky-edge); }

/* Tables */
.dspark-post table{ border-collapse:collapse; width:100%; font-size:.95rem; margin:1.4rem auto; }
.dspark-post th, .dspark-post td{ border-bottom:1px solid var(--line); padding:.55rem .7rem; text-align:left; vertical-align:top; }
.dspark-post thead th{ background:var(--bg-soft); font-weight:800; border-bottom:2px solid var(--line); }

/* Mini bars */
.dspark-post .bars{ max-width:600px; margin:1.3rem auto; }
.dspark-post .bar-row{ display:flex; align-items:center; gap:.7rem; margin:.5rem 0; }
.dspark-post .bar-row .lbl{ width:9.5rem; font-size:.88rem; font-weight:700; text-align:right; color:var(--muted); }
.dspark-post .bar-row .val{ font-size:.82rem; color:var(--muted); width:5.5rem; }
.dspark-post .bar{ flex:1; height:1rem; border-radius:999px; background:#eef2f7; overflow:hidden; }
.dspark-post .bar > i{ display:block; height:100%; border-radius:999px; background:var(--sky-edge); }
.dspark-post .bar > i.mint{ background:var(--mint-edge); }
.dspark-post .bar > i.pink{ background:var(--pink-edge); }
.dspark-post .bar > i.lav{ background:var(--lav-edge); }
.dspark-post .bar > i.amber{ background:var(--amber-edge); }

/* Code / pre */
.dspark-post pre{ background:var(--bg-soft); border:1px solid var(--line); border-radius:14px;
  padding:1rem 1.2rem; overflow-x:auto; font-size:.86rem; line-height:1.55; color:#2c3a4a;
  max-width:760px; margin:1.6rem auto; }
</style>

<div class="dspark-post" markdown="1">

<figure class="fig">
<svg viewBox="0 0 960 340" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-hero" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="340" fill="#ffffff"/>
  <rect x="22" y="116" width="112" height="104" rx="10" fill="#ffffff" stroke="#cdd9e8" stroke-width="2"/>
  <line x1="34" y1="140" x2="122" y2="140" stroke="#dbe6f2" stroke-width="2"/>
  <line x1="34" y1="160" x2="122" y2="160" stroke="#dbe6f2" stroke-width="2"/>
  <line x1="34" y1="180" x2="122" y2="180" stroke="#dbe6f2" stroke-width="2"/>
  <text x="40" y="136" font-size="11" fill="#5a6b80">The capital</text>
  <text x="40" y="156" font-size="11" fill="#5a6b80">of France is…</text>
  <text x="78" y="238" font-size="12.5" fill="#65748a" text-anchor="middle" font-weight="700">context</text>
  <line x1="138" y1="168" x2="172" y2="168" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-hero)"/>
  <rect x="178" y="70" width="298" height="208" rx="18" fill="#f3f8ff" stroke="#cfe0f4" stroke-width="2"/>
  <text x="327" y="98" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">DSpark drafter</text>
  <rect x="196" y="112" width="262" height="46" rx="12" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="327" y="133" font-size="13.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">Parallel backbone</text>
  <text x="327" y="150" font-size="11" fill="#3f6e9e" text-anchor="middle">drafts γ positions at once</text>
  <rect x="196" y="166" width="262" height="46" rx="12" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="327" y="187" font-size="13.5" font-weight="800" fill="#6442cf" text-anchor="middle">Sequential head</text>
  <text x="327" y="204" font-size="11" fill="#6a59a8" text-anchor="middle">repairs token-to-token coherence</text>
  <g font-size="13" font-weight="800" text-anchor="middle">
    <rect x="206" y="228" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/><text x="223" y="250" fill="#1d5fa6">E</text>
    <rect x="248" y="228" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/><text x="265" y="250" fill="#1d5fa6">F</text>
    <rect x="290" y="228" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/><text x="307" y="250" fill="#1d5fa6">G</text>
    <rect x="332" y="228" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/><text x="349" y="250" fill="#1d5fa6">H</text>
    <text x="410" y="250" font-size="11" font-weight="700" fill="#65748a">draft</text>
  </g>
  <line x1="476" y1="174" x2="556" y2="174" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-hero)"/>
  <rect x="492" y="150" width="48" height="48" rx="12" fill="#fdeecb" stroke="#f1d089" stroke-width="1.5"/>
  <text x="516" y="181" font-size="19" fill="#a9760f" text-anchor="middle">⛒</text>
  <text x="516" y="222" font-size="11.5" font-weight="700" fill="#a9760f" text-anchor="middle">confidence</text>
  <text x="516" y="237" font-size="11.5" font-weight="700" fill="#a9760f" text-anchor="middle">scheduler</text>
  <rect x="520" y="114" width="26" height="26" rx="7" fill="#eef1f5" stroke="#dde3ea" stroke-width="1.5"/>
  <text x="533" y="132" font-size="11" fill="#9aa6b3" text-anchor="middle">✕</text>
  <rect x="576" y="104" width="206" height="132" rx="18" fill="#fbe0ec" stroke="#f3b3d0" stroke-width="2"/>
  <text x="679" y="150" font-size="16" font-weight="800" fill="#b83673" text-anchor="middle">Target verifier</text>
  <text x="679" y="174" font-size="12" fill="#a8557e" text-anchor="middle">certifies reusable</text>
  <text x="679" y="190" font-size="12" fill="#a8557e" text-anchor="middle">probability mass</text>
  <line x1="782" y1="170" x2="818" y2="170" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-hero)"/>
  <g font-size="12.5" font-weight="800" text-anchor="middle">
    <rect x="824" y="120" width="112" height="32" rx="9" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/><text x="880" y="141" fill="#1f7a45">Paris ✓</text>
    <rect x="824" y="158" width="112" height="32" rx="9" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/><text x="880" y="179" fill="#1f7a45">is ✓</text>
    <rect x="824" y="196" width="112" height="32" rx="9" fill="#ffe1e1" stroke="#f4b1b1" stroke-width="1.5"/><text x="880" y="217" fill="#c0392b">repair</text>
    <text x="880" y="246" font-size="11" font-weight="700" fill="#65748a">accepted prefix</text>
  </g>
</svg>
<figcaption>DSpark keeps the fast parallel path, adds a tiny sequential head for coherence, and spends scarce target verification only where the confidence scheduler expects it to pay off.</figcaption>
</figure>

## The map of the paper

DSpark starts from two observations about speculative decoding.

First, **parallel drafters are fast** because they propose many future tokens in one pass. But they often lose coherence later in the block because position 5 did not condition on the actual sampled positions 1–4.

Second, **verification is not free**. Under heavy serving load, every low-confidence draft token sent to the target model consumes batch capacity that could have served another user.

So DSpark adds two narrow fixes:

<div class="panel">
  <div class="flow">
    <div class="node sky"><span class="t">Parallel backbone</span><span class="s">draft many positions once</span></div>
    <span class="arrow">→</span>
    <div class="node lav"><span class="t">Sequential head</span><span class="s">repair token dependencies</span></div>
    <span class="arrow">→</span>
    <div class="node amber"><span class="t">Confidence scheduler</span><span class="s">verify only useful prefix</span></div>
    <span class="arrow">→</span>
    <div class="node pink"><span class="t">Target verifier</span><span class="s">preserve target law</span></div>
  </div>
</div>

The rest of the post builds the intuition ladder in the following order: serving objective, verification waste, exact speculative sampling, survival length, latency equation, drafter architectures, DSpark's design, training, scheduling, and production results.

## Serving objective: latency versus throughput

A production serving system has two goals:

| Metric | Question | Restaurant analogy |
|---|---|---|
| **Per-user tok/s** | How fast does one user see tokens? | How quickly one table gets food |
| **Tokens/s/GPU** | How many total tokens does the GPU produce? | How many plates the kitchen outputs |

These are not the same. A system can make one VIP user very fast while leaving the GPU under-utilized. Or it can fully utilize the GPU while every individual user waits too long.

<figure class="fig">
  <svg viewBox="0 0 900 340" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
    <rect x="0" y="0" width="900" height="340" fill="#ffffff"/>
    <line x1="90" y1="280" x2="840" y2="280" stroke="#c2ccd8" stroke-width="2"/>
    <line x1="90" y1="280" x2="90" y2="50" stroke="#c2ccd8" stroke-width="2"/>
    <text x="465" y="316" font-size="14" fill="#65748a" text-anchor="middle">aggregate tokens/s/GPU</text>
    <text x="34" y="165" font-size="14" fill="#65748a" text-anchor="middle" transform="rotate(-90 34,165)">per-user tok/s</text>
    <path d="M120 252 C240 214, 340 196, 430 184 C540 168, 640 138, 720 96" fill="none" stroke="#cdd9e6" stroke-width="7" stroke-linecap="round"/>
    <path d="M150 224 C290 172, 410 148, 520 122 C640 96, 720 72, 800 50" fill="none" stroke="#a9cdf2" stroke-width="7" stroke-linecap="round"/>
    <text x="262" y="232" font-size="15" font-weight="800" fill="#9aa6b3">old frontier</text>
    <text x="612" y="84" font-size="15" font-weight="800" fill="#1d5fa6">DSpark frontier</text>
  </svg>
  <figcaption>A serving improvement is strongest when the whole frontier moves outward: same throughput with lower latency, or same latency with higher throughput.</figcaption>
</figure>

For a rough model, if a GPU serves `N` concurrent users and produces aggregate rate `R`, then one user sees roughly:

$$
r \approx \frac{\eta(N) R_{\max}}{N}
$$

where $\eta(N)$ captures batching efficiency, scheduler overhead, memory pressure, and verification waste.

DSpark's goal is not merely "better draft tokens." It tries to improve this serving efficiency curve.

## Verification waste under load

Speculative decoding often sounds like:

> "Draft more tokens, then verify more tokens."

The hidden assumption is that verification is free. It is not.

If five users each propose four draft tokens, the target verifier sees twenty candidate positions instead of five ordinary decode positions.

<div class="cols2">
  <div class="card">
    <h4>Ordinary decode batch</h4>
    <div class="tokens" style="justify-content:flex-start">
      <span class="chip">T</span><span class="chip">T</span><span class="chip">T</span><span class="chip">T</span><span class="chip">T</span>
    </div>
    <p class="small" style="margin:.6rem 0 0">Five users, five target positions.</p>
  </div>
  <div class="card">
    <h4>Speculative verification batch</h4>
    <div class="tokens" style="justify-content:flex-start">
      <span class="chip">a</span><span class="chip">b</span><span class="chip warn">c</span><span class="chip bad">d</span>
      <span class="chip">e</span><span class="chip">f</span><span class="chip warn">g</span><span class="chip bad">h</span>
    </div>
    <p class="small" style="margin:.6rem 0 0">Long suffixes may be paid for even when they almost never survive.</p>
  </div>
</div>

Under light load, the GPU may have spare capacity. Wasting a few verification slots barely matters.

Under heavy load, the same extra token has opportunity cost: it delays another request, increases queueing, or lowers total tokens/s/GPU.

<div class="callout warn">
  <p><strong>Key systems shift.</strong> The question changes from "is this token individually plausible?" to "is verifying this token the best use of scarce target-model capacity right now?"</p>
</div>

That is why DSpark needs a scheduler, not only a better drafter.

## Speculative decoding invariant: propose cheaply, certify expensively

Ordinary autoregressive decoding performs one target-model step per token:

<div class="panel">
  <div class="tokens">
    <span class="chip lg">context</span><span class="arrow">Target →</span><span class="chip good">x₁</span><span class="arrow">Target →</span><span class="chip good">x₂</span><span class="arrow">Target →</span><span class="chip good">x₃</span>
  </div>
</div>

Speculative decoding asks a cheap draft process to propose a possible future:

<div class="panel">
  <div class="tokens" style="margin-bottom:.7rem">
    <span class="chip lg">c</span><span class="arrow">draft →</span><span class="chip">E</span><span class="chip">F</span><span class="chip">G</span><span class="chip">H</span>
  </div>
  <div class="tokens">
    <span class="chip ghost" style="visibility:hidden"> </span><span class="arrow">target scores in one pass →</span><span class="chip good">p₁</span><span class="chip good">p₂</span><span class="chip good">p₃</span><span class="chip good">p₄</span>
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

<figure class="fig">
<svg viewBox="0 0 960 372" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-coupling" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="372" fill="#ffffff"/>

  <text x="480" y="36" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">Accepted mass is the overlap of draft and target.</text>

  <rect x="286" y="54" width="14" height="14" rx="4" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="306" y="65" font-size="12" fill="#65748a">draft q</text>
  <rect x="372" y="54" width="14" height="14" rx="4" fill="#fbe0ec" stroke="#f3b3d0" stroke-width="1.5"/>
  <text x="392" y="65" font-size="12" fill="#65748a">target p</text>
  <rect x="464" y="54" width="14" height="14" rx="4" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="484" y="65" font-size="12" fill="#65748a">accepted = min(p, q)</text>

  <line x1="120" y1="168" x2="880" y2="168" stroke="#e4eaf1" stroke-width="1"/>
  <line x1="120" y1="234" x2="880" y2="234" stroke="#e4eaf1" stroke-width="1"/>
  <text x="112" y="172" font-size="11" fill="#8b97a8" text-anchor="end">.40</text>
  <text x="112" y="238" font-size="11" fill="#8b97a8" text-anchor="end">.20</text>

  <rect x="159" y="102" width="30" height="198" rx="5" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <rect x="195" y="168" width="30" height="132" rx="5" fill="#fbe0ec" stroke="#f3b3d0" stroke-width="1.5"/>
  <rect x="231" y="168" width="30" height="132" rx="5" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="174" y="94" font-size="11" fill="#1d5fa6" text-anchor="middle" font-weight="700">.60</text>
  <text x="210" y="160" font-size="11" fill="#b83673" text-anchor="middle" font-weight="700">.40</text>
  <text x="246" y="160" font-size="11" fill="#1f7a45" text-anchor="middle" font-weight="700">.40</text>

  <rect x="359" y="201" width="30" height="99" rx="5" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <rect x="395" y="168" width="30" height="132" rx="5" fill="#fbe0ec" stroke="#f3b3d0" stroke-width="1.5"/>
  <rect x="431" y="201" width="30" height="99" rx="5" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="374" y="193" font-size="11" fill="#1d5fa6" text-anchor="middle" font-weight="700">.30</text>
  <text x="410" y="160" font-size="11" fill="#b83673" text-anchor="middle" font-weight="700">.40</text>
  <text x="446" y="193" font-size="11" fill="#1f7a45" text-anchor="middle" font-weight="700">.30</text>

  <rect x="559" y="267" width="30" height="33" rx="5" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <rect x="595" y="234" width="30" height="66" rx="5" fill="#fbe0ec" stroke="#f3b3d0" stroke-width="1.5"/>
  <rect x="631" y="267" width="30" height="33" rx="5" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="574" y="259" font-size="11" fill="#1d5fa6" text-anchor="middle" font-weight="700">.10</text>
  <text x="610" y="226" font-size="11" fill="#b83673" text-anchor="middle" font-weight="700">.20</text>
  <text x="646" y="259" font-size="11" fill="#1f7a45" text-anchor="middle" font-weight="700">.10</text>

  <rect x="759" y="296" width="30" height="4" rx="2" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <rect x="795" y="296" width="30" height="4" rx="2" fill="#fbe0ec" stroke="#f3b3d0" stroke-width="1.5"/>
  <rect x="831" y="296" width="30" height="4" rx="2" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="810" y="288" font-size="11" fill="#8b97a8" text-anchor="middle" font-weight="700">.00</text>

  <line x1="120" y1="300" x2="880" y2="300" stroke="#e4eaf1" stroke-width="2"/>
  <text x="210" y="320" font-size="13" fill="#22303f" text-anchor="middle" font-weight="800">A</text>
  <text x="410" y="320" font-size="13" fill="#22303f" text-anchor="middle" font-weight="800">B</text>
  <text x="610" y="320" font-size="13" fill="#22303f" text-anchor="middle" font-weight="800">C</text>
  <text x="810" y="320" font-size="13" fill="#22303f" text-anchor="middle" font-weight="800">D</text>
  <text x="104" y="320" font-size="11" fill="#8b97a8" text-anchor="end">token</text>

  <rect x="120" y="343" width="14" height="14" rx="4" fill="#ffe1e1" stroke="#f4b1b1" stroke-width="1.5"/>
  <text x="142" y="354" font-size="12" fill="#65748a">Unmatched target mass (p &#8722; q)+ is supplied by a residual correction (coral).</text>
</svg>
<figcaption>Accepted mass is the overlap of the draft and target distributions, <span style="white-space:nowrap">$\min(p(x),q(x))$</span>. The unmatched target mass is supplied by a residual correction.</figcaption>
</figure>

The unmatched target mass must be supplied by a residual correction distribution:

$$
r(x) \propto (p(x)-q(x))_+
$$

This is why "fall back to the target distribution" is imprecise. If rejection sampled directly from $p$, the algorithm would double-count target mass already covered by accepted draft proposals.

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

So a draft model is good when its probability mass overlaps the target's mass, not merely when it shares the same argmax.

### Why only the longest accepted prefix survives

Suppose the draft proposes:

<div class="panel">
  <div class="tokens">
    <span class="chip good">E ✓</span><span class="arrow">→</span><span class="chip good">F ✓</span><span class="arrow">→</span><span class="chip bad">G ✗</span><span class="arrow">→</span><span class="chip ghost">H</span>
  </div>
</div>

After $G$ is rejected, the correct next token is some $G^\star$. The real context becomes $c,E,F,G^\star$, not $c,E,F,G$.

The old `H` was generated on the wrong branch:

<pre>context ── E ── F ── G  ── H    old branch
                    \
                     G* ── ?    actual branch</pre>

The causal invariant is:

> Reuse is valid only inside the causal region whose assumptions remain unchanged.

For a linear autoregressive chain, that valid region is exactly the longest accepted prefix.

## Prefix survival and accepted length

Accepted prefix length is a survival process. Imagine stepping stones — to stand on a later stone, every earlier stone must have held:

<figure class="fig">
<svg viewBox="0 0 960 300" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-stones" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="300" fill="#ffffff"/>
  <rect x="0" y="246" width="960" height="40" fill="#eaf4fb"/>
  <text x="480" y="52" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">To stand on a later stone, every earlier stone must have held</text>
  <g text-anchor="middle">
    <rect x="70" y="150" width="120" height="84" rx="16" fill="#d8f0de" stroke="#a8e7ad" stroke-width="2"/>
    <text x="130" y="182" font-size="15" font-weight="800" fill="#1f7a45">token 1</text>
    <text x="130" y="206" font-size="13" fill="#2f8a56">c₁ = .90</text>
    <rect x="248" y="150" width="120" height="84" rx="16" fill="#d8f0de" stroke="#a8e7ad" stroke-width="2"/>
    <text x="308" y="182" font-size="15" font-weight="800" fill="#1f7a45">token 2</text>
    <text x="308" y="206" font-size="13" fill="#2f8a56">c₂ = .80</text>
    <rect x="426" y="150" width="120" height="84" rx="16" fill="#fdeecb" stroke="#f1d089" stroke-width="2"/>
    <text x="486" y="182" font-size="15" font-weight="800" fill="#a9760f">token 3</text>
    <text x="486" y="206" font-size="13" fill="#b07a16">c₃ = .70</text>
    <rect x="604" y="150" width="120" height="84" rx="16" fill="#ffe1e1" stroke="#f4b1b1" stroke-width="2"/>
    <text x="664" y="182" font-size="15" font-weight="800" fill="#c0392b">token 4</text>
    <text x="664" y="206" font-size="13" fill="#c0392b">c₄ = .20</text>
    <text x="130" y="126" font-size="12.5" fill="#65748a">reach .90</text>
    <text x="308" y="126" font-size="12.5" fill="#65748a">reach .72</text>
    <text x="486" y="126" font-size="12.5" fill="#65748a">reach .50</text>
    <text x="664" y="126" font-size="12.5" fill="#65748a">reach .10</text>
  </g>
  <g stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-stones)">
    <line x1="190" y1="192" x2="248" y2="192"/>
    <line x1="368" y1="192" x2="426" y2="192"/>
    <line x1="546" y1="192" x2="604" y2="192"/>
  </g>
</svg>
<figcaption>Accepted prefix length is a survival process: the probability of reaching token <em>k</em> is the product of all earlier conditional acceptances.</figcaption>
</figure>

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

<div class="callout">
  <p><strong>Why early tokens matter so much.</strong> Improving $c_1$ helps every downstream prefix. Improving $c_4$ helps only the length-4 prefix.</p>
</div>

This explains why DSpark pays special attention to early-token quality and why the training objective weights early positions more heavily.

## Latency equation: three levers

Speculative decoding happens in rounds:

<div class="panel">
  <div class="flow">
    <div class="node sky"><span class="t">draft block</span><span class="s">cost $T_{draft}$</span></div>
    <span class="arrow">→</span>
    <div class="node pink"><span class="t">verify prefix</span><span class="s">cost $T_{verify}$</span></div>
    <span class="arrow">→</span>
    <div class="node mint"><span class="t">emit tokens</span><span class="s">accepted length τ</span></div>
  </div>
</div>

So latency per token is an efficiency ratio:

$$
L=\frac{T_{draft}+T_{verify}}{\tau}
$$

There are only three levers:

<figure class="fig">
<svg viewBox="0 0 960 344" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-levers" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="344" fill="#ffffff"/>

  <text x="480" y="40" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">Three levers, one ratio.</text>

  <text x="305" y="100" font-size="34" font-weight="800" fill="#22303f" text-anchor="middle">L</text>
  <text x="349" y="100" font-size="28" fill="#65748a" text-anchor="middle">=</text>

  <text x="400" y="87" font-size="22" fill="#65748a" text-anchor="middle">(</text>
  <text x="470" y="87" font-size="22" font-weight="800" fill="#1d5fa6" text-anchor="middle">T_draft</text>
  <text x="527" y="87" font-size="22" fill="#65748a" text-anchor="middle">+</text>
  <text x="595" y="87" font-size="22" font-weight="800" fill="#a9760f" text-anchor="middle">T_verify</text>
  <text x="658" y="87" font-size="22" fill="#65748a" text-anchor="middle">)</text>

  <line x1="393" y1="99" x2="665" y2="99" stroke="#22303f" stroke-width="2.5"/>

  <text x="529" y="128" font-size="28" font-weight="800" fill="#1f7a45" text-anchor="middle">&#964;</text>

  <rect x="64" y="170" width="262" height="146" rx="16" fill="#dcebfb" stroke="#a9cdf2" stroke-width="2"/>
  <text x="195" y="206" font-size="15" font-weight="800" fill="#1d5fa6" text-anchor="middle">lower T_draft</text>
  <text x="195" y="228" font-size="12" fill="#65748a" text-anchor="middle">parallel backbone</text>
  <line x1="135" y1="276" x2="255" y2="276" stroke="#a9cdf2" stroke-width="3"/>
  <circle cx="168" cy="276" r="11" fill="#dcebfb" stroke="#1d5fa6" stroke-width="2.5"/>

  <rect x="349" y="170" width="262" height="146" rx="16" fill="#fdeecb" stroke="#f1d089" stroke-width="2"/>
  <text x="480" y="206" font-size="15" font-weight="800" fill="#a9760f" text-anchor="middle">lower effective T_verify</text>
  <text x="480" y="228" font-size="12" fill="#65748a" text-anchor="middle">confidence scheduler</text>
  <line x1="420" y1="276" x2="540" y2="276" stroke="#f1d089" stroke-width="3"/>
  <circle cx="495" cy="276" r="11" fill="#fdeecb" stroke="#a9760f" stroke-width="2.5"/>

  <rect x="634" y="170" width="262" height="146" rx="16" fill="#d8f0de" stroke="#a8e7ad" stroke-width="2"/>
  <text x="765" y="206" font-size="15" font-weight="800" fill="#1f7a45" text-anchor="middle">raise &#964; (accepted length)</text>
  <text x="765" y="228" font-size="12" fill="#65748a" text-anchor="middle">sequential head + training</text>
  <line x1="705" y1="276" x2="825" y2="276" stroke="#a8e7ad" stroke-width="3"/>
  <circle cx="792" cy="276" r="11" fill="#d8f0de" stroke="#1f7a45" stroke-width="2.5"/>
</svg>
<figcaption>Optimizing one variable can hurt another. A very accurate draft that costs almost as much as the target may lose; a tiny draft that is fast but rarely accepted also loses. The real objective is the whole ratio.</figcaption>
</figure>

| Lever | How to improve it | DSpark component |
|---|---|---|
| Lower $T_{draft}$ | Avoid heavy autoregressive draft loops | parallel backbone |
| Lower effective $T_{verify}$ | Avoid low-value verification slots | confidence scheduler |
| Increase $\tau$ | Improve accepted prefix length | sequential head + better training |

The important lesson is that optimizing one variable can hurt another. A very accurate draft that costs almost as much as the target may lose. A tiny draft that is fast but rarely accepted also loses.

The real objective is the whole ratio.

## Autoregressive vs parallel drafters

An autoregressive drafter proposes each token conditioned on the previous draft token. This gives strong coherence, because position 4 knows what positions 1–3 actually became — but the critical path has length $\gamma$. A parallel drafter does the opposite: it lowers $T_{draft}$ by producing all positions at once, but may reduce $\tau$ because later positions do not know the realized earlier tokens.

<figure class="fig">
<svg viewBox="0 0 960 320" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-ar-vs-parallel" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="320" fill="#ffffff"/>
  <text x="480" y="44" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">Coherence vs latency: the drafter trade-off.</text>
  <text x="197" y="104" font-size="15" font-weight="800" fill="#6442cf" text-anchor="middle">Autoregressive drafter</text>
  <rect x="44" y="150" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="61" y="172" font-size="14" font-weight="800" fill="#6442cf" text-anchor="middle">E</text>
  <line x1="82" y1="167" x2="106" y2="167" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <rect x="112" y="150" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="129" y="172" font-size="14" font-weight="800" fill="#6442cf" text-anchor="middle">F</text>
  <line x1="150" y1="167" x2="174" y2="167" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <rect x="180" y="150" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="197" y="172" font-size="14" font-weight="800" fill="#6442cf" text-anchor="middle">G</text>
  <line x1="218" y1="167" x2="242" y2="167" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <rect x="248" y="150" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="265" y="172" font-size="14" font-weight="800" fill="#6442cf" text-anchor="middle">H</text>
  <line x1="286" y1="167" x2="310" y2="167" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <rect x="316" y="150" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="333" y="172" font-size="14" font-weight="800" fill="#6442cf" text-anchor="middle">I</text>
  <text x="197" y="220" font-size="12" fill="#65748a" text-anchor="middle">critical path = &#947; steps</text>
  <text x="197" y="238" font-size="12" fill="#65748a" text-anchor="middle">strong coherence</text>
  <text x="714" y="104" font-size="15" font-weight="800" fill="#1d5fa6" text-anchor="middle">Parallel drafter</text>
  <rect x="556" y="147" width="92" height="40" rx="12" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="602" y="172" font-size="12.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">context</text>
  <line x1="652" y1="167" x2="772" y2="124" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <line x1="652" y1="167" x2="772" y2="160" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <line x1="652" y1="167" x2="772" y2="196" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <line x1="652" y1="167" x2="772" y2="232" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-ar-vs-parallel)"/>
  <rect x="778" y="107" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="795" y="129" font-size="14" font-weight="800" fill="#1d5fa6" text-anchor="middle">E</text>
  <rect x="778" y="143" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="795" y="165" font-size="14" font-weight="800" fill="#1d5fa6" text-anchor="middle">F</text>
  <rect x="778" y="179" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="795" y="201" font-size="14" font-weight="800" fill="#1d5fa6" text-anchor="middle">G</text>
  <rect x="778" y="215" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="795" y="237" font-size="14" font-weight="800" fill="#1d5fa6" text-anchor="middle">H</text>
  <text x="714" y="284" font-size="12" fill="#65748a" text-anchor="middle">one pass</text>
  <text x="714" y="302" font-size="12" fill="#65748a" text-anchor="middle">weak intra-block dependency</text>
</svg>
<figcaption>If one draft step takes 1 ms, seven autoregressive draft tokens cost roughly 7 ms. Parallel drafting collapses that critical path to a single pass — at the cost of intra-block dependency.</figcaption>
</figure>

The design question becomes:

> Can we preserve enough autoregressive dependency modeling without paying full autoregressive latency?

DSpark's answer is: put the heavy computation in a parallel backbone, then add a tiny sequential correction head.

### Parallel drafters and DFlash

DFlash takes the parallel idea further. Since the sequential loop has been removed, it can spend the saved latency on a deeper backbone. It also injects rich target-model context through KV features, so the draft model does not have to rediscover everything the target already understood.

The trade is:

| Architecture | Strength | Weakness |
|---|---|---|
| Autoregressive | strong token-to-token coherence | sequential critical path |
| Parallel / DFlash | very low draft latency, deeper backbone possible | weak intra-block dependency |

The meta-principle:

> Eliminating a dependency does not simply reduce latency. It creates a computational budget that can be reinvested into better representations.

## Context injection and shared heads

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

For the prompt "The capital of France is", the target's hidden state may already encode:

<div class="panel">
  <div class="tokens">
    <span class="chip lg">France</span><span class="arrow">+</span><span class="chip lg">capital question</span><span class="arrow">+</span><span class="chip good lg">city expected</span><span class="arrow">≈</span><span class="chip good lg">Paris</span>
  </div>
</div>

The draft model receives this compressed understanding instead of reconstructing it from scratch.

Shared LM head means the draft's hidden state is interpreted using the target's vocabulary geometry. It does not guarantee identical probabilities, but it improves alignment at the interface.

> A cheap component need not duplicate an expensive component if it can reuse the expensive component's intermediate representation.

## Multi-modal collision

Parallel prediction can preserve correct token marginals while destroying the joint sequence. Suppose two continuations are valid — "of course" and "no problem" — each with probability $0.5$, and cross-combinations have probability zero. An independent parallel predictor sees correct marginals:

$$
P(X_1=\text{of})=P(X_1=\text{no})=0.5,\qquad P(X_2=\text{course})=P(X_2=\text{problem})=0.5
$$

and then samples as if $\hat P(X_1,X_2)=P(X_1)P(X_2)$. Now it invents cross-world pairs:

<figure class="fig">
<svg viewBox="0 0 960 396" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-collision" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="396" fill="#ffffff"/>

  <text x="480" y="37" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">Correct marginals, wrong joint: parallel prediction can mix worlds.</text>

  <!-- Two valid continuations as chip pairs -->
  <!-- World A: of -> course (sky) -->
  <rect x="150" y="58" width="34" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="167" y="80" font-size="13" font-weight="800" fill="#1d5fa6" text-anchor="middle">of</text>
  <line x1="190" y1="75" x2="218" y2="75" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-collision)"/>
  <rect x="224" y="58" width="60" height="34" rx="9" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
  <text x="254" y="80" font-size="12" font-weight="800" fill="#1d5fa6" text-anchor="middle">course</text>
  <text x="218" y="108" font-size="11" fill="#65748a" text-anchor="middle">valid, P=0.5</text>

  <!-- World B: no -> problem (lav) -->
  <rect x="600" y="58" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="617" y="80" font-size="13" font-weight="800" fill="#6442cf" text-anchor="middle">no</text>
  <line x1="640" y1="75" x2="668" y2="75" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-collision)"/>
  <rect x="674" y="58" width="66" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
  <text x="707" y="80" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">problem</text>
  <text x="687" y="108" font-size="11" fill="#65748a" text-anchor="middle">valid, P=0.5</text>

  <!-- 2x2 grid -->
  <!-- Column headers (X2) -->
  <text x="438" y="150" font-size="12" fill="#65748a" text-anchor="middle" font-weight="700">X2 = course</text>
  <text x="618" y="150" font-size="12" fill="#65748a" text-anchor="middle" font-weight="700">X2 = problem</text>

  <!-- Row labels (X1) -->
  <text x="300" y="206" font-size="12" fill="#65748a" text-anchor="middle" font-weight="700">X1 = of</text>
  <text x="300" y="294" font-size="12" fill="#65748a" text-anchor="middle" font-weight="700">X1 = no</text>

  <!-- Row 1, Col 1: of + course -> valid (mint) -->
  <rect x="350" y="160" width="176" height="76" rx="14" fill="#d8f0de" stroke="#a8e7ad" stroke-width="2"/>
  <path d="M373 197 l8 9 l16 -19" fill="none" stroke="#1f7a45" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
  <text x="448" y="194" font-size="13" font-weight="800" fill="#1f7a45" text-anchor="middle">of + course</text>
  <text x="448" y="216" font-size="11" fill="#1f7a45" text-anchor="middle">valid</text>

  <!-- Row 1, Col 2: of + problem -> mode collision (coral) -->
  <rect x="530" y="160" width="176" height="76" rx="14" fill="#ffe1e1" stroke="#f4b1b1" stroke-width="2"/>
  <path d="M553 188 l16 16 M569 188 l-16 16" fill="none" stroke="#c0392b" stroke-width="3" stroke-linecap="round"/>
  <text x="630" y="194" font-size="13" font-weight="800" fill="#c0392b" text-anchor="middle">of + problem</text>
  <text x="630" y="216" font-size="11" fill="#c0392b" text-anchor="middle">mode collision</text>

  <!-- Row 2, Col 1: no + course -> mode collision (coral) -->
  <rect x="350" y="248" width="176" height="76" rx="14" fill="#ffe1e1" stroke="#f4b1b1" stroke-width="2"/>
  <path d="M373 276 l16 16 M389 276 l-16 16" fill="none" stroke="#c0392b" stroke-width="3" stroke-linecap="round"/>
  <text x="448" y="282" font-size="13" font-weight="800" fill="#c0392b" text-anchor="middle">no + course</text>
  <text x="448" y="304" font-size="11" fill="#c0392b" text-anchor="middle">mode collision</text>

  <!-- Row 2, Col 2: no + problem -> valid (mint) -->
  <rect x="530" y="248" width="176" height="76" rx="14" fill="#d8f0de" stroke="#a8e7ad" stroke-width="2"/>
  <path d="M553 285 l8 9 l16 -19" fill="none" stroke="#1f7a45" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
  <text x="630" y="282" font-size="13" font-weight="800" fill="#1f7a45" text-anchor="middle">no + problem</text>
  <text x="630" y="304" font-size="11" fill="#1f7a45" text-anchor="middle">valid</text>

  <!-- Bottom explanatory line -->
  <text x="480" y="364" font-size="12" fill="#65748a" text-anchor="middle">Independent sampling uses P(X1)P(X2): it keeps each marginal but invents cross-world pairs.</text>
</svg>
<figcaption>Even a perfect independent predictor can learn every position's marginal and still fail to preserve which tokens belong to the same world. The missing ingredient is branch identity.</figcaption>
</figure>

This is not a low-capacity problem. The missing ingredient is branch identity.

> Dependencies are not merely computational obstacles. Some dependencies are communication channels.

## Conditional acceptance curves

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

This is why the DSpark paper's position-wise analysis is so useful: it shows DFlash's strong early-token capacity, Eagle3's sequential coherence, and DSpark's attempt to combine both.

## DSpark design thesis

DSpark adds one small fix for each bottleneck:

<div class="cols2">
  <div class="card pink">
    <span class="tag" style="background:#fff;border-color:var(--pink-edge);color:var(--pink-ink)">problem</span>
    <h4>Parallel suffix decay</h4>
    <p class="small">Later draft tokens do not condition on actual earlier draft tokens.</p>
    <strong style="color:var(--lav-ink)">Fix: tiny sequential head</strong>
  </div>
  <div class="card pink">
    <span class="tag" style="background:#fff;border-color:var(--pink-edge);color:var(--pink-ink)">problem</span>
    <h4>Verification waste</h4>
    <p class="small">Long low-confidence suffixes occupy target-model batch slots.</p>
    <strong style="color:var(--amber-ink)">Fix: confidence scheduler</strong>
  </div>
</div>

Given prompt tokens `ABC`, the target first produces anchor `D`. Then DSpark drafts from that anchor:

<div class="panel">
  <div class="tokens" style="margin-bottom:.6rem"><span class="chip good lg">ABC</span><span class="arrow">Target →</span><span class="chip good">D</span><span class="arrow">anchor</span></div>
  <div class="tokens" style="margin-bottom:.6rem"><span class="chip lg">Parallel</span><span class="arrow">→</span><span class="chip">E</span><span class="chip">G</span><span class="chip">J</span><span class="chip">K</span></div>
  <div class="tokens" style="margin-bottom:.6rem"><span class="chip lav lg">Sequential head</span><span class="arrow">→</span><span class="chip good">E</span><span class="chip good">F</span><span class="chip good">H</span><span class="chip warn">I</span></div>
  <div class="tokens"><span class="chip amber lg">Scheduler</span><span class="arrow">→</span><span class="chip good">E</span><span class="chip good">F</span><span class="chip good">H</span><span class="chip ghost">I</span></div>
</div>

The backbone writes quickly. The head proofreads line by line. The scheduler tears off the low-confidence suffix before sending it to the target editor.

The deeper engineering principle:

> Preserve the high-performance path. Repair only the failure mode.

## Parallel backbone stage

DSpark separates expensive global prediction from cheap sequential correction.

<figure class="fig">
<svg viewBox="0 0 960 420" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-backbone" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="420" fill="#ffffff"/>

  <text x="480" y="34" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">Intelligence in the parallel path; only dependency repair in the serial path.</text>

  <!-- ghost input: anchor + masks -->
  <rect x="34" y="96" width="108" height="74" rx="14" fill="#eef1f5" stroke="#dde3ea" stroke-width="2"/>
  <text x="88" y="128" font-size="12.5" font-weight="800" fill="#9aa6b3" text-anchor="middle">anchor</text>
  <text x="88" y="146" font-size="12.5" font-weight="800" fill="#9aa6b3" text-anchor="middle">+ masks</text>

  <!-- one big parallel arrow into backbone -->
  <line x1="142" y1="133" x2="168" y2="133" stroke="#b8c2cf" stroke-width="7" marker-end="url(#ah-backbone)"/>

  <!-- TOP: parallel backbone box -->
  <rect x="170" y="76" width="730" height="118" rx="18" fill="#dcebfb" stroke="#a9cdf2" stroke-width="2"/>
  <text x="535" y="106" font-size="15.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">Parallel backbone</text>
  <text x="535" y="126" font-size="11.5" fill="#65748a" text-anchor="middle">one parallel pass</text>

  <!-- seven sky tensor squares h1..h7 on the bottom edge -->
  <g>
    <rect x="227" y="150" width="28" height="28" rx="7" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
    <text x="241" y="168" font-size="10.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">h1</text>
    <rect x="325" y="150" width="28" height="28" rx="7" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
    <text x="339" y="168" font-size="10.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">h2</text>
    <rect x="423" y="150" width="28" height="28" rx="7" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
    <text x="437" y="168" font-size="10.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">h3</text>
    <rect x="521" y="150" width="28" height="28" rx="7" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
    <text x="535" y="168" font-size="10.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">h4</text>
    <rect x="619" y="150" width="28" height="28" rx="7" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
    <text x="633" y="168" font-size="10.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">h5</text>
    <rect x="717" y="150" width="28" height="28" rx="7" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
    <text x="731" y="168" font-size="10.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">h6</text>
    <rect x="815" y="150" width="28" height="28" rx="7" fill="#dcebfb" stroke="#a9cdf2" stroke-width="1.5"/>
    <text x="829" y="168" font-size="10.5" font-weight="800" fill="#1d5fa6" text-anchor="middle">h7</text>
  </g>

  <!-- thin parallel feed lines from each h down into the head box -->
  <g stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-backbone)">
    <line x1="241" y1="180" x2="241" y2="230"/>
    <line x1="339" y1="180" x2="339" y2="230"/>
    <line x1="437" y1="180" x2="437" y2="230"/>
    <line x1="535" y1="180" x2="535" y2="230"/>
    <line x1="633" y1="180" x2="633" y2="230"/>
    <line x1="731" y1="180" x2="731" y2="230"/>
    <line x1="829" y1="180" x2="829" y2="230"/>
  </g>

  <!-- BOTTOM: lightweight sequential head -->
  <rect x="170" y="236" width="730" height="58" rx="16" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="2"/>
  <text x="535" y="262" font-size="14.5" font-weight="800" fill="#6442cf" text-anchor="middle">lightweight sequential head</text>
  <text x="535" y="281" font-size="11.5" fill="#65748a" text-anchor="middle">serial repair, tiny per-step cost</text>

  <!-- short drops from head to each output chip -->
  <g stroke="#c4b9f3" stroke-width="2">
    <line x1="241" y1="294" x2="241" y2="320"/>
    <line x1="339" y1="294" x2="339" y2="320"/>
    <line x1="437" y1="294" x2="437" y2="320"/>
    <line x1="535" y1="294" x2="535" y2="320"/>
    <line x1="633" y1="294" x2="633" y2="320"/>
    <line x1="731" y1="294" x2="731" y2="320"/>
    <line x1="829" y1="294" x2="829" y2="320"/>
  </g>

  <!-- seven lavender output chips x1..x7 -->
  <g>
    <rect x="224" y="320" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
    <text x="241" y="342" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">x1</text>
    <rect x="322" y="320" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
    <text x="339" y="342" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">x2</text>
    <rect x="420" y="320" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
    <text x="437" y="342" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">x3</text>
    <rect x="518" y="320" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
    <text x="535" y="342" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">x4</text>
    <rect x="616" y="320" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
    <text x="633" y="342" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">x5</text>
    <rect x="714" y="320" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
    <text x="731" y="342" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">x6</text>
    <rect x="812" y="320" width="34" height="34" rx="9" fill="#e9e4fb" stroke="#c4b9f3" stroke-width="1.5"/>
    <text x="829" y="342" font-size="12" font-weight="800" fill="#6442cf" text-anchor="middle">x7</text>
  </g>

  <!-- tiny SERIAL arrows only between consecutive x chips -->
  <g stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-backbone)">
    <line x1="258" y1="337" x2="318" y2="337"/>
    <line x1="356" y1="337" x2="416" y2="337"/>
    <line x1="454" y1="337" x2="514" y2="337"/>
    <line x1="552" y1="337" x2="612" y2="337"/>
    <line x1="650" y1="337" x2="712" y2="337"/>
    <line x1="748" y1="337" x2="810" y2="337"/>
  </g>

  <!-- contrast captions -->
  <text x="88" y="190" font-size="11" fill="#8b97a8" text-anchor="middle">one big parallel arrow</text>
  <text x="535" y="392" font-size="11" fill="#8b97a8" text-anchor="middle">many tiny serial arrows, left to right</text>
</svg>
<figcaption>The backbone does the heavy lifting once, in parallel. The lightweight head only repairs token-to-token dependencies, one cheap step at a time.</figcaption>
</figure>

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

## Markov sequential head

The Markov head adds a low-rank transition bias based only on the previous sampled token:

$$
p_k(v\mid x_{k-1})=\operatorname{softmax}\left(U_k(v)+B(x_{k-1},v)\right)
$$

If the previous token is `of`, the head can boost `course` and suppress `problem`.

<div class="cols2">
  <div class="card">
    <h4>Backbone logits</h4>
    <div class="bars" style="margin:.4rem 0 0">
      <div class="bar-row"><span class="lbl">course</span><span class="bar"><i style="width:82%"></i></span></div>
      <div class="bar-row"><span class="lbl">problem</span><span class="bar"><i style="width:80%"></i></span></div>
    </div>
  </div>
  <div class="card">
    <h4>After Markov bias: previous = of</h4>
    <div class="bars" style="margin:.4rem 0 0">
      <div class="bar-row"><span class="lbl">course</span><span class="bar"><i class="mint" style="width:96%"></i></span></div>
      <div class="bar-row"><span class="lbl">problem</span><span class="bar"><i class="amber" style="width:45%"></i></span></div>
    </div>
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

## RNN sequential head

The RNN head keeps a small running state instead of only the previous token:

$$
s_k=f(s_{k-1},E(x_{k-1}),h_k)
$$

$$
z_k=U_k+W_os_k
$$

This helps when the right next token depends on longer draft history.

Example:

> "The keys to the cabinet …"

A Markov head sees only `cabinet` and may prefer `is`. An RNN state can remember the earlier plural subject `keys` and boost `are`.

The tradeoff is deployment cost: recurrent state reads/writes, less fusion, stricter ordering, and batching complexity.

So the production question is not:

> Which head has the highest acceptance?

It is:

> Which head gives the best end-to-end latency and operational reliability?

In the paper's production setup, DSpark uses the Markov head.

## Confidence head

The confidence head predicts conditional survival:

$$
c_k=P(A_k=1\mid A_1=\cdots=A_{k-1}=1)
$$

It is not generating better tokens. It is deciding whether spending target verification on this position is likely to be useful.

<div class="panel">
  <div class="tokens">
    <span class="chip good">x₁<small>.97</small></span>
    <span class="chip good">x₂<small>.94</small></span>
    <span class="chip warn">x₃<small>.72</small></span>
    <span class="chip bad">x₄<small>.20</small></span>
    <span class="chip ghost">x₅<small>.69</small></span>
  </div>
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

## Total variation label

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

## Losslessness and non-anticipation

A scheduler must not use information caused by a sampled token to decide whether that same token should be included.

<div class="cols2">
  <div class="card mint">
    <h4>Valid order</h4>
    <div class="tokens" style="justify-content:flex-start">
      <span class="chip good">past info</span><span class="arrow">→</span><span class="chip good">decide budget</span><span class="arrow">→</span><span class="chip good">sample/verify</span>
    </div>
  </div>
  <div class="card">
    <h4 style="color:var(--coral-ink)">Invalid order</h4>
    <div class="tokens" style="justify-content:flex-start">
      <span class="chip bad">sample future</span><span class="arrow">→</span><span class="chip bad">inspect future</span><span class="arrow">→</span><span class="chip bad">revise earlier</span>
    </div>
  </div>
</div>

Suppose the target samples:

$$
P(A)=0.7,\qquad P(B)=0.3
$$

If `A` tends to lead to confident futures and `B` tends to lead to uncertain futures, a future-peeking scheduler would keep more `A` paths. The visible output might become 85/15 instead of 70/30.

That changes the target distribution.

Early stopping in the ideal scheduler prevents this: once the scheduler decides the next extension is not worth admitting, it stops and does not inspect later tokens to reconsider earlier decisions.

## Training objective

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

An early $c_i$ appears in more terms of this sum. That is the mathematical version of "the first stone supports the whole bridge."

## Sequential temperature scaling

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

## Scheduler objective

For a batch, the scheduler chooses how many draft tokens to verify per request.

The throughput objective is:

$$
\Theta=\tau^*\times \operatorname{SPS}(B)
$$

where:

- $\tau^*$ is expected accepted tokens per verification round;
- $\operatorname{SPS}(B)$ is the profiled number of target verification steps per second at batch size $B$.

Adding another verification token has two effects:

<div class="cols2">
  <div class="card mint"><h4>Benefit</h4><p class="small">Expected accepted tokens may rise.</p></div>
  <div class="card pink"><h4>Cost</h4><p class="small">Verification batch grows, so SPS may fall.</p></div>
</div>

Example:

| Choice | expected accepted tokens | SPS | throughput |
|---|---:|---:|---:|
| verify 3 | 2.4 | 100 | 240 |
| verify 4, high survival | 3.2 | 95 | 304 |
| verify 4, low survival | 2.6 | 80 | 208 |

The scheduler should add the fourth token only in the high-survival case.

## Greedy prefix allocation

Each request offers prefix extensions with decreasing cumulative survival. Sort all candidate prefix extensions globally and scan the greedy admission path while checking throughput:

<figure class="fig">
<svg viewBox="0 0 960 372" xmlns="http://www.w3.org/2000/svg" font-family="-apple-system,Helvetica Neue,Arial,sans-serif">
  <defs><marker id="ah-scheduler" markerWidth="9" markerHeight="9" refX="6.5" refY="4" orient="auto"><path d="M0 0 L8 4 L0 8 z" fill="#b8c2cf"/></marker></defs>
  <rect x="0" y="0" width="960" height="372" fill="#ffffff"/>

  <text x="480" y="34" font-size="16" font-weight="800" fill="#22303f" text-anchor="middle">Sort candidate prefixes by survival; admit the valuable ones until the budget runs out.</text>

  <!-- ===================== LEFT: two request rows ===================== -->
  <text x="60" y="78" font-size="13" font-weight="800" fill="#65748a">Per-request chains</text>
  <text x="60" y="95" font-size="11" fill="#8b97a8">cumulative survival decreases along each chain</text>

  <!-- Request A -->
  <text x="60" y="146" font-size="12.5" font-weight="800" fill="#22303f">Request A</text>
  <rect x="128" y="120" width="92" height="44" rx="9" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="174" y="139" font-size="13" font-weight="800" fill="#1f7a45" text-anchor="middle">A1</text>
  <text x="174" y="155" font-size="11.5" fill="#1f7a45" text-anchor="middle">.90</text>
  <line x1="226" y1="142" x2="268" y2="142" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-scheduler)"/>
  <rect x="274" y="120" width="92" height="44" rx="9" fill="#fdeecb" stroke="#f1d089" stroke-width="1.5"/>
  <text x="320" y="139" font-size="13" font-weight="800" fill="#a9760f" text-anchor="middle">A2</text>
  <text x="320" y="155" font-size="11.5" fill="#a9760f" text-anchor="middle">.70</text>

  <!-- Request B -->
  <text x="60" y="226" font-size="12.5" font-weight="800" fill="#22303f">Request B</text>
  <rect x="128" y="200" width="92" height="44" rx="9" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="174" y="219" font-size="13" font-weight="800" fill="#1f7a45" text-anchor="middle">B1</text>
  <text x="174" y="235" font-size="11.5" fill="#1f7a45" text-anchor="middle">.80</text>
  <line x1="226" y1="222" x2="268" y2="222" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-scheduler)"/>
  <rect x="274" y="200" width="92" height="44" rx="9" fill="#ffe1e1" stroke="#f4b1b1" stroke-width="1.5"/>
  <text x="320" y="219" font-size="13" font-weight="800" fill="#c0392b" text-anchor="middle">B2</text>
  <text x="320" y="235" font-size="11.5" fill="#c0392b" text-anchor="middle">.30</text>

  <!-- bridge arrow into the queue -->
  <line x1="392" y1="182" x2="470" y2="182" stroke="#b8c2cf" stroke-width="2.5" marker-end="url(#ah-scheduler)"/>
  <text x="431" y="172" font-size="11" fill="#8b97a8" text-anchor="middle">sort</text>

  <!-- ===================== RIGHT: admission queue ===================== -->
  <text x="496" y="78" font-size="13" font-weight="800" fill="#65748a">Global admission queue</text>
  <text x="496" y="95" font-size="11" fill="#8b97a8">sorted by survival, high to low</text>

  <!-- queue track -->
  <rect x="496" y="118" width="436" height="92" rx="16" fill="#ffffff" stroke="#e4eaf1" stroke-width="2"/>

  <!-- A1 .90 (admitted, mint) -->
  <rect x="514" y="142" width="78" height="44" rx="9" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="553" y="161" font-size="13" font-weight="800" fill="#1f7a45" text-anchor="middle">A1</text>
  <text x="553" y="177" font-size="11.5" fill="#1f7a45" text-anchor="middle">.90</text>

  <!-- B1 .80 (admitted, mint) -->
  <rect x="600" y="142" width="78" height="44" rx="9" fill="#d8f0de" stroke="#a8e7ad" stroke-width="1.5"/>
  <text x="639" y="161" font-size="13" font-weight="800" fill="#1f7a45" text-anchor="middle">B1</text>
  <text x="639" y="177" font-size="11.5" fill="#1f7a45" text-anchor="middle">.80</text>

  <!-- A2 .70 (admitted, amber - last in budget) -->
  <rect x="686" y="142" width="78" height="44" rx="9" fill="#fdeecb" stroke="#f1d089" stroke-width="1.5"/>
  <text x="725" y="161" font-size="13" font-weight="800" fill="#a9760f" text-anchor="middle">A2</text>
  <text x="725" y="177" font-size="11.5" fill="#a9760f" text-anchor="middle">.70</text>

  <!-- budget cutoff (dashed) after A2 -->
  <line x1="784" y1="120" x2="784" y2="208" stroke="#a9760f" stroke-width="2" stroke-dasharray="5 4"/>
  <text x="784" y="111" font-size="11" font-weight="800" fill="#a9760f" text-anchor="middle">budget cutoff</text>

  <!-- B2 .30 (skipped, beyond cutoff - ghost/coral) -->
  <rect x="806" y="142" width="78" height="44" rx="9" fill="#eef1f5" stroke="#dde3ea" stroke-width="1.5"/>
  <text x="845" y="161" font-size="13" font-weight="800" fill="#c0392b" text-anchor="middle">B2</text>
  <text x="845" y="177" font-size="11.5" fill="#9aa6b3" text-anchor="middle">.30</text>
  <line x1="819" y1="170" x2="871" y2="170" stroke="#c0392b" stroke-width="1.4"/>

  <!-- verify / skip labels under the queue -->
  <text x="639" y="203" font-size="12" font-weight="800" fill="#1f7a45" text-anchor="middle">verify</text>
  <text x="845" y="203" font-size="12" font-weight="800" fill="#9aa6b3" text-anchor="middle">skip</text>

  <!-- ===================== note line ===================== -->
  <line x1="60" y1="312" x2="900" y2="312" stroke="#e4eaf1" stroke-width="1"/>
  <text x="480" y="340" font-size="12.5" fill="#65748a" text-anchor="middle">Cumulative survival is non-increasing, so prerequisites are automatically respected (A2 can never precede A1).</text>
</svg>
<figcaption>Sorting by cumulative survival automatically respects prerequisites: A2 cannot appear before A1 because cumulative survival is non-increasing along each chain.</figcaption>
</figure>

The paper's scheduler uses this monotonic structure to turn a combinatorial allocation problem into "sort candidates, then scan the greedy admission path while checking throughput."

## Offline evaluation setup

Offline evaluation disables the scheduler and forces all drafters to propose a fixed block size. Why? Because we want to isolate raw draft quality.

<div class="panel">
  <div class="flow">
    <div class="node"><span class="t">Context</span></div>
    <span class="arrow">→</span>
    <div class="node sky"><span class="t">Draft fixed K</span></div>
    <span class="arrow">→</span>
    <div class="node pink"><span class="t">Target verifies</span></div>
    <span class="arrow">→</span>
    <div class="node mint"><span class="t">Measure τ</span></div>
  </div>
</div>

This is like a wind-tunnel test. Turn off the pilot so the aircraft design is measured under the same conditions.

The DSpark paper evaluates across math, code, and chat, comparing against Eagle3 and DFlash in the same training framework.

## Main results and mechanism

The reported offline result is that DSpark improves accepted length over both a strong autoregressive baseline and a strong parallel baseline.

The mechanism is easier to see position-wise:

<div class="panel">
  <div class="bars" style="max-width:560px">
    <div class="bar-row"><span class="lbl">DFlash p1</span><span class="bar"><i class="sky" style="width:92%"></i></span></div>
    <div class="bar-row"><span class="lbl">DFlash suffix</span><span class="bar"><i class="sky" style="width:45%"></i></span></div>
    <div class="bar-row"><span class="lbl">Eagle3 p1</span><span class="bar"><i class="lav" style="width:72%"></i></span></div>
    <div class="bar-row"><span class="lbl">Eagle3 suffix</span><span class="bar"><i class="lav" style="width:74%"></i></span></div>
    <div class="bar-row"><span class="lbl">DSpark p1</span><span class="bar"><i class="mint" style="width:92%"></i></span></div>
    <div class="bar-row"><span class="lbl">DSpark suffix</span><span class="bar"><i class="mint" style="width:78%"></i></span></div>
  </div>
</div>

DFlash has strong first-token quality because the parallel backbone can be deeper under the same latency budget. Eagle3 has better suffix coherence because it is autoregressive. DSpark tries to keep the first and recover the second.

The broader principle:

> Performance along a chain is dominated by both the strength of its entry point and the propagation of consistency afterward.

## A little autoregression goes a long way

Adding more parallel capacity does not fully solve a wrong factorization. A 5-layer DFlash can produce stronger independent guesses, but it still mostly answers:

$$
P(x_k\mid \text{context})
$$

DSpark's sequential head moves closer to:

$$
P(x_k\mid \text{context},x_{<k})
$$

As proposal length grows, that missing dependency matters more.

<div class="callout">
  <p><strong>Analogy.</strong> Adding lanes does not fix a bad intersection. Sometimes one traffic officer gives more value than another lane.</p>
</div>

Correct inductive bias can dominate brute-force capacity.

## Production training optimizations

Training the drafter requires target-model supervision. The paper describes two production optimizations.

### Hidden-state communication

Instead of moving full vocabulary logits across workers, move the hidden states before the LM head and compute selected logits locally.

If vocabulary size is about 100,000 and hidden dimension is about 4,096, the raw size ratio is roughly 24×.

<div class="panel">
  <div class="flow">
    <div class="node sky"><span class="t">Target worker</span><span class="s">hidden vector h</span></div>
    <span class="arrow">→</span>
    <div class="node lav"><span class="t">Draft worker</span><span class="s">LM head locally</span></div>
  </div>
</div>

### Anchor-bounded sequence packing

Do not train at every token position. Sample anchor positions and train draft blocks around those anchors.

For a 1,000-token document and block length 4, training everywhere would create roughly $1000\times4$ draft predictions. Sampling three anchors creates only $3\times4=12$ draft predictions.

<pre>Full document:  t1 t2 t3 ... t1000
Anchors:                     [100 → 101..104]   [500 → 501..504]   [800 → 801..804]
Packed batch:      [anchor block 1] [anchor block 2] [anchor block 3]</pre>

Attention indices keep packed blocks logically independent even though they share one physical batch.

> Move the smallest sufficient representation, and compute only where supervision exists.

## Async scheduler in practice

The ideal scheduler chooses capacity from current confidence. Production serving complicates this because CUDA graph replay and zero-overhead scheduling need the next batch size before the current step finishes.

If the GPU waits for the scheduler, latency is lost:

<div class="panel">
  <div class="tokens"><span class="chip lg">GPU step t</span><span class="arrow">→</span><span class="chip bad">wait</span><span class="arrow">→</span><span class="chip lg">GPU step t+1</span></div>
</div>

DSpark uses an asynchronous approximation:

<div class="panel">
  <div class="tokens"><span class="chip good lg">confidence t−2</span><span class="arrow">→</span><span class="chip good lg">choose capacity K for t+1</span><span class="arrow">→</span><span class="chip good lg">current scores fill K slots</span></div>
</div>

Old information decides **how many seats** the bus has. Current information decides **which passengers** get those seats.

This does two useful things:

1. hides scheduling latency behind GPU work;
2. prevents capacity from expanding after seeing current sampled consequences.

Delay becomes a causal barrier.

## Variable-length verification without wasting kernels

The scheduler produces different verification lengths per request:

<div class="cols3">
  <div class="card"><h4>Request A</h4><div class="tokens" style="justify-content:flex-start"><span class="chip good">A1</span><span class="chip good">A2</span><span class="chip good">A3</span></div></div>
  <div class="card"><h4>Request B</h4><div class="tokens" style="justify-content:flex-start"><span class="chip good">B1</span></div></div>
  <div class="card"><h4>Request C</h4><div class="tokens" style="justify-content:flex-start"><span class="chip good">C1</span><span class="chip good">C2</span></div></div>
</div>

Naively padding every request to the same length wastes compute:

<div class="panel">
  <div class="tokens" style="justify-content:flex-start">
    <span class="chip good">A1</span><span class="chip good">A2</span><span class="chip good">A3</span>
    <span class="chip good">B1</span><span class="chip ghost">pad</span><span class="chip ghost">pad</span>
    <span class="chip good">C1</span><span class="chip good">C2</span><span class="chip ghost">pad</span>
  </div>
</div>

The production solution described in the paper is to flatten all verified tokens into one physical batch while carrying marker tensors that preserve logical sequence structure inside sparse attention kernels:

<div class="panel">
  <div class="tokens">
    <span class="chip good">A1</span><span class="chip good">A2</span><span class="chip good">A3</span><span class="chip good">B1</span><span class="chip good">C1</span><span class="chip good">C2</span>
  </div>
</div>

Logical dependency is tracked by metadata, not by padding the physical execution shape. This is a common systems trick: decouple the physical layout that hardware likes from the logical structure the algorithm needs.

## Live traffic Pareto shift

The paper reports DSpark-5 deployed against the prior MTP-1 production baseline in DeepSeek-V4-Flash and DeepSeek-V4-Pro preview serving.

At matched practical throughput levels, DSpark improves per-user generation speed by:

- **60%–85%** on V4-Flash;
- **57%–78%** on V4-Pro.

At moderate SLA anchors, it improves aggregate throughput by about:

- **51%** at 80 tok/s/user on V4-Flash;
- **52%** at 35 tok/s/user on V4-Pro.

<div class="panel">
  <div class="bars" style="max-width:600px">
    <div class="bar-row"><span class="lbl">Flash SLA 80</span><span class="bar"><i class="mint" style="width:51%"></i></span><span class="val">+51% TPS</span></div>
    <div class="bar-row"><span class="lbl">Pro SLA 35</span><span class="bar"><i class="mint" style="width:52%"></i></span><span class="val">+52% TPS</span></div>
    <div class="bar-row"><span class="lbl">Flash matched</span><span class="bar"><i class="sky" style="width:85%"></i></span><span class="val">+60–85%/user</span></div>
    <div class="bar-row"><span class="lbl">Pro matched</span><span class="bar"><i class="sky" style="width:78%"></i></span><span class="val">+57–78%/user</span></div>
  </div>
</div>

At strict SLA anchors, the baseline approaches a low-concurrency boundary. The large relative gains there are best interpreted as "DSpark keeps the system in a usable operating regime," not as a stable multiplicative speedup.

The key production behavior is adaptive:

- light/moderate load: use spare compute to verify longer prefixes;
- heavy load: restrict verification to protect batch capacity.

Adaptive systems convert spare capacity into latency improvement without committing that extra cost under congestion.

## Limitations and future direction

DSpark adapts verification after drafting. It may still generate a full $\gamma$-token block even when the scheduler later keeps only one or two tokens:

<div class="panel">
  <div class="tokens"><span class="chip lg">drafted</span><span class="arrow">→</span><span class="chip good">A</span><span class="chip good">B</span><span class="chip ghost">C</span><span class="chip ghost">D</span><span class="chip ghost">E</span></div>
</div>

So DSpark removes target-side waste first — fixed drafting paired with adaptive verification. The next step is likely adaptive drafting paired with adaptive verification:

<div class="panel">
  <div class="flow">
    <div class="node sky"><span class="t">fixed drafting</span></div>
    <span class="arrow">→</span>
    <div class="node mint"><span class="t">adaptive verification</span><span class="s">today</span></div>
    <span class="arrow">⇒</span>
    <div class="node lav"><span class="t">adaptive drafting</span></div>
    <span class="arrow">→</span>
    <div class="node mint"><span class="t">adaptive verification</span><span class="s">next</span></div>
  </div>
</div>

Hard requests could early-exit from the draft backbone or choose a smaller draft length before paying for all positions. This follows the broader pattern:

> Optimization removes waste one stage at a time. Move the decision closer to where the wasted work begins.

## Final compression

Speculative decoding is exact because accepted draft mass plus residual correction mass equals the target distribution.

Accepted length is a survival process because each later token exists only if every earlier token survived.

Parallel drafting creates speed by shortening the critical path, but it loses branch identity and causes suffix decay.

DSpark keeps the parallel backbone, adds a tiny sequential head for coherence, and uses calibrated confidence to spend verification where prefix survival justifies the hardware cost.

The transferable principle is:

<div class="callout good">
  <p><strong>Speculate aggressively inside the system; enforce the invariant at commitment; allocate scarce verification compute by expected useful work, not by candidate availability.</strong></p>
</div>

</div>
