# SAP IBP — The Integrated Plan, v2 (Rigorous)

*A corrected, model-faithful rebuild of the integrated worked example, written to be **learned
from** and **applied at a company**. Every section has four layers: the **rigorous model**, an
**intuition** box (what it means), an **in practice** box (who owns it, what data it eats, which
KPI it moves, how it breaks), and **labeled assumptions** with failure modes. Where v1 was a
teaching scaffold that quietly conflated single-echelon math with MEIO and used EOQ where
dynamic lot-sizing belongs, v2 uses the actual models and says exactly where each one stops
being valid.*

> **What changed from v1 — the correction register**
> 1. v1's per-node $z\sigma\sqrt{L+R}$ is **single-echelon safety stock, not MEIO.** v2 does real
>    multi-echelon **placement optimization** (Graves–Willems guaranteed-service; Clark–Scarf
>    echelon stock) — Part 7.
> 2. **EOQ for lumpy dependent demand was wrong.** v2 uses **Wagner–Whitin / Silver–Meal**
>    dynamic lot-sizing — Part 8.
> 3. v1's three capacity heuristics (lot sizing, pre-build, schedule) become **one CLSP/CLSD
>    MILP** where pre-build emerges endogenously — Part 9.
> 4. **Service level was asserted (95%).** v2 sets it by **newsvendor critical fractile** from
>    real understock/overstock costs — Part 4.
> 5. **Pooling assumed independence and overstated the benefit.** v2 adds the **correlation
>    correction** — Part 6.
> 6. **Plant cycle stock used $Q/2$;** the finite-rate **EPQ maximum is $Q(1-d/p)/2$** — Part 8.
> 7. **Bullwhip was hand-waved.** v2 uses the **Lee–Padmanabhan–Whang** amplification formula —
>    Part 12.
> 8. **MAPE→σ via 1.25 is a normal-error approximation** invalid for intermittent demand — v2
>    derives it and gives the **Croston** alternative — Part 2.

---

## Part 0 — The mental model (read this first)

### The three questions every inventory system answers
1. **How much buffer, and where?** → safety stock + its *placement* across echelons (Parts 4–7)
2. **How big a batch?** → lot sizing (Parts 8–9)
3. **When to order / make?** → reorder points and the production plan (Parts 9–10)

Everything else — forecasting, capacity, the schedule, the dynamics, the financials — exists to
feed or execute those three answers.

### The two paradigms you must not mix
Multi-echelon inventory theory splits into two families, and a practitioner must always know
which one a tool/screen is using:

- **Guaranteed-service (GSM — Graves & Willems, 2000).** Demand is *bounded*; each stage quotes a
  deterministic **service time** to its customer; within the bound there are *no* stockouts.
  Tractable on large general networks → this is what IBP-style MEIO engines lean on. *Output: where
  to place decoupling buffers.*
- **Stochastic-service (Clark & Scarf, 1960).** Demand is probabilistic; stockouts occur with some
  probability; optimal **echelon base-stock** levels are found by stage-by-stage decomposition.
  Exact for serial systems, conceptually foundational. *Output: echelon order-up-to levels.*

v1 was implicitly neither — it independently buffered every node for its own lead time, which
**double-counts** protection. v2 (Part 7) does GSM placement and explains the Clark–Scarf
echelon view.

### How this maps to a real company (the "who does what")
This is the framing that matters most for application — no single person does all of it:

| Layer | Team that owns it | Tool screen | Inputs they own | KPI |
|---|---|---|---|---|
| Forecast + sensing | **Demand Planning** | IBP Demand | history cleansing, models, consensus | forecast accuracy, bias |
| Buffer sizing & **placement** | **Inventory / Network Opt** | IBP Inventory (MEIO) | lead-time variability, cost gradient, service targets | inventory $, service level |
| Lot sizing, MRP, master schedule | **Supply / Production Planning** | IBP Supply, S/4 MRP | BOM, capacity, setup costs, lot rules | utilization, schedule adherence |
| Reconciliation, tradeoffs | **S&OP / IBP process owner** | IBP for S&OP | scenarios, financial assumptions | margin, turns, GMROI |
| Execution, deviation | **Control Tower / order mgmt** | IBP Response / CT | live orders, alerts | OTIF, expedite cost |

> **The practitioner's actual job.** The tool automates the *arithmetic* (it will happily compute
> $z\sigma\sqrt{\tau}$, run the MILP, explode the BOM). It does **not** supply judgment. Your job
> is to get the **inputs** right (lead-time *variability*, cost ratios, service economics, clean
> history), understand **which input moves which output and in which direction**, and know **which
> model is valid for which item** (Part-14 playbook + the appendix decision table). Garbage
> $\sigma_L$ in → confidently wrong buffer placement out.

---

## Part 1 — The dataset (costed, capacitated, and honest about assumptions)

Same physical network as v1, carried forward so the corrections are visible against a known case.

### Network
`Suppliers → Plant → CDC → {RDC A → Stores 1,2 ; RDC B → Stores 3,4}`. One shared production line
makes **Product A** and **Product B**; both flow to all four stores. (The *shared-line* assumption
is load-bearing for Parts 8–9; the **separate-line** case is handled explicitly in Part 9.)

### Demand (per day) and forecast error
| | S1 | S2 | S3 | S4 | Σ |
|---|--|--|--|--|--|
| **A** demand / MAPE | 20 / 20% | 30 / 25% | 24 / 15% | 16 / 30% | 90 |
| **B** demand / MAPE | 12 / 18% | 18 / 22% | 14 / 20% | 10 / 28% | 54 |

