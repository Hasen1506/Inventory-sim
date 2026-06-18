# Operating SAP IBP — A Practitioner's Navigation Guide

*You have the theory and the MEIO math. This is the other half: what you actually
**do** inside the tool, in what order, and how to tell — for each module — what
separates someone clicking buttons from someone who knows why.*

> Assumes you already hold the supply-chain fundamentals and the inventory math.
> Nothing here re-teaches safety stock or EOQ; it teaches the **operating decisions**
> SAP IBP puts in front of you and the **sequence** to make them in.

---

## 0 — Where your focus actually goes (read this first)

The modules are not equal in leverage, and the parts that *feel* sophisticated are
rarely where the value or the failures live. The real distribution:

| Where effort feels like it should go | Where value and failure actually live |
|---|---|
| Picking the perfect forecast algorithm | **History cleansing + segmentation** |
| Tuning MEIO smoothing/parameters | **Lead-time-variability data quality** |
| Running the supply optimizer | **Getting the cost ratios right** |
| Knowing every Fiori app | **The data model & disaggregation logic** |
| The math inside each engine | **The monthly S&OP decision cycle** |

Four things, in priority order, that you should over-index on:

1. **The data model and disaggregation come before everything.** If you don't know
   *at what planning level* a key figure is stored and *how* a number splits down and
   rolls up, every screen will surprise you. This is the single most-skipped foundation
   and the source of most "why did my number land there?" confusion.
2. **In Demand, history quality beats model choice.** Beginners obsess over algorithms.
   The forecast is mostly determined by the history you feed it — stockout correction,
   outliers, promo separation. *Clean history + mediocre model beats garbage history +
   perfect model, every time.* Spend your energy on the cleansing pipeline and on
   segmentation so the right model class is applied automatically.
3. **MEIO and the optimizer are only as good as their inputs.** MEIO lives or dies on
   lead-time-variability data; the optimizer lives or dies on cost rates. The "modeling"
   *is* configuring those correctly — not the run itself.
4. **The S&OP cycle is the point.** Every module exists to feed a monthly decision
   meeting. A planner who can speak to the demand / supply / exec reviews is worth more
   than one who only knows screens.

**The dependency chain — never optimize downstream of bad upstream:**

```
  DATA MODEL  →  DEMAND  →  INVENTORY (MEIO)  →  SUPPLY  →  RESPONSE
   (Tier 0)      (error)      (buffers)        (plan)     (execute)
       └──────────── all reconciled in ──────────────┘
                         S&OP cycle
                              │
                       CONTROL TOWER  (monitors the whole loop, feeds alerts back)
```

You cannot optimize inventory on a bad forecast. You cannot plan supply on bad buffers.
Work the chain left to right; never tune a downstream engine to compensate for an
upstream defect — fix it upstream.

---

## Tier 0 — The data model & disaggregation (the foundation nobody assigns you)

Before any forecasting or optimization, IBP is a **time-series planning database**. Get
these five objects straight and the entire UI becomes legible:

| Object | What it is | Why you care operationally |
|---|---|---|
| **Planning area** | The container — the model of your whole planning problem | Everything you touch lives inside one |
| **Planning levels** | The dimensional grids (e.g. Product×Location×Customer×Month) | A key figure is *stored* at one level; edits elsewhere disaggregate to it |
| **Key figures** | The numbers (demand, forecast, stock, capacity…) | Some are **stored**, some **calculated** by formula — know which |
| **Attributes & master data** | Product/Location/Customer characteristics & hierarchies | Drive aggregation paths and segmentation |
| **Time profile** | The calendar buckets (day/week/month) and horizon | Defines what "a period" means everywhere |

**Disaggregation is the concept that trips up everyone.** When you type a number at an
aggregate level (say, a monthly regional forecast) IBP must push it down to the stored
base level. It does this by a **disaggregation basis**:

