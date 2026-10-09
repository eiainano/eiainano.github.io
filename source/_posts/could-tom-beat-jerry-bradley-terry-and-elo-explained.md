---
title: Could Tom Beat Jerry? Bradley-Terry and Elo explained
date: 2026-09-11 23:41:00
tags:
  - ranking
---

Every rating system ever built answers the same impossibly small question: *given a pile of who-beat-whom, how strong is everybody?*

Tom and Jerry is the purest form of it. We have a few hundred episodes. We know who came out on top in each one. We never see a number. And yet we want a number — one per character — that lets us say something about an episode that hasn't been made yet, and about a matchup that never happened.

That is the whole problem. Everything below is one particular answer to it, worked out slowly, and then the same answer again wearing different clothes.

(One note on presentation: this site runs no JavaScript, so the maths is set in monospaced blocks rather than rendered LaTeX. It reads fine, and it loads in zero milliseconds.)

---

## 1. What we have, and what we want

Fix a set of `N` players. For every ordered pair we record two counts:

```
w_ij  =  number of times i beat j
n_ij  =  w_ij + w_ji   =  number of games between i and j
```

That's the data. Note what is *not* in it: no score margins, no clock, no context, no notion of how close a game was. One bit per game.

Now what we want. Not a ranking — a **scale**. A ranking is ordinal: it tells you Jerry is above Tom. A scale is cardinal: it tells you *how far* above, and that extra structure is what buys you three things a ranking can't:

1. **Prediction.** `P(Tom beats Jerry in episode 162)`.
2. **Composition.** If you know the gap from A to B and from B to C, you get A to C for free — even if A and C never met.
3. **Arithmetic.** You can average, interpolate, and put error bars on it.

The crucial thing to internalise: **strength is not observable.** It is a latent quantity that we postulate and then fit. Every rating you have ever seen is a model output, not a measurement. This matters more than it sounds, because it means the interesting question is never "how do we compute the rating" but "what did we assume to make the rating computable at all."

And one more piece of motivation before we derive anything. Win rate is useless on its own, because **schedule strength is not a detail, it is most of the signal.** A player who goes 90–10 against beginners is probably worse than a player who goes 20–20 against champions. Jerry's record looks pristine, and it looks pristine against exactly one opponent. Any honest rating has to discount the record by the quality of the opposition — which is circular, because the quality of the opposition is the thing we're trying to compute. Bradley–Terry is the cleanest way to break that circle.

---

## 2. Four wishes, one model

We're going to *derive* the model from desiderata rather than pull it out of a hat. This is worth doing once, because it makes the assumptions visible, and visible assumptions are the only kind you can argue with.

Let `P(i ≻ j)` be the probability that `i` beats `j`. We want a number `θ_i` per player such that:

### Wish 1 — only differences matter

```
P(i ≻ j) = F(θ_i − θ_j)        for some increasing F,  F(0) = 1/2
```

Justification: `θ` is a latent scale with no origin. Adding 100 to every player changes nothing real, so the model must be blind to it. Working with differences is forced.

### Wish 2 — the two outcomes are exhaustive

```
P(i ≻ j) + P(j ≻ i) = 1   ⟹   F(−x) = 1 − F(x)
```

So `F` is symmetric about `(0, 1/2)`. This is also the moment to switch from probabilities to **odds**, which is where the structure lives:

```
        P(i ≻ j)
O(i,j) = ─────────  =  g(θ_i − θ_j),      and  g(−x) = 1/g(x)
        P(j ≻ i)
```

### Wish 3 — advantages add

Here is the load-bearing one. For any three players:

```
O(i,k) = O(i,j) · O(j,k)
```

Read it in words: *the edge you hold over me, times the edge I hold over her, is the edge you hold over her.* Or, in the form that shows why it's called a scale: **if you put advantages on a ruler, they compose by addition.** This single line is what distinguishes a cardinal rating from an ordinal ranking, and it is the entire content of the Bradley–Terry model.

Substitute `g`:

```
g(θ_i − θ_j) · g(θ_j − θ_k) = g(θ_i − θ_k) = g((θ_i − θ_j) + (θ_j − θ_k))
```