### Costs (the architecture that drives everything downstream)
Part costs P1–P7: ₹50 / 80 / 30 / 120 / 60 / 90 / 45. FG cost (material + conversion): **A ₹800**,
**B ₹400**. Value gradient per echelon — A: plant 800 → CDC 850 → RDC 900 → store 950; B: 400 →
430 → 460 → 490. Carrying rate **25%/yr** → holding $h = 0.25\times$ value. Ordering: PO ₹500;
**changeover ₹10,000**; shipment ₹4,000 / 2,500 / 1,200 per truck (800/400/120). Sell price A
₹1,200, B ₹650. **Understock cost** (lost margin) A ₹250, B ₹160 — used by the newsvendor in
Part 4, *not* an arbitrary 95%.

### Lead times (mean, std — days)
Suppliers Sup1–7: (7,2)(10,3)(5,1)(14,4)(8,2)(12,3)(6,1.5). Mfg: A (2,0.4), B (1.5,0.3). Transit:
Plant→CDC (3,0.5); CDC→RDC A (2,0.4), →RDC B (2.5,0.5); RDC→stores (1,0.2)(1.5,0.3)(1,0.2)(2,0.4).

### Capacity — now multi-resource (v1 only checked machine-hours)
One line, **16 machine-hours/day**, rates A 10/hr, B 20/hr. **Add labor**: assembly needs **2
operators** while A runs, **1** while B runs; labor availability **32 operator-hours/day** (2
shifts × ... ). Steady machine load 12.03 h/day (75%); labor and machine are *separate* capacity
rows in the Part-9 MILP — a plan feasible on machines can still be infeasible on labor, which v1
could not even express.

### Assumptions register (carried and flagged throughout)
| # | Assumption | Where it bites | Failure mode |
|---|---|---|---|
| A1 | Forecast errors **normal, unbiased** | every safety stock | skew/intermittency → wrong $z$, wrong buffer |
| A2 | Store demands **independent** (pooling) | Part 6 | positive correlation → pooling benefit overstated |
| A3 | Demand & lead time **independent**, no cross-terms in $\sigma_{DDLT}$ | Parts 5,7 | correlated LT/demand → understated buffer |
| A4 | Holding rate **flat 25%** | all $ figures | capital≠obsolescence≠space; differs by node/SKU |
| A5 | Lead times **deterministic** in base GSM | Part 7 | $\sigma_L$ ignored → under-buffer; needs extension |
| A6 | Demand is a **flat rate** | Parts 3,8 | real schedules are time-varying → EOQ invalid |
| A7 | **Shared** production line | Parts 8,9 | dedicated lines decouple the problem |

These aren't disclaimers to wave away — each one tells you *when to reach for a different model*,
which is the entire point of the appendix decision table.

---

## Part 2 — Forecast error, done right (the 1.25, and when it lies)

Safety stock is $z\times$(a standard deviation). That standard deviation is the single most
abused input in practice, so get it exactly right.

### Where 1.25 comes from
For an **unbiased normal** error $X\sim\mathcal N(0,\sigma^2)$, the mean absolute deviation is
$\mathrm{MAD}=\mathbb E|X|=\sigma\sqrt{2/\pi}$, hence

$$\sigma=\mathrm{MAD}\cdot\sqrt{\pi/2}=1.2533\cdot\mathrm{MAD}.$$

Then the loose step: $\mathrm{MAD}\approx \mathrm{MAPE}\cdot\mu$ (mean absolute error ≈ percentage
error × mean demand), giving the rule of thumb $\sigma_d\approx 1.25\cdot\mathrm{MAPE}\cdot\mu_d$.
This constant scales **every buffer in the network**, so its assumptions matter:

> **Assumption A1 in detail.** The 1.2533 is exact *only* for a normal, unbiased error. And
> $\mathrm{MAD}=\mathrm{MAPE}\cdot\mu$ holds only when demand is roughly level — MAPE is the mean of
> $|e_t/A_t|$, not $|e_t|/\bar A$; the two diverge when demand varies or is small, and MAPE itself
> is asymmetric (penalizes over-forecast less) and explodes as $A_t\to0$.

### What to use instead, in practice
- Prefer measuring $\sigma$ **directly from the forecast-vs-actual error series** on a holdout
  (its sample standard deviation), or use **RMSE**, which already estimates $\sigma$ for an unbiased
  error — no 1.25 needed and no normality-of-MAD assumption.
- Separate **error from bias.** Error sizes the buffer; **bias is a defect** — a persistent
  over/under-forecast doesn't need more safety stock, it needs the forecast fixed. Track it with
  the **tracking signal** $\mathrm{TS}=\frac{\sum e_t}{\mathrm{MAD}}$; $|\mathrm{TS}|>4$ ⇒ the model
  is broken, not noisy.
- For **intermittent / lumpy** demand (exactly the dependent part demand downstream of campaigns),
  the normal-MAD machinery is **invalid**. Use **Croston's method** (decompose into demand *size*
  and *inter-arrival interval*, forecast each) or **Syntetos–Boylan (SBA)**; the spread is then
  driven by interval variability, not a MAPE.

> **Intuition.** "1.25 × MAPE × mean" is a *fallback* for when all you're handed is a MAPE on a
> stable, normal SKU. The moment demand is spiky, seasonal, skewed, or you have the raw error
> series, throw it away and measure $\sigma$ (or model it) properly.

> **In practice — Demand Planning owns this.** Inputs: cleansed history (stockout-corrected,
> outlier-treated, promo-separated — the most-skipped step). KPI: forecast accuracy *and* bias.
> The number this team produces ($\sigma_d$ per SKU-location) is the seed for the entire inventory
> plan — a 20% error in $\sigma_d$ is a ~20% error in every safety stock it feeds. **Forecast Value
> Added (FVA)** is the discipline that checks each forecasting step (naïve → statistical →
> consensus override) actually *reduces* error rather than adding human noise.

---

## Part 3 — Time-phasing and granularity (the rate-to-bucket question, answered)