- **Proportional** — splits by the historical proportions of another key figure
  (e.g. spread regional demand to stores by each store's share of last year). This is the
  one you want most of the time.
- **Equal** — splits evenly. Almost never what you actually want; a silent source of bad
  store-level numbers.
- **Based-on key figure** — split by a chosen driver.

> **The mental model:** you can *view and edit* at any level, but the truth is stored at
> the base level, and roll-ups/splits are governed by rules. When a number "looks wrong,"
> 80% of the time you're looking at a disaggregation you didn't expect — not a calculation
> error. Always know which level you're standing on.

**Where you actually work:** the **Excel planning-view add-in** (most planner data entry,
fast grids) and the **Fiori web apps** (analytics, alerts, configuration, dashboards).
Master the Excel add-in first — it's where the day goes.

---

## 1 — IBP for Demand (your biggest-leverage module)

This is where you asked about outlier correction and data cleaning, and rightly — this is
where the most value and the most damage happen. The operational sequence, in order:

### Step 1 — Load and understand the history
You forecast on **demand history**, not raw sales history. The distinction matters because
sales are **censored**: you never sold what you couldn't fulfil. Before anything else,
establish which signal you have and whether stockouts are hiding true demand.

### Step 2 — Cleanse the history (the pipeline that determines your forecast)
Run these *in this order* — each later step assumes the earlier ones are done:

1. **Stockout / lost-sales correction.** If history is sales during periods when you were
   out of stock, you will systematically **under-forecast**. Uplift or flag those periods.
   This is the most commonly skipped correction and the most damaging.
2. **Outlier detection & correction.** IBP offers automatic methods you tune by
   sensitivity, broadly:
   - **Interquartile-range (Tukey) tests** — flag points outside *k×IQR*.
   - **Variance / ex-post tests** — flag points whose deviation from an expected value
     exceeds a tolerance.
   Once flagged, IBP **substitutes** a corrected value (median / mean / model-expected)
   into an *adjusted history* key figure — your raw history is preserved. **Decision:**
   correct automatically for high-volume stable items; **flag for human review** for
   high-value or strategic items where a "spike" might be a real signal.
3. **Promotion / event separation.** Pull promotional lift out of the baseline so the
   statistical model learns true base demand, and model promotions separately
   (with cannibalization where relevant). Don't let a past promo inflate the baseline.
4. **Missing-value & zero handling.** Decide: is a zero *true zero demand* or *no data*?
   They are treated very differently. Misclassifying zeros wrecks intermittent forecasts.
5. **Realignment.** When product/customer hierarchies change, realign history so the new
   structure inherits the right past.

> **Focus rule:** if you only get good at one thing in Demand, get good at this pipeline.
> The algorithm is downstream of it.

### Step 3 — Segment, then assign models by segment
Do **not** run one algorithm across everything. Segment with **ABC** (value) × **XYZ**
(demand variability/predictability), then map each segment to the right model *class*:

| Segment | Demand pattern | Model class to assign |
|---|---|---|
| AX / BX | High value, **stable** | Exponential smoothing (single/double), or **Automatic ES** |
| AY / BY | Trend &/or **seasonal** | **Holt-Winters** (additive or multiplicative) |
| Any **Z** | **Intermittent / lumpy** | **Croston / TSB**-type intermittent methods |
| Promo / causal-driven | Driven by price, weather, events | **Multiple Linear Regression** (causals) or **ML / gradient boosting** |
| Long, rich history, nonlinear | Enough data to learn from | **ML-based / Best-Fit** automated selection |

IBP's **Best-Fit / Automatic** mode will test a portfolio and pick by error — useful, but
*supervise it*: don't let it fit a seasonal model to 6 months of data, and don't let it
pick a complex model that beats a simple one by a rounding error (overfitting).

### Step 4 — Run the statistical forecast
Set **history horizon** (how far back it learns), **forecast horizon** (how far forward),
and the **periodicity**. The internal order of operations is: *preprocess (outliers,
missing) → fit model → post-process*. Know that this happens so you don't double-correct.

### Step 5 — Measure error AND bias (they are different)
Pick the metric for the job:

| Metric | Use it when |
|---|---|
| **MAPE** | Single item, reasonable volume; intuitive % |
| **WMAPE / weighted** | A **portfolio** — stops tiny low-volume items dominating |
| **MAD / RMSE** | Feeding safety-stock sizing; RMSE punishes big misses |
| **MASE** | Comparing across series of different scale |
| **Bias / Tracking signal** | **Always, separately** — detects systematic over/under-forecast |

> **The error vs bias trap:** a forecast can have low MAPE and still be badly **biased**
> (always 5% low). Error sizes your safety stock; **bias is a process defect you fix, not
> buffer.** Watch the tracking signal — if `|TS| > 4`, something is structurally wrong.

### Step 6 — Forecast Value Added (the maturity marker)
Compare statistical vs naïve vs the human-adjusted consensus forecast. **FVA** tells you
who is *adding* accuracy and who is *destroying* it. Frequently a well-meaning sales
override makes the forecast worse — FVA is how you prove it, diplomatically.

### Step 7 — Demand sensing (a different engine, short horizon)
Separate from statistical forecasting: **demand sensing** uses very recent signals (daily
orders, shipments, POS) to refine the **near-term** (days–weeks). Different horizon,
different purpose — it corrects the short end while the statistical forecast holds the
mid/long term.

### Step 8 — Consensus / demand review
The collaborative layer: sales/marketing reconcile statistical output into one
**consensus demand** plan. This becomes the input to the **demand review** in the S&OP
cycle (Section 5).

**Novice → expert in Demand:** novice tunes alphas and chases MAPE; expert engineers the
history pipeline, segments correctly, separates bias from error, and uses FVA to manage
the humans.

---

## 2 — IBP for Inventory (MEIO) — operating the engine you already understand

You have the math. Operationally the loop is short, and the skill is **input discipline**:

1. **Confirm inputs are ready** (all flow *from* upstream): mean demand + **forecast
   error/CV** (from Demand), **lead time *and its variability***, holding/cost data,
   service-level targets, and the BOM/sourcing network.
2. **Model the network** — echelons, BOM relationships, sourcing. This is where shared
   suppliers and convergence points get represented.
3. **Choose scope** — single-stage (each node in isolation) vs **multi-stage MEIO**
   (the optimizer places buffers across the network jointly). Multi-stage is the point of
   the module; single-stage throws away the pooling benefit.
4. **Run the optimization operator.** It returns recommended **safety stock**, **target
   stock**, and reorder positions per node.
5. **Read the outputs as components** — cycle / safety / pipeline / prebuild — not one
   lump. Sanity-check against the math: does the largest buffer sit where lead-time
   variance is worst? (If yes, the engine is behaving.)
6. **Scenario / what-if** — change service targets or service-time placement and compare
   total buffer cost across versions.

> **Focus rule for MEIO:** the engine is not where you spend time — **lead-time-variability
> data is.** It usually dominates the buffer. Most "MEIO gives crazy numbers" tickets are
> bad lead-time data, not bad math. Fight for that data quality before touching settings.

**Novice → expert in MEIO:** novice trusts (or distrusts) the number blindly; expert knows
*why* a given node's buffer is large, which input is driving it, and whether the
independence/pooling assumptions hold for that network.

---

## 3 — IBP for Supply & Response — planning, then executing

Two questions dominate: **heuristic or optimizer?** and **are my cost rates right?**

### Supply planning — heuristic vs optimizer

| | **Supply Heuristic** | **Optimizer** |
|---|---|---|
| Logic | Rule-based, follows sourcing & lead-time | Cost-based LP — minimizes cost / maximizes profit |
| Capacity | **Infinite** by default; finite via leveling after | Respects **constraints natively** |
| Tradeoffs | Won't make sourcing tradeoffs — follows rules | **Makes** sourcing/allocation tradeoffs |
| Use when | You want a transparent, rule-faithful plan; debugging | You want cost-optimal decisions across constraints |
| Risk | Can produce an infeasible (over-capacity) plan you must level | **Garbage cost rates → garbage decisions** |

**The heuristic** runs infinite-capacity first, then you apply **capacity checks /
leveling** to make it feasible — useful because it's transparent and easy to trace.

**The optimizer** is only as good as its **cost rates**: production, procurement,
transport, storage, and crucially **non-delivery / lost-sales penalty** costs. The engine
trades these off, so the *ratios* between them drive every decision. Setting an arbitrary
non-delivery cost is the classic optimizer failure — it'll either ship at any cost or
starve service depending on a number someone guessed.

> **Focus rule for Supply:** if you run the optimizer, you are really *maintaining a cost
> model.* Spend your time making the cost ratios reflect reality, not on the run.

### Response & deployment (execution layer)
Once a feasible supply exists:
- **Deployment** pushes available supply downstream toward demand.
- **Fair-share** logic allocates when supply is constrained (who gets shorted, and by how
  much).
- **Order-based / Response planning** handles confirmation and allocation closer to
  real time (ATP-like gating). This is the near-real-time end of the spectrum, distinct
  from the periodic time-series supply plan.

**Novice → expert in Supply:** novice runs the optimizer and trusts the output; expert
audits the cost model, knows when the heuristic's transparency is worth more than the
optimizer's optimality, and can explain *why* the plan shorted a given customer.

---

## 4 — IBP for S&OP — the cycle everything feeds

The modules exist to drive a **monthly decision process**. IBP is the system of record for
it. The standard cycle:

```
 Product Review → Demand Review → Supply Review → Reconciliation (Pre-S&OP) → Executive S&OP
   (portfolio)    (consensus      (feasibility,     (gaps, scenarios,           (decisions,
                   demand in)      capacity)          tradeoffs)                 sign-off)
```

What you operate here:

1. **Versions & scenarios.** A **version** is a baseline plan; **scenarios** are
   simulations branched off it ("what if demand +10%?", "what if we add a line?"). You
   compare them side by side to support the meeting.
2. **Financial integration.** The volume plan is translated to **revenue / cost / margin**
   so the operational plan and the financial plan are *one* plan. This is what makes S&OP
   an executive process and not a supply-chain hobby.
3. **Dashboards & analytics** for each review — the visuals that drive the decision.

**Novice → expert in S&OP:** novice produces numbers; expert frames **tradeoffs and
scenarios** for a decision meeting and connects volume to money.

---

## 5 — Control Tower — monitoring the whole loop

This is the layer that keeps the closed loop honest and feeds exceptions back upstream:

1. **Custom alerts** on thresholds — projected stockout, forecast bias breach
   (`|TS|>4`), capacity overload, target-inventory deviation. You *define* the conditions.
2. **Case management** — alerts become cases with an owner and a resolution workflow, so
   exceptions are worked, not just displayed.
3. **Network visibility & KPIs** — real-time view of the supply-chain network and the
   metrics that matter (service, inventory, forecast accuracy).

**Focus rule:** a good alert framework is what turns IBP from a planning batch job into a
*managed* operation. Design alerts around the failure modes you actually fear.

---

## 6 — A worked example: one SKU through the whole pipeline

*One dataset, walked end-to-end, so every step visibly does work — the same treatment as
the MEIO problem, but for the **operational** flow.*

**Setup.** Ceiling-fan SKU **CF-200** at the **Chennai RDC**, monthly demand, **24 months**
of history (two seasonal cycles — the minimum to trust a seasonal model). India seasonality:
fans peak Apr–Jun, trough Nov–Jan, on a mild upward trend.

### The recorded history (what the system actually shows)

| | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | Nov | Dec |
|---|--|--|--|--|--|--|--|--|--|--|--|--|
| **Y1** | 760 | 814 | 966 | 1218 | 1375 | **2100** | 1098 | 1006 | 963 | **1400** | 824 | 675 |
| **Y2** | 837 | 896 | 1062 | 1338 | **900** | 1412 | 1203 | 1102 | 1055 | 1006 | 901 | 737 |

Three problems are hiding here, and only one is obvious to the eye:

- **Jun Y1 = 2100** — a one-off bulk institutional order → an **outlier**.
- **Oct Y1 = 1400** — a **Diwali promotion** lift → must be *separated*, not deleted.
- **May Y2 = 900** — a peak month wrecked by a **stockout** → recorded sales are *censored*;
  true demand was far higher. **This is the dangerous one:** it looks like *low demand* but
  it's *unmet demand*. Forecast on it and you under-stock next peak and repeat the failure.

### Step 1 — Establish the signal
This is **sales** history, and the availability log shows CF-200 was **out of stock ~12 days
in May Y2**. That one fact reclassifies May Y2 from "low demand" to "censored." *You cannot
know a low month is a stockout without availability data* — so this is the first thing to go
find.

### Step 2 — Cleanse, in order

**2a · Stockout correction (May Y2, recorded 900).** Don't guess — estimate unconstrained
demand two independent ways:
- Prior-year May (1375) × observed YoY growth (~9.5%) ≈ **1506**
- Deseasonalized trend level (~1078) × May seasonal index (1.40) ≈ **1509**

Both converge → correct **up to ≈ 1509**. (Confidence comes from two methods agreeing, not
from one estimate.)

**2b · Outlier detection — deseasonalize FIRST.** *The nuance that separates novice from
expert:* run the outlier test on **deseasonalized** data, or your legitimate May/Jun peaks
get flagged as outliers. Dividing each month by its seasonal index gives a smooth rising
series (~950 → 1134) with three points sticking out: **Jun Y1 ≈ 1615**, **Oct Y1 ≈ 1556**
(both high), **May Y2 ≈ 643** (low). Tukey/IQR on that series:

- Q1 ≈ 986, Q3 ≈ 1108, IQR ≈ 122
- Fences (k = 1.5): lower ≈ **803**, upper ≈ **1292**

All three break the fence — the statistics catch every anomaly. **But the test also flags the
promo**, which is exactly why you never auto-correct blindly. Cross-reference the **promo
calendar**:

| Flagged month | On promo calendar? | Treatment |
|---|---|---|
| May Y2 (low) | — (it's a stockout) | already corrected **up** → 1509 ✓ |
| Jun Y1 (high) | No | genuine **outlier** → substitute ≈ 990 × 1.30 = **1287** (down) |
| Oct Y1 (high) | **Yes — Diwali** | *not* an outlier → send to promo separation |

**2c · Promotion separation (Oct Y1, recorded 1400).** Baseline = trend level (~1022) × Oct
index (0.90) ≈ **920**. Uplift = 1400 − 920 = **+480** (a +52% lift). Split it: the **baseline
history** for Oct Y1 becomes **920**; the **+480** goes into a separate promo key figure, so
the model learns clean baseline and the lift is re-applied only when a future promo is planned.

After cleansing, the history is the smooth signal underneath — trend + clean seasonality —
with all three distortions removed or quarantined.

### Step 3 — Segment, then assign the model
CF-200: high volume (~1000/mo → an **A** item), strong seasonality + mild trend → **AY**.
Seasonal amplitude **scales with level** (peak:trough ≈ 2:1 holds as volume grows — a *ratio*,
not a fixed unit gap), so assign **Holt-Winters multiplicative**, not additive.

### Step 4 — Run the forecast
On the **cleaned** history, Holt-Winters estimates ending level ≈ 1134, trend ≈ +8/mo, and the
familiar seasonal profile. Baseline forecast for next year's peak run:

| Y3 | Jan | Feb | Mar | Apr | May | Jun |
|---|--|--|--|--|--|--|
| **Forecast** | 914 | 978 | 1158 | 1458 | **1644** | 1537 |

### The payoff — why that stockout correction earned its keep
Watch what the May correction bought you. **Uncorrected**, Holt-Winters learns the May index
from two Mays — 1375 (real) and the censored 900 — pulling the estimated May index down from
1.40 to roughly **1.12**. Next May's forecast lands near **1315** instead of **1644**.

You'd plan to ~1315, true demand arrives at ~1644, and you **stock out by ~330 units in your
single highest-volume month** — the same failure, now baked into the forecast. *That one
cleansing step is worth ~330 units of avoided peak lost sales.* This is the concrete reason
history quality beats model tuning.

### Step 5 — Error AND bias (the trap, with numbers)
Six-month holdout, statistical forecast vs actual:

| | M1 | M2 | M3 | M4 | M5 | M6 |
|---|--|--|--|--|--|--|
| Forecast | 920 | 1180 | 1450 | 1010 | 980 | 1120 |
| Actual | 965 | 1235 | 1410 | 1070 | 1035 | 1185 |
| Error (A−F) | +45 | +55 | −40 | +60 | +55 | +65 |

- **MAPE = 4.7%** — passes any review on sight.
- **MAD = 53.3 units**
- **Bias (mean error) = +40 units/month** — five of six errors positive: a *systematic
  under-forecast.*
- **Tracking signal** = Σ error / MAD = +240 / 53.3 = **+4.50** → **|TS| > 4 → ALARM.**

In one line: *MAPE said "great forecast," the tracking signal said "stop."* A 4.7% error with
TS = 4.5 is **not** a buffering problem — it's a **structural bias you fix** (hunt the
always-on demand the model is missing), not one you paper over with safety stock. *Error sizes
the buffer; bias is a defect.*

### Step 6 — Forecast Value Added (manage the humans)

| Forecast | MAPE |
|---|---|
| Naïve (last-year-same-month) | 7.2% |
| Statistical (Holt-Winters) | 4.7% |
| Consensus (after sales override) | 5.9% |

Statistical **adds +2.5 pts** over naïve — real value. But the sales override **destroyed
~1.2 pts** versus the statistical forecast. FVA is how you prove that, with evidence, without
starting a turf war.

### Step 7 — Hand it down the chain (the whole loop)
The cleaned, bias-checked forecast now flows downstream — where it meets your MEIO work:

- **→ Inventory / MEIO:** mean ≈ 1644 for May with forecast-error std
  `σ_d ≈ 1.25 × 0.047 × 1644 ≈ 97 units` (the MAPE→σ rule from the worked MEIO problem). MEIO
  sizes the CF-200 buffer on this **corrected** mean and error — not a depressed, biased
  signal. Right number here → every buffer down the branch is right; the censored 900 → you
  under-buffer the whole branch.
- **→ Supply:** the optimizer/heuristic plans production to **1644**, not 1315 — the peak
  stockout never happens.
- **→ Control Tower:** the **TS = 4.50** breach is precisely the alert that fires and routes a
  case back to Demand — closing the loop this guide opened with.

**One SKU, one dataset, every module:** establish the signal → cleanse (stockout, outlier,
promo) → segment → model → measure error *and* bias → add value over naïve → hand a
trustworthy number down the chain.

---

## 7 — Your first 90 days walking into a company

Concrete sequence for arriving cold at an IBP shop:

**Week 1 — map the model, don't touch it.**
- Get the **planning area** structure: what planning levels exist, which key figures are
  **stored vs calculated**, the **time profile** and horizons.
- Learn the **disaggregation rules** for the key figures you'll edit. (This alone prevents
  most early mistakes.)