So for all real `a, b`:

```
g(a + b) = g(a) · g(b)
```

### Wish 4 — regularity (the fine print)

`F` is continuous and strictly increasing, hence so is `g`. Without some condition like this, Cauchy's equation admits pathological solutions built on a Hamel basis — everywhere-discontinuous monsters that no one has ever wanted as a rating system. Continuity kills them, and this is the only place in the derivation where a purely technical assumption does real work.

With it:

```
g(x) = e^{cx}
```

and `c` is not identifiable anyway, because it is absorbed into the units of `θ`. Set `c = 1`. Done.

### The model

```
                 e^{θ_i}                     1
P(i ≻ j)  =  ─────────────────  =  σ(θ_i − θ_j)  =  ────────────────────
              e^{θ_i} + e^{θ_j}                    1 + e^{−(θ_i − θ_j)}
```

or, in the form that should be tattooed somewhere:

```
log O(i,j)  =  θ_i − θ_j
```

> **The log-odds of a matchup is the difference in strength.** That is the entire Bradley–Terry model, in one line. Everything else is bookkeeping.

Equivalently, with `γ_i = e^{θ_i} > 0`: `P(i ≻ j) = γ_i / (γ_i + γ_j)`. The `γ` parameterisation makes the geometry obvious: BT says each player owns a positive weight, and the chance of winning a matchup is their share of the total weight.

### What Wish 3 actually costs you

Be honest with yourself here, because Wish 3 is not a fact about the world — it's a purchase. It implies **stochastic transitivity**: no rock-paper-scissors. If the world genuinely has cyclic structure (style matchups in fighting games, candidate-voter geometry in elections, and — as we'll see — human preferences between LLM outputs), BT is the wrong model, and no quantity of data will fix a wrong model. It will instead silently average the cycle away into a compromise linear order.

Naming the assumption is ninety percent of modelling. BT's assumption is beautifully nameable: *there exists a ruler on which advantages add.*

### Provenance, briefly

The model is old and has been rediscovered so often that the naming is unfair to everyone: Zermelo derived it for chess tournaments in 1929; Thurstone's Case V (1927) is its Gaussian sibling; Bradley and Terry published it for paired taste tests in 1952 and got the name; Ford (1957) sorted out when it even has a solution.

---

## 3. The noise distribution hiding inside the link function

Here is the second door into the same room, and it is the one that generalises.

Stop modelling the outcome directly. Instead, say that in any given game each player *performs at some level*, and the better performance wins:

```
S_i = θ_i + ε_i ,      i wins  ⟺  S_i > S_j
```

Now the only choice left is the distribution of the noise `ε`. Everything follows from that choice, and different choices give different link functions:

| noise on performance `ε` | resulting `P(i ≻ j)` | name |
|---|---|---|
| Gumbel(0, 1) | `σ(θ_i − θ_j)` | Bradley–Terry / Zermelo |
| Gaussian(0, σ²) | `Φ((θ_i − θ_j)/(σ√2))` | Thurstone Case V (probit) |