A demand **rate** (90/day) is an expected value. Planning needs **time-phased buckets**: a
requirement per period. Two things v1 left implicit:

**Pick the bucket to match the decision cadence and the lead times — not by habit.** If your
shortest lead times and your replenishment review are measured in days, plan in **days**; if the
production wheel turns weekly, the master schedule is **weekly**; S&OP reconciles **monthly**. A
mature tool holds **one number set at the finest grain and aggregates up** — daily demand sensing,
weekly supply, monthly S&OP — never three disconnected forecasts.

**If you have a true daily demand schedule (not a flat rate), you don't need the rate abstraction
at all.** You net daily, and — critically — lot sizing becomes **daily dynamic lot-sizing**
(Wagner–Whitin over days, Part 8), *not* EOQ. The flat-rate assumption (A6) is precisely what lets
EOQ sneak in; with a real time-varying schedule, EOQ is structurally wrong because it ignores
*when* demand lands.

> **Why dependent demand is lumpy (and why it matters).** Finished-goods demand may be smooth, but
> the moment you batch production into campaigns, the *components'* demand becomes spiky — zero
> between campaigns, a large pulse at each campaign. So the part-level forecast is intermittent
> (→ Croston, Part 2), and the part-level lot size must be dynamic (→ Wagner–Whitin, Part 8). v1's
> EOQ-on-parts contradicted its own campaign model.

> **In practice — the granularity trap.** Teams forecast monthly because that's how the business
> talks, then try to drive daily replenishment off it and wonder why service is erratic. The fix
> is *temporal disaggregation* (spread the monthly number to a daily profile) plus **demand
> sensing** for the near horizon — the short-term signal that catches a spike within the cycle
> rather than a month late.

---

## Part 4 — Service level by economics: the newsvendor (not "95% because")

v1 asserted 95% / $z=1.645$. The economically correct target comes from the **newsvendor critical
fractile** — balance the cost of being short against the cost of holding too much.

### The model
With underage (stockout) cost $C_u$ per unit and overage (excess-holding) cost $C_o$ per unit, the
optimal in-stock probability (cycle service level) is the **critical ratio**:

$$\mathrm{CSL}^\*=\frac{C_u}{C_u+C_o},\qquad z^\*=\Phi^{-1}(\mathrm{CSL}^\*),\qquad SS=z^\*\,\sigma_{DDLT}.$$

### Worked, Product A at Store 2
$C_u$ = lost margin if short = **₹250** (from the cost sheet). $C_o$ = cost of carrying one *surplus*
unit until it sells down. If excess bleeds off in ~2 weeks, $C_o\approx 237.5\times\frac{14}{365}\approx ₹9$:

$$\mathrm{CSL}^\*=\frac{250}{250+9}=0.965\approx \mathbf{96.5\%},\qquad z^\*=\Phi^{-1}(0.965)=1.81.$$

So the economics justify ~96–97%, and the buffer is $SS=1.81\times\sigma_{DDLT}$ — not 1.645×.
For Store 2 ($\sigma_{DDLT}\approx20.8$) that's **≈ 38 units**, vs v1's 34. The number was never
"95%"; it falls out of the margin and the holding/obsolescence horizon.

### The point: differentiate, don't globalize
$\mathrm{CSL}^\*$ is **per item**, and it swings hard with the cost ratio:

| Item profile | $C_u$ | $C_o$ | $\mathrm{CSL}^\*$ | $z^\*$ |
|---|--|--|--|--|
| High-margin, cheap to hold (flagship A) | 250 | 9 | 96.5% | 1.81 |
| Same margin, **obsolescence-prone** (fashion/tech) | 250 | 120 | 67.6% | 0.46 |
| **Low-margin** commodity | 40 | 9 | 81.6% | 0.90 |
| Critical spare (stockout halts a line) | 5,000 | 50 | 99.0% | 2.33 |

> **Intuition.** A unit that's cheap to hold and painful to be without (high margin, no
> obsolescence) *wants* near-100% service. A unit that rots or marks down (high $C_o$) wants a
> *low* target — better to stock out occasionally than eat write-offs. One global 95% over-serves
> the second and under-serves the first.

> **In practice — Inventory team owns this, with Finance.** $C_u$ comes from gross margin and the
> cost of a lost sale/expedite; $C_o$ from holding **plus** obsolescence/markdown risk — the part
> people forget. Segment by **ABC** (value) × **XYZ** (variability): set high CSL on high-value
> stable items, accept lower CSL on volatile/obsolescent ones. Note CSL (in-stock *probability*)
> differs from **fill rate** (fraction of *units* served); fill-rate targets use the standard-normal
> **loss function** $G(z)$, not just $\Phi(z)$ — most retail SLAs are fill-rate.

---

## Part 5 — The single-location buffer (the primitive Part 7 will *place*)

Before placing stock across echelons, get the single-stage buffer right. Over a replenishment
exposure of $L$ (lead time) plus $R$ (review period), with demand error $\sigma_d$/day and
lead-time std $\sigma_L$:

$$\sigma_{DDLT}=\sqrt{\underbrace{(L+R)\,\sigma_d^2}_{\text{demand risk}}+\underbrace{\mu_d^2\,\sigma_L^2}_{\text{lead-time risk}}},\qquad SS=z^\*\,\sigma_{DDLT}.$$

The $\mu_d^2\sigma_L^2$ term is why an unreliable supplier, not demand noise, dominates the buffer
for fast-moving or high-volume parts (it squares the *mean throughput*) — the headline of the
original MEIO problem.

> **Assumptions A3 + A5.** This combines demand and lead-time variance *assuming independence and
> no cross-terms* — the exact variance of a random sum has covariance terms this drops. And it
> treats demand-during-lead-time as **normal**; for slow movers, skewed, or intermittent items
> (where the mean is small and the distribution is right-skewed) the normal $z$ over-/under-states
> the tail — use a **gamma**, **negative-binomial**, or empirical distribution, or Croston-based
> methods. The periodic-review $(L+R)$ vs continuous-review $L$ distinction also changes the
> exposure window.