- Find out where work actually happens: which **Excel planning views** and which **Fiori
  apps** the team lives in.

**Weeks 2–4 — Demand, because it's upstream of everything.**
- Audit the **history-cleansing pipeline**: is stockout correction happening? How are
  outliers handled — auto or reviewed? Are promos separated?
- Check the **segmentation**: is one model run across everything (red flag), or is there
  ABC/XYZ-driven model assignment?
- Pull the **bias / tracking-signal** report. Systematic bias is the fastest credibility
  win you can deliver in month one.

**Month 2 — Inventory & Supply.**
- For MEIO: verify **lead-time-variability data quality**. Trace one oversized buffer to
  its driver.
- For Supply: find out **heuristic or optimizer**, and if optimizer, **review the cost
  rates** — especially the non-delivery penalty.

**Month 3 — the process and the monitoring.**
- Sit in (or shadow) the **S&OP cycle** — understand how demand/supply reviews feed the
  exec meeting and where the volume↔financial bridge lives.
- Review the **Control Tower alert** set — are the alerts aimed at the real failure modes,
  or is it noise nobody acts on?

By the end you should be able to answer, for this specific company: *where does the
forecast come from, how clean is its input, where are the buffers and why, how is supply
decided, and how do exceptions get caught and worked.* That five-part answer **is** the
core grasp.

---

## 7 — The whole thing on one page

| Module | The one operational question | Where your focus goes | Novice tell vs expert tell |
|---|---|---|---|
| **Data model** | Where is this number *stored*, and how does it split? | Disaggregation logic | "Why did it land there?" vs predicts it |
| **Demand** | Is my **history** clean and **segmented**? | Cleansing pipeline + segmentation | Tunes alphas vs engineers inputs |
| **Inventory** | Is my **lead-time-variability** data good? | Input data quality | Trusts number vs knows the driver |
| **Supply** | Heuristic or optimizer — and are **costs** right? | The cost model | Runs it vs audits the rates |
| **S&OP** | What's the **tradeoff** for the meeting? | Scenarios + financials | Makes numbers vs frames decisions |
| **Control Tower** | Are alerts aimed at real **failure modes**? | Alert design | Noise vs managed exceptions |

> **The single sentence:** *clean the demand history and segment it, trust MEIO only as far
> as your lead-time data, treat the optimizer as a cost model you maintain, and remember
> every engine exists to feed one monthly decision cycle that Control Tower keeps honest.*