The Gumbel case is exact, not approximate: the difference of two independent standard Gumbels is a standard **logistic** random variable. (Simulate it if you don't believe it: `P(G₁ − G₂ < ln 3) ≈ 0.7501` over 400k draws, against a predicted `0.75`.)

So:

> **Choosing a link function is choosing a noise model.** Every pairwise comparison model is a claim about the distribution of the *noise*, not about the players.

And the claim is testable, and it lives in the tails. The logistic has heavier tails than the Gaussian, so BT says freak upsets are more likely than Thurstone does. Matched at their scales the two curves never differ by more than about **0.0095** in probability — under one percentage point, everywhere — which is why nobody in chess ever lost sleep over the choice. But they diverge in the far tail, and the far tail is precisely where upsets, giant-killings, and low-probability-but-consequential events live. If you care about the 1-in-1000 outcome, the link choice is not cosmetic.

This door also generalises instantly. Take `m > 2` players, give each a Gumbel-perturbed score, and rank them: the probability of a full ordering is the **Plackett–Luce** model, of which BT is the `m = 2` special case. The Gumbel-max trick you know from sampling is the same object.

---

## 4. Fitting it: the MLE and a very pretty fixed point

Now the estimation. Maximum likelihood over all `N` strengths:

```
L(θ) = Σ_{i<j} [ w_ij · log σ(θ_i − θ_j)  +  w_ji · log σ(θ_j − θ_i) ]
```

Differentiate with respect to one `θ_i`. Using `d/dx log σ(x) = σ(−x) = 1 − σ(x)`:

```
∂L/∂θ_i = Σ_{j≠i} [ w_ij · σ(θ_j − θ_i) − w_ji · σ(θ_i − θ_j) ]
```

Write `p_ij = σ(θ_i − θ_j)` and `n_ij = w_ij + w_ji`, so `σ(θ_j − θ_i) = 1 − p_ij`:

```
        = Σ_{j≠i} [ w_ij (1 − p_ij) − w_ji p_ij ]
        = Σ_{j≠i} [ w_ij − n_ij p_ij ]
        = s_i − Σ_{j≠i} n_ij · σ(θ_i − θ_j)
```

where `s_i = Σ_j w_ij` is simply player `i`'s **total number of wins**. Setting the gradient to zero:

```
              ┌─────────────────────────────────────┐
              │  s_i  =  Σ_{j≠i} n_ij · σ(θ̂_i − θ̂_j) │      for every i
              └─────────────────────────────────────┘
```

> **At the maximum, every player's actual number of wins equals their expected number of wins under the model.**

Sit with that for a second. It is one of those results that is obvious in hindsight and deeply clarifying in foresight. Three things fall out of it:

**(a) It's a fixed point, not a closed form.** The equation for `θ_i` involves every `θ_j`. You iterate.

**(b) It's moment matching.** `s_i` is the sufficient statistic, and — true of every exponential family — the MLE is exactly the parameter vector that matches the sufficient statistics to their expectations. Equivalently: BT is the **maximum-entropy** distribution over outcomes consistent with the observed win totals. You are not assuming much; you are assuming as little as possible while reproducing what you saw.

**(c) It gives you the algorithm for free.** In `γ_i = e^{θ_i}` form:

```
              s_i
γ_i  ←  ──────────────────────────
        Σ_{j≠i} n_ij / (γ_i + γ_j)
```

This is Zermelo's 1929 iteration, rediscovered independently in the statistics literature and in every recommendation system that fits pairwise preferences. It is a handful of lines:

```python
import math

def bt_fit(win, iters=300):
    """win[i][j] = number of times i beat j.  Returns gamma = exp(theta),
    normalised so that the geometric mean is 1."""
    n = len(win)
    gamma = [1.0] * n
    for _ in range(iters):
        new = []
        for i in range(n):
            s = sum(win[i][j] for j in range(n) if j != i)
            tot = sum(win[i][j] + win[j][i] for j in range(n) if j != i)
            if s == 0 or tot == 0:
                new.append(gamma[i])          # unidentifiable: leave alone
                continue
            den = sum((win[i][j] + win[j][i]) / (gamma[i] + gamma[j])
                      for j in range(n) if j != i)
            new.append(s / den)
        # theta is identified only up to an additive constant; pin the scale
        g = math.exp(sum(math.log(x) for x in new) / n)
        gamma = [x / g for x in new]
    return gamma
```

### Does a solution exist? Not always.

**Concavity.** `d²/dx² log σ(x) = −σ(x)σ(−x) < 0`, so `log σ` is concave, and `L` is a sum of concave functions of linear forms — hence concave. Any stationary point is a global maximum, and it is unique up to the additive shift once the comparison graph is connected. Good: no local optima, no tricks needed.

**Existence — Ford's condition (1957).** A finite MLE exists if and only if, in every partition of the players into two nonempty groups `A` and `B`, someone in `A` beat someone in `B` at least once **and** someone in `B` beat someone in `A` at least once. Otherwise the likelihood is monotonically increasing along some direction and `θ` runs off to infinity.

Note what this is *not*: it is not "everyone played everyone". A cycle A≻B≻C≻A satisfies it perfectly (check each of the three partitions). What it forbids is a **clean sweep** — a group that is undefeated against, or winless against, everyone outside it. That is exactly the Tom and Jerry situation, and we will come back to it.

---

## 5. A worked example: filling an empty cell

Three players, eight games, and one matchup that never happened:

| matchup | games | result |
|---|---|---|
| A vs B | 4 | A 3, B 1 |
| B vs C | 4 | B 3, C 1 |
| A vs C | **0** | — |

The fit. `A` has played one opponent; from the two-player case (set `σ(θ̂_A − θ̂_B) = 3/4`) we get `θ̂_A − θ̂_B = ln 3 ≈ 1.0986`. Symmetrically `θ̂_B − θ̂_C = ln 3`.

Now check that this is genuinely the joint MLE and not just a pairwise guess. Apply the stationarity condition to `B`:

```
s_B = w_BA + w_BC = 1 + 3 = 4
Σ_j n_Bj · σ(θ_B − θ_j) = 4·σ(−ln 3) + 4·σ(ln 3) = 4(1/4) + 4(3/4) = 1 + 3 = 4  ✓
```

It holds exactly. And therefore:

```
θ̂_A − θ̂_C = ln 3 + ln 3 = ln 9 ≈ 2.197
P(A beats C) = σ(ln 9) = 9/10 = 0.90
```

**Nobody has ever seen A play C, and the model just returned 0.90.** This is the most useful and most dangerous feature Bradley–Terry has, and it comes entirely from Wish 3.

There's an elegant structural fact hiding here too, worth stating because it's checkable and surprising: **when the comparison graph is a tree, the MLE is exactly the sum of per-edge log-odds along the unique path.** (Proof: on each edge, `σ(θ_i − θ_j) = w_ij/n_ij`, so `Σ_j n_ij σ(θ_i − θ_j) = Σ_j w_ij = s_i` — the stationarity condition is satisfied identically.) Add a single cycle to the graph and this stops being true: the per-edge log-odds around a loop no longer sum to zero, and the MLE has to find a compromise instead of an exact fit. **Cycles are where the model is forced to admit the data disagrees with it.**

Which brings us to the price. That 0.90 is not an observation; it is an extrapolation, and it inherits every error in the chain *multiplicatively*. If A is a stylistic counter to B, if B was injured for two of those four games, if the four games were all played under conditions that favoured B — the chain propagates it. And because errors add in `θ` while odds are `e^θ`, a per-link error of `ε` becomes `kε` across a `k`-link chain, which is an error of `e^{kε}` in the odds. **Inference chains are where ratings go to die, and they die exponentially.**

---

## 6. Elo: the same model, one game at a time

Here is the Elo update rule, in the form every chess server implements:

```
E_i  =  1 / (1 + 10^{−(R_i − R_j)/400})
R_i ← R_i + K (y − E_i)
R_j ← R_j − K (y − E_i)

y = 1 if i won, 0 if i lost, 1/2 for a draw
```

Two observations.

**First, the prediction function is Bradley–Terry.** Change units to `θ = R · ln 10 / 400` and `E_i` becomes exactly `σ(θ_i − θ_j)`. The famous 400 is nothing but a choice of units: **400 rating points is one decade of odds**, i.e. `400 / ln 10 ≈ 173.72` points per natural log unit. (Elo himself derived his expected-score tables from a normal performance distribution — Thurstone's path, not Bradley's. The closed form everyone actually implements is the logistic one. As noted above the two agree to within a hundredth of a probability point, so the distinction has never mattered in practice.)

**Second, the update is a gradient step.** Take one single game between `i` and `j` with outcome `y ∈ {0,1}`. The Bradley–Terry log-likelihood for that one game is

```
ℓ(θ) = y · log σ(θ_i − θ_j) + (1 − y) · log σ(θ_j − θ_i)
```

Differentiate, writing `p_ij = σ(θ_i − θ_j)` and `σ(θ_j − θ_i) = 1 − p_ij`:

```
∂ℓ/∂θ_i = y(1 − p_ij) − (1 − y)p_ij = y − p_ij
∂ℓ/∂θ_j = −(y − p_ij)
```

Gradient ascent with step `η`: `Δθ_i = η (y − p_ij)`, `Δθ_j = −η (y − p_ij)`. Translate back to rating units, `η = K ln 10 / 400`, and you have the Elo update, character for character.

> **Bradley–Terry is the loss. Elo is SGD.**

This one identity explains everything about Elo's behaviour, so let's just cash it out:

1. **`K` is a learning rate.** Every piece of SGD intuition transfers immediately. Large `K`: ratings react fast, but they're noisy and they oscillate around the truth. Small `K`: stable, but laggy — a genuinely improved player spends months under-rated. It's a bias–variance dial and there is no free setting, which is why federations use `K`-schedules (bigger `K` for newcomers, smaller for established players) — that's a decaying step size, arrived at empirically.

2. **Elo never converges, and that is correct.** Constant step size means it moves forever. Elo is not an estimator of a fixed quantity; it is a *tracker* of a moving one. Bradley–Terry MLE is the batch limit; Elo is the stochastic-approximation version. If players improve, decline, age and retire, the batch assumption (a frozen truth) is the wrong one and Elo's restlessness is a feature.

3. **Elo is order-dependent; BT-MLE is not.** Shuffle the game log and the BT fit is bit-identical. The Elo trajectory changes. So Elo implicitly weights recent games more heavily — recency bias is built in at no extra cost, and is usually what you want.

4. **Zero-sum is a conservation law.** `ΔR_i + ΔR_j = 0` always. Rating points are never created or destroyed by play, only moved between players. Therefore the mean rating of a *closed* population is an invariant of the update rule. Which means: **long-run rating inflation cannot be caused by the Elo update.** It comes from the boundary — newcomers entering at a fixed initial rating and retiring at a different one. If entrants arrive below the mean they eventually settle at and leave above it, the population rating inflates, and no tweaking of `K` will stop it. (Real federations layer on rating floors and `K`-schedules, so this is the idealised statement; but the conservation law is what tells you where to look.)

5. **Elo carries no uncertainty.** One number per player, no error bar. A player with 3 games and a player with 300 are trusted identically by the update. Glicko and TrueSkill fix this by carrying a second number (a deviation / variance) and shrinking updates accordingly — which is, unsurprisingly, the same idea as a per-player adaptive learning rate.

6. **Elo is O(1) per game.** No matrix, no iteration, no stored history. You can run it on a scoreboard. That, and not statistical virtue, is why it won.

```python
def elo_update(r_i, r_j, y, k=32):
    """y = 1 if i won, 0 if i lost, 0.5 for a draw.  Returns updated ratings."""
    e_i = 1.0 / (1.0 + 10 ** (-(r_i - r_j) / 400))
    return r_i + k * (y - e_i), r_j - k * (y - e_i)
```

Note `ΔR_j = −K(y − E_i) = K((1 − y) − E_j)`, so the draw case `y = 1/2` is handled correctly with no extra machinery. It's the same gradient, evaluated on a `0.5` label.

---

## 7. The family portrait

| | Bradley–Terry | Elo | Glicko / TrueSkill | Plackett–Luce |
|---|---|---|---|---|
| estimates | a fixed strength | a tracked strength | strength **+ uncertainty** | a full ranking distribution |
| batch / online | batch | online | online (rating periods) | batch |
| numbers per player | 1 | 1 | 2 | 1 |
| draws | no (needs extension) | yes (`y = ½`) | yes | n/a |
| uncertainty | Fisher info / bootstrap | none | explicit | — |
| comparison arity | pairwise | pairwise | pairwise | `m`-way |
| core assumption | additive log-odds | same, + slow drift | same, + Gaussian drift | same, via Gumbel-max |

They are not competing theories. They are one theory — *advantages live on a ruler* — with four different answers to the questions "batch or online?", "how many numbers?", and "two players or `m`?".

---

## 8. Why this is the most important model you use without knowing it

If you work on modern ML, you have already fitted a Bradley–Terry model this year. You just called it something else.

**Reward models.** RLHF collects pairwise human preferences and trains a reward `r(x, y)` with the loss

```
−log σ( r(x, y_win) − r(x, y_lose) )
```

That is Bradley–Terry, exactly, with "players" = responses and "games" = human judgments. Every failure mode in §2 and §4 transfers verbatim: judge intransitivity violates Wish 3; the reward is only identified up to a shift; and a response never compared to anything gets its score purely from the chain rule.

**DPO.** The trick in Direct Preference Optimization is to notice that under a BT preference model, the optimal policy has a closed form, so you can substitute `r = β · log(π/π_ref)` and delete the reward model entirely:

```
L_DPO = −log σ( β·log π(y_w|x)/π_ref(y_w|x)  −  β·log π(y_l|x)/π_ref(y_l|x) )
```

> **DPO is Bradley–Terry with `θ := β × (log-probability ratio)`.** The BT assumption is load-bearing, not decorative: if your preference data is substantially intransitive — and human preferences between LLM outputs often are — then DPO is confidently optimising the wrong objective, and the failure will look like overfitting rather than like a modelling error.

**Chatbot Arena.** BT fitted over crowd-sourced pairwise votes, with bootstrap resampling for error bars. The empty-cell machinery from §5 is precisely what lets a model released last week be ranked against one retired last year after a few hundred votes. Those error bars are not decoration either — they're the uncertainty that plain Elo doesn't have, bolted back on.

**Elo's second life.** Online and iterative DPO, arena-style live leaderboards, self-play rating loops — all of them are the §6 story again, with `K` replaced by a learning rate and a KL coefficient. Same SGD, same bias–variance tradeoff, same conservation law if the update is zero-sum.

**And the Gumbel thread.** Bradley–Terry ↔ Plackett–Luce ↔ Gumbel-top-k sampling ↔ the reparameterisation tricks used all over generative modelling. One object, four costumes. Recognising it in the fourth costume is worth a lot.

---

## 9. What the model cannot see

Worth listing plainly, because the failure modes are all silent:

- **Draws.** Standard BT has no tie outcome. Davidson (1970) and Rao–Kupper (1967) add a tie parameter and fix it.
- **Intransitivity.** BT cannot represent a cycle. Feed it the perfect cycle A≻B≻C≻A with one win each and the MLE puts all three players at *exactly the same strength* — the loop is averaged into a three-way tie, and nothing in the fit tells you the data was cyclic. Test for cycles before you trust a fit; BT will otherwise report a confident linear order that does not exist.
- **Context.** Home advantage, first-move advantage, position bias in side-by-side LLM evaluation, prompt difficulty. The standard fix is BT with covariates, `θ_i = Σ_k x_ik β_k` (Springall, 1973), plus terms for the situational factors. Once you write it this way you'll notice it's a logistic regression, because it is one.
- **Uncertainty.** Get error bars from the observed Fisher information — the Hessian of `L`, which you already have — or just bootstrap the games.
- **Drift.** BT assumes the truth is frozen. Elo tracks it; Glicko models it explicitly. Pick based on whether your players are chess players or language models.

---

## So: could Tom beat Jerry?

Suppose Tom never wins a single episode. Take `A = {Jerry}`, `B = {Tom}` in Ford's condition: nobody in `B` ever beat anyone in `A`, so the condition fails, the likelihood has no finite maximum, and `θ_Jerry − θ_Tom` runs off to `+∞`.

That is the right answer, and it is the answer a naive win-rate table would never give you. The data does not contain a bounded comparison. It contains a half-ordering and an unbounded gap.

Now give Tom exactly one win, out of 160 episodes. The two-player MLE is `σ(θ̂_Tom − θ̂_Jerry) = 1/160`, so

```
θ̂_Jerry − θ̂_Tom = ln 159 ≈ 5.07
P(Tom wins the next one) = 1/160 ≈ 0.0063
```

A single observation converts an unanswerable question into a finite one. **That is what a rating system is**: not a measurement of strength, but a machine for converting absence of evidence into a number — at a price you are entitled to inspect.

The price is Wish 3. Bradley–Terry and Elo will always tell you how far apart Tom and Jerry are. Neither will ever admit that they were measured on different rulers.