> **In practice.** The biggest lever here is almost never the forecast — it's **$\sigma_L$,
> supplier lead-time variability**, which most ERPs *don't even store* (they keep a single planned
> lead time). Pulling actual receipt-date variance from PO history and feeding it in is the
> highest-ROI data project most planning teams never do. This per-stage $\sigma_{DDLT}$ is the
> input the network optimizer in Part 7 consumes — but it does **not** simply sum these node-by-node
> (that's the v1 error); it *places* buffers to avoid covering the same risk twice.

---

## Part 6 — Risk pooling (the *why* of MEIO), with the correlation correction

The reason a central buffer beats dispersed ones: **variances add, standard deviations don't.**
Consolidating $N$ independent demand streams gives

$$\sigma_{\text{pooled}}=\sqrt{\textstyle\sum_k\sigma_k^2}\;<\;\sum_k\sigma_k.$$

For our four stores ($\sigma_d$ = 5.0, 9.375, 4.5, 6.0): $\sum\sigma_k=24.9$ vs
$\sqrt{\sum\sigma_k^2}=\sqrt{169.1}=13.0$ — a **48% smaller** spread to buffer, for the same
service. For $N$ *identical* stores it's the clean $\sigma\sqrt N$ vs $\sigma N$, i.e. pooling
saves a factor $1/\sqrt N$.

### The correction v1 ignored (Assumption A2)
Real store demands are **positively correlated** (shared seasonality, national promotions,
weather). With pairwise correlation $\rho$:

$$\sigma_{\text{pooled}}=\sqrt{\textstyle\sum_k\sigma_k^2+2\sum_{i<j}\rho_{ij}\,\sigma_i\sigma_j}.$$

At $\rho=0.3$ across our four stores, $\sigma_{\text{pooled}}=\sqrt{169.1+2(0.3)(224.8)}=\sqrt{304.0}=17.4$
— so the saving versus $\sum\sigma_k=24.9$ is **30%, not 48%.** And the cruelest part: demand is
*most* correlated exactly when it's highest — during the seasonal peak everyone spikes together,
so the pooling benefit you were counting on **evaporates when you need it most.**

> **Intuition.** Pooling is portfolio diversification for demand. Independent risks cancel
> (√N law); correlated risks don't diversify away. v1's 48% was the best case (zero correlation);
> the achievable number is lower and *worst at the peak.*

> **In practice.** This is the math behind **DC consolidation**, **inventory centralization**, and
> **postponement** (hold generic stock pooled upstream, customize late). Before promising the
> pooling saving to Finance, estimate the actual demand correlation from history — a centralization
> business case built on the independence assumption will over-promise.

---

## Part 7 — Multi-echelon optimization: the *real* MEIO (Clark–Scarf and Graves–Willems)

This is the section v1 got fundamentally wrong: it computed a single-stage buffer at every node
and summed them, which protects against the *same* demand risk multiple times. Real MEIO decides
**where** to hold stock so total cost is minimized while the customer service target is met.

### 7a. Clark–Scarf (1960) — the stochastic-service foundation
For a **serial** chain, Clark & Scarf proved that **echelon base-stock policies are optimal** and
solvable by **decomposition**. Two ideas you must internalize:

- **Echelon inventory** = stock at a stage **plus everything downstream of it** (not just local
  "installation" stock). Policies and costs are expressed in echelon terms.
- **Stage-by-stage decomposition:** solve the most-downstream stage as a newsvendor against
  customer demand; it passes an **induced shortage penalty** up to the next stage; that stage
  solves its own newsvendor including the induced penalty; repeat upstream.

> **Intuition.** Thinking in *echelon* stock is what stops double-counting. v1 asked "how much does
> each node need locally?" and added it up. Clark–Scarf asks "how much should this stage hold given
> everything already sitting downstream of it?" — which is why the right answer is *less* than the
> naïve sum.

### 7b. Graves–Willems (2000) — guaranteed-service, and what your solver implements
GSM scales to **general tree networks** (ours) by giving each stage a **service time**. Definitions:

- $T_j$ = stage $j$'s own processing/replenishment lead time.
- $S_j$ = outbound service time stage $j$ **promises its customer** (the decision variable).
- $SI_j$ = inbound service time stage $j$ is **promised by its supplier** ($=\max$ of suppliers'
  $S$ for an assembly; the upstream stage's $S$ in a serial/distribution link).
- **Net replenishment time** $\tau_j = SI_j + T_j - S_j$ — the window stage $j$ must self-buffer.
- Stage cost $=h_j\,z\,\sigma_j\sqrt{\tau_j}$.

**The program:** $\min\sum_j h_j z\sigma_j\sqrt{\tau_j}$ over the $S_j$, subject to the
customer-facing service time meeting target, and $0\le S_j\le SI_j+T_j$ (so $\tau_j\ge0$). A key
theorem: the objective is **concave** in the service times, so the optimal $S_j$ sit at **extreme
points** — each internal stage either **HOLDS** (quote $S_j=0$, buffer its full $\tau$) or **PASSES
THROUGH** (quote $S_j=SI_j+T_j$, hold *nothing*). On a tree this is solved exactly by dynamic
programming; on our small network we can enumerate.

### 7c. Worked placement — Product A
Stage data (demand $\sigma_d$ pooled appropriately at each tier; $z=1.645$; raw materials buffered
separately so $SI_{\text{Plant}}=0$; stores must serve customers instantly so $S=0$ is forced
there):

| Stage | $T$ (d) | $\sigma_d$ | $h$ (₹/u/yr) | cost coef $k=h z\sigma$ |
|---|--|--|--|--|
| Plant FG | 2 | 13.00 | 200 | 4,279 |
| CDC | 3 | 13.00 | 212.5 | 4,546 |
| RDC A | 2 | 10.63 | 225 | 3,933 |
| RDC B | 2.5 | 7.50 | 225 | 2,776 |
| Stores 1–4 | 1/1.5/1/2 | 5.0/9.4/4.5/6.0 | 237.5 | 1,953/3,663/1,758/2,344 |

Cost of a stage $=k\sqrt{\tau}$. Comparing placement policies (HOLD $S{=}0$ / PASS $S{=}SI{+}T$):

| Policy | Plant | CDC | RDC A | RDC B | Stores hold | **Total SS cost/yr** |
|---|--|--|--|--|--|--|
| **1. Buffer everywhere** (v1's philosophy) | HOLD | HOLD | HOLD | HOLD | transit only | **₹35,400** |
| **2. Pool at CDC**, flow-through else | PASS | HOLD | PASS | PASS | ~3–4.5 d | **₹28,700** |
| **3. Push to stores**, no upstream buffer | PASS | PASS | PASS | PASS | ~8–9.5 d | **₹28,600** |
| 4. Hold at RDCs + stores | PASS | PASS | HOLD | HOLD | transit only | ₹29,500 |

**The optimum (~₹28,600) is ~19% below buffer-everywhere (₹35,400)** — pure waste eliminated by
*not* protecting the same risk at every echelon. Notice Policies 2 and 3 nearly **tie**: with our
*shallow* 19% holding-cost gradient (₹200→₹237.5), pooling at the CDC barely beats concentrating at
the stores. **Make the gradient steeper** — say expensive retail floor space pushes store holding
to ₹400/yr — and CDC-pooling wins decisively; flatten it and the optimizer is indifferent to
location and just cares that you don't double-buffer.

> **Intuition — the three forces.** Where the optimizer puts stock is a tug-of-war between
> (i) **pooling** (favors upstream, where demand aggregates), (ii) the **holding-cost gradient**
> (favors upstream, where units are cheap), and (iii) the **service-time requirement** (favors
> downstream, close to the customer). The decoupling point is wherever those balance. v1 ignored
> all three by buffering everywhere.

> **In practice — this is your GSM solver, and IBP's MEIO engine.** The inputs that actually move
> the answer: lead-time variability ($\sigma_L$), the holding-cost gradient, and the per-stage
> service targets (from Part 4's newsvendor, *per item*). Pitfalls: (1) feeding deterministic lead
> times (Assumption A5) — base GSM ignores $\sigma_L$, so you bolt on a stochastic-service
> extension or inflate $\sigma$; (2) trusting the placement when $\sigma_d$ or $\sigma_L$ is
> garbage; (3) forgetting that GSM assumes *bounded* demand and **no stockouts within the bound** —
> if you need actual stockout probabilities or have heavy-tailed demand, you're in stochastic-service
> territory (Clark–Scarf base-stock or simulation), not GSM. Run MEIO **after** the demand and
> lead-time data are clean, never before.

---

## Part 8 — Lot sizing, done right (EOQ where it's valid, dynamic where it isn't)

### When EOQ is legitimate
$Q^\*=\sqrt{2DS/h}$ is correct **only** for demand that is **constant, continuous, and
deterministic** (Assumption A6). That describes a few high-volume, stable, independent finished
goods — not much else, and *definitely not* dependent component demand, which is lumpy by
construction (Part 3). v1's use of EOQ for the parts contradicted its own campaign model.

### Lumpy / time-varying demand → dynamic lot-sizing
**Wagner–Whitin (1958)** gives the **exact cost-minimizing** lot plan via dynamic programming,
exploiting the **zero-inventory-ordering property** (an optimal plan only orders in a period when
entering inventory is zero). **Silver–Meal** is the standard heuristic: extend the current order to
cover more future periods while the **average cost per period** keeps falling; stop when it rises.
(Least-Unit-Cost and Part-Period-Balancing are cousins.)

**Worked — a lumpy part stream** (weekly demand, $S=₹500$, $h=₹0.288$/unit/wk):

| Week | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|--|--|--|--|--|--|--|--|
| Demand | 6,000 | 0 | 0 | 4,000 | 0 | 7,000 | 0 | 0 |

*Silver–Meal at week 1:* cover wk1 → 500/1 = ₹500/period; extend over the empty wks 2–3 →
(500+0)/3 = ₹167/period (falling, because empty periods add no holding); extend to wk4 → carrying
4,000 for 3 wks = ₹3,456, so (500+3,456)/4 = ₹989/period (jumps) → **stop**. Order **6,000 at wk1**.
Repeat → order **4,000 at wk4**, **7,000 at wk6**. The plan is order-per-campaign: 3 setups,
≈ zero carrying across gaps.

*EOQ on the same demand:* $Q^\*=\sqrt{2(110{,}500)(500)/15}\approx 2{,}714$ — a **fixed** quantity
that fits *none* of the actual requirements. Place it mechanically and you either stock out at wk1
(needs 6,000) or carry stock through the empty weeks you don't need it. **EOQ has no concept of
*when*** — that's the structural failure on lumpy demand. (Wagner–Whitin agrees with Silver–Meal
here; if setups were far costlier, WW would *combine* campaigns into fewer large orders, trading
holding for setups — it adapts to the $S/h$ ratio, EOQ can't.)

### The EPQ correction (v1's inflated plant inventory)
With a **finite production rate** $p$ and demand $d$, inventory builds and depletes
simultaneously, so the **maximum** on-hand is $Q(1-d/p)$ and average cycle stock is

$$\bar I_{\text{cycle}}=\frac{Q(1-d/p)}{2}\quad(\text{not } Q/2).$$

For Product A, $d/p=90/160=0.5625$, so cycle stock $=3{,}093\times0.4375/2=\mathbf{677}$ units —
**not 1,547.** v1 overstated plant FG cycle stock by **2.3×**, which alone pulls the headline
network inventory well below v1's ₹4.98M. *A finite production rate is a free reduction in cycle
stock that the simple $Q/2$ throws away.*

### Coordinating lots across echelons
Independent per-node EOQ ignores that stages share flows. **Roundy's power-of-two policies**
restrict every stage's reorder interval to a power-of-two multiple of a base period, producing
**nested / integer-ratio** replenishment that is provably within **~2%** (6% worst case) of the
optimal coordinated solution — the lot-sizing analogue to MEIO's buffer coordination.

> **In practice.** In the ERP, this is the **lot-sizing rule** per material: `lot-for-lot` (chase
> demand, min inventory, max setups — good for expensive/lumpy A items), `fixed-period` (cover N
> periods — simple, decent for steady items), `EOQ/POQ` (stable demand only), or `Wagner–Whitin`
> (where the system offers it, for lumpy items where it pays). Layer on **MOQ** and rounding values
> (pallet/case). The rule should match the item's demand pattern — *not* one default applied to the
> whole catalog.

---

## Part 9 — The integrated supply model: one CLSP/CLSD MILP (replacing v1's three heuristics)

v1 solved lot sizing (Part 4), pre-build (Part 8), and the schedule (Part 9) as three disconnected
hand-waves, and checked only machine-hours. They are **one problem** — the **Capacitated
Lot-Sizing Problem** — and it's what an APS supply optimizer actually solves.

### Formulation
Indices: products $p$, periods $t$, **resources** $r\in\{\text{machine, labor},\dots\}$. Parameters:
demand $d_{p,t}$; holding $h_p$; setup cost $s_p$; unit resource use $a_{p,r}$; setup time $b_{p,r}$;
capacity $C_{r,t}$; overtime cost/limit $o_{r,t},\bar O_{r,t}$; lost-sales penalty $\ell_p$.
Decisions: $x_{p,t}\ge0$ (production), $y_{p,t}\in\{0,1\}$ (setup), $I_{p,t}\ge0$ (inventory),
$O_{r,t}\ge0$ (overtime), $L_{p,t}\ge0$ (lost sales).

$$\min\ \sum_{p,t}\big(s_p y_{p,t}+h_p I_{p,t}+\ell_p L_{p,t}\big)+\sum_{r,t}o_{r,t}O_{r,t}$$

subject to

$$
\begin{aligned}
&\textbf{inventory balance:} & I_{p,t}&=I_{p,t-1}+x_{p,t}-d_{p,t}+L_{p,t}\\
&\textbf{multi-resource capacity:} & \sum_p\big(a_{p,r}x_{p,t}+b_{p,r}y_{p,t}\big)&\le C_{r,t}+O_{r,t} &&\forall r,t\\
&\textbf{setup forcing:} & x_{p,t}&\le M\,y_{p,t}\\
&\textbf{overtime / safety floor:} & O_{r,t}\le\bar O_{r,t},\quad I_{p,t}&\ge SS_p\\
&\textbf{domains:} & x,I,O,L&\ge0,\ y\in\{0,1\}.
\end{aligned}
$$

### Why this is the whole point
**Pre-build, overtime, and lost sales are not separate decisions — they are outcomes of this one
program.** When a future period's capacity is tight, the optimizer raises $I_{p,t}$ in an earlier
period *iff* carrying cost beats overtime and lost margin — the seasonal pre-build **emerges
endogenously**, with no separate heuristic. The machine **and** labor rows are both present, so a
plan feasible on machines but infeasible on labor is correctly rejected — something v1's
single-resource check could not even represent.

- **Shared line** (our case): both products consume the *same* machine-resource row → they compete,
  and the setup binaries create the changeover tradeoff. The equal-interval **ELSP common cycle**
  from v1 is just a *heuristic special case* of this (impose a common period); ELSP is NP-hard, and
  the independent single-product solution only gives a **lower bound**.
- **Separate lines:** map each product to its **own** resource row → the capacity constraints
  decouple and the problem splits into independent per-line CLSPs. (This is why v1's whole
  shared-line ELSP story was inapplicable to a dedicated-line shop — Assumption A7.)
- **CLSD** extends this with **setup carryover** across periods (don't pay the setup again if you're
  already configured for that product), and adding sequence variables turns it into lot-sizing-
  *and*-scheduling for the Part-10 production sequence.

> **In practice — Supply Planning owns this; it *is* the optimizer run.** Configuration: which
> costs are real (setup, holding, overtime, penalty), capacity calendars per resource, min lot /
> rounding. Reality checks: large instances need decomposition/heuristics (the MILP is NP-hard, so
> the engine time-boxes it); planners **override** because the model only knows the constraints
> you *told* it — unmodeled tooling, quality holds, supplier allocations. The skill is reading the
> optimizer's plan, spotting where its cost assumptions diverge from reality, and adjusting the
> *inputs* rather than hand-editing the output.

---

## Part 10 — Reorder points and policies (falling out of the placement)

Reorder logic only lives at stages the GSM solution says **HOLD**; flow-through stages have no
reorder point — they pass demand straight up. At a holding stage:

- **Continuous review $(s,Q)$:** order $Q$ when inventory **position** hits $s=\mu_d L + SS$.
- **Periodic review $(R,S)$:** every $R$, order up to $S=\mu_d(L+R)+SS$. The extra $\mu_d R$ is the
  cost of waiting for the next review — why periodic needs more than continuous.
- **$(s,S)$:** hybrid — at review, if position $\le s$, order up to $S$.

The $SS$ in these formulas is now the **placed** safety stock from Part 7 (and sized with the
**newsvendor $z^\*$** from Part 4), not an independently-computed per-node buffer. So the policy at
each node is a *consequence* of the network design, not an independent decision — that's the
through-line v1 missed.

> **In practice.** In the system these are auto-maintained per SKU-location and recomputed as
> forecast/lead-time inputs drift. The planner works **by exception** (Control Tower alerts), not by
> touching every reorder point — there are tens of thousands of them.

---

## Part 11 — The dynamics, corrected (one demand convention, a justified tail)

v1's simulation muddled backorders and lost sales. Fix the convention: **retail = lost sales**
(unmet demand walks). Simulating one holding stage (Store 2 A, $z^\*$=1.81 → $SS\approx38$, order-up-to
$S$, lead time $L$), the system shows four behaviors:

1. **Normal sawtooth** — on-hand cycles between a near-$s$ low and $S$; safety stock is *brushed*
   on the unlucky-but-routine cycles. That's the buffer **working**, not failing.
2. **A demand spike pierces the buffer — by design.** At a 96.5% service level, ~3.5% of cycles
   stock out; a $+4\sigma$ day is exactly that tail. The lost units are the *economically chosen*
   shortage (Part 4), not a planning error. Raising the buffer to prevent it costs more than the
   ₹250 it saves — the newsvendor already made that call.
3. **The next reorder self-corrects.** Because the order-up-to fires from a lower position, the
   replenishment after a spike is automatically larger — the policy pulls more without anyone
   intervening.
4. **Transient vs level shift.** A one-off spike → rebuild to the *same* $S$. A *sustained* lift →
   leaving $S$ alone causes chronic stockouts; **demand sensing** must raise $\mu$, which recomputes
   $\sigma_d, SS, s, S$ **upward**. The reorder point is only as good as the forecast — closing the
   loop back to Part 2.

---

## Part 12 — Bullwhip, quantified (Lee–Padmanabhan–Whang, not hand-waved)

v1 asserted "CV 31% → 45%." The amplification is **derivable**. For an order-up-to policy with a
$p$-period moving-average forecast and lead time $L$:

$$\frac{\mathrm{Var(orders)}}{\mathrm{Var(demand)}}\;\ge\;1+\frac{2L}{p}+\frac{2L^2}{p^2}.$$

**Store → RDC** ($L=2$ days, $p=5$-day MA): $1+\tfrac{4}{5}+\tfrac{8}{25}=2.12$ → order std is
$\sqrt{2.12}=1.46\times$ demand std, so store demand CV **31% → order CV 45%** (the hand-waved
number, now earned). **RDC → CDC** applies the same amplification to that 45% input → CDC sees
CV **≈ 66%**. Variance compounds at each echelon — which is precisely why the *upstream* buffers and
the plant's capacity headroom (Part 9) must absorb a far lumpier signal than the shelf ever sees.

The **four causes** (Lee et al.): demand-signal processing (re-forecasting off orders), **order
batching** (our review periods/FTL), **price fluctuation** (promotions → forward-buying), and
**rationing/shortage gaming**. Mitigations map one-to-one: share POS data upstream (kill the
re-forecasting), smaller/more frequent batches via EDI (our FTL is a batching source), EDLP
(kill forward-buying), allocate by historical share not orders (kill gaming).

> **In practice.** Bullwhip is why your factory sees swings the customer never created. The single
> highest-leverage fix is **demand visibility** — give upstream stages the *actual* end-demand
> signal (POS / sell-through), so they stop amplifying their own customers' orders. This is the
> business case for VMI and CPFR.

---

## Part 13 — S&OP, financials, and the closed loop (corrected numbers)

### Financials, with the EPQ fix flowing through
Correcting plant cycle stock ($Q(1-d/p)/2$, Part 8) pulls network inventory from v1's overstated
**₹4.98M to ≈ ₹4.1M**. Re-deriving the KPIs:

| Metric | Value |
|---|--|
| Annual revenue | ₹52.2M |
| COGS | ₹34.2M |
| Gross margin | ₹18.1M (34.6%) |
| **Avg inventory (corrected)** | **≈ ₹4.1M** |
| **Inventory turns** | COGS/inv ≈ **8.3×** (was 6.9) |
| **GMROI** | margin/inv ≈ **4.4** (was 3.6) |

The correction matters: a 2.3× error in one cycle-stock formula moved turns by ~20% — exactly the
kind of silent modeling error that makes a plan look worse (or better) than it is.

### The cycle and the loop
The monthly S&OP run reconciles demand, supply, and finance; **scenarios are MILP re-solves** of
Part 9 (pre-build vs overtime vs demand-shaping), not separate spreadsheets — same model, different
costs/constraints, compared on margin. The **Control Tower** then watches execution and routes
exceptions back to the owning team: capacity overload → Supply; projected stockout → Response
(expedite); $|\mathrm{TS}|>4$ → Demand (re-forecast → recompute buffers); single-source $P5$ risk →
Sourcing. Detect → route → re-plan: the loop, closed.

> **In practice — RACI.** Demand Planning is *responsible* for the forecast, Supply for the plan,
> Inventory for the buffers, S&OP-owner *accountable* for the reconciled number, Finance *consulted*
> on cost assumptions, executives *informed/deciding* on the tradeoffs surfaced. The process fails
> when these blur — e.g., when Supply quietly pads safety stock to cover a forecast it doesn't trust,
> double-buffering the network and hiding the real problem (a bad forecast) from the people who own
> it.

---

## Part 14 — The practitioner's playbook (how to actually run this at a company)

This is the part that turns theory into a job. Everything above is sequenced here into what you'd
actually *do*, in order.

### The sequence of work
1. **Fix the data first.** Master data (BOMs, routings, planned lead times), then the data nobody
   maintains: **lead-time variability** from PO receipt history (Part 5's biggest lever), **yield/
   scrap** from MES, **costs and margins** from Finance, **clean demand history** (stockout-corrected,
   outlier-treated, promo-separated) from POS/sales. Garbage here invalidates everything downstream.
2. **Forecast + measure error honestly** (Part 2). Statistical baseline → consensus, gated by FVA.
   Output $\mu_d$ and a *measured* $\sigma_d$ per SKU-location; separate **bias** from error.
3. **Set service targets by economics** (Part 4), per ABC×XYZ segment — not one global 95%.
4. **Optimize buffer *placement*** (Part 7 MEIO), not per-node buffers. Inputs that move it:
   $\sigma_L$, the cost gradient, the service targets.
5. **Choose lot rules per item** (Part 8): lot-for-lot/Wagner–Whitin for lumpy A items, fixed-period/
   EOQ only for stable ones; add MOQ/rounding.
6. **Run the constrained supply plan** (Part 9 CLSP), multi-resource; let pre-build/overtime/lost-
   sales fall out of the optimizer.
7. **Set reorder policies** (Part 10) from the placed buffers; auto-maintain, work by exception.
8. **Execute and monitor** (Control Tower); sense demand; expedite/re-deploy on deviation.
9. **Reconcile monthly in S&OP** (Part 13); compare scenarios as model re-solves; decide tradeoffs.

### Where the data lives
| Need | Source system |
|---|---|
| BOM, routing, planned lead time, costs | ERP (S/4HANA) master data |
| **Actual** lead-time variability $\sigma_L$ | PO receipt-date history (warehouse/procurement) |
| Capacity, run rates, yield, downtime | MES / shop floor |
| Margin, holding rate, obsolescence | Finance |
| End demand, stockouts, promotions | POS / order management / trade calendar |

### The 80/20 — where focus actually pays
Most of the value is in **four** places, all upstream of the clever math: stockout-corrected demand
history; **lead-time-variability data** (the input most ERPs don't even store); **cost ratios**
(margin, holding, setup, penalty — they set service levels *and* lot sizes); and **placement over
per-node buffering**. Get those right and a mediocre optimizer does well; get them wrong and the
best optimizer is confidently wrong.

### The mistakes checklist (literally the v1 errors — don't do these)
- ❌ EOQ on lumpy/dependent demand → use Wagner–Whitin/Silver–Meal.
- ❌ Buffer at every echelon → optimize placement (GSM); thinking in **echelon stock** (Clark–Scarf)
  prevents double-counting.
- ❌ Assert a global 95% → derive $\mathrm{CSL}^\*$ from $C_u/(C_u+C_o)$, per segment.
- ❌ $Q/2$ cycle stock for in-house production → use the EPQ max $Q(1-d/p)/2$.
- ❌ Check only machine capacity → model **every** binding resource (labor, tooling) in the MILP.
- ❌ Assume independent demand for the pooling case → apply the correlation correction.
- ❌ Derive $\sigma$ from MAPE on intermittent items → measure it, or use Croston/SBA.
- ❌ Pad safety stock to cover a forecast you don't trust → fix the forecast (it's a bias defect).

### Maturity ladder (where an organization sits, and the next rung)
`Spreadsheet reorder points → ERP MRP (infinite capacity) → APS finite-capacity planning →
MEIO buffer placement → full stochastic optimization + demand sensing.` Most companies are stuck
between MRP and APS; the jump to **MEIO + economically-set service levels** is usually the highest-
ROI move, and it's a *data and method* change, not a software purchase.

---

## Appendix — Model-selection reference (which model, when)

| Problem | Stable / simple | Realistic / hard | Don't use |
|---|---|---|---|
| **Forecast spread** | exp. smoothing, Holt–Winters; $\sigma$ from RMSE | **Croston / SBA** for intermittent | 1.25·MAPE·μ on lumpy items |
| **Service level** | — | **newsvendor** $C_u/(C_u+C_o)$, per ABC×XYZ | global fixed 95% |
| **Single-stage SS** | $z\sigma_{DDLT}$ (normal) | gamma / neg-binomial / empirical for skew | normal on slow movers |
| **Pooling benefit** | $\sqrt{\sum\sigma^2}$ (independent) | add $2\sum\rho\sigma_i\sigma_j$ correction | independence at the peak |
| **Multi-echelon placement** | — | **Graves–Willems** (guaranteed-service, general tree); **Clark–Scarf** (stochastic, serial); **METRIC** (repairable/low-volume) | per-node sum (v1) |
| **Lot sizing** | EOQ (constant demand); EPQ (finite rate) | **Wagner–Whitin** (optimal lumpy), Silver–Meal (heuristic); **Roundy** (cross-echelon) | EOQ on lumpy demand |
| **Capacity + lots** | fixed-period heuristics | **CLSP / CLSD MILP** (multi-resource); ELSP common-cycle (shared-line heuristic) | single-resource check |
| **Bullwhip** | — | **Lee–Padmanabhan–Whang** amplification; mitigate via visibility/EDI/EDLP/allocation | assuming demand = orders upstream |

### The two-paradigm rule, one more time
**Guaranteed-service (GSM):** bounded demand, deterministic service times, no stockouts within
bound, scales to big networks — *where to place decoupling buffers.* **Stochastic-service
(Clark–Scarf / base-stock):** probabilistic, allows stockouts, gives echelon order-up-to levels,
exact for serial systems. Pick GSM for network buffer *placement* at scale (your solver); reach for
stochastic-service or simulation when you need actual stockout probabilities, heavy-tailed demand,
or explicit lead-time uncertainty that GSM's deterministic-service core can't represent.

*End of v2. Every number here is reproducible from the Part-1 dataset; every assumption is in the
register; every model has a named source and a stated validity boundary. This is the reference;
the interactive explorers accompanying it let you feel how each lever moves the answer.*
