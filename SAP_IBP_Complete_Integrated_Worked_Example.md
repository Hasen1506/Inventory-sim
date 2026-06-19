# SAP IBP — The Complete Integrated Worked Example

*Extends the MEIO worked problem (`MEIO_SAP_IBP_Worked_Problem.md`) from a safety-stock
calculation into a **full enterprise plan**: the same physical network, now carried through
**every IBP module** — Demand → Inventory/MEIO → Supply (MRP + capacity) → Response →
S&OP/Finance → Control Tower — with the pieces the first pass deliberately left out:
**holding cost at each node, ordering and setup costs, reorder points, lot sizing, a
capacity-constrained production plan, the actual production schedule, and a live simulation
of how demand shocks consume safety stock and reshape the next reorder.***

> Currency is ₹ (illustrative). The network, demand, lead times, BOM, scrap and the
> safety-stock results are inherited unchanged from the MEIO problem; everything cost-,
> capacity-, lot- and time-related is new. One consistent dataset runs end to end.

---

## Part 0 — What the first pass left out (the gap this fills)

The MEIO problem answered exactly one question: *how big is the safety buffer at each node?*
That is one of **three** inventory components, and safety stock is usually the **smallest**
of the three in rupee terms. A real plan needs all of it:

| Layer | Question it answers | First pass? | Here |
|---|---|---|---|
| **Safety stock** | Buffer against demand & lead-time *variability* | ✅ done | inherited |
| **Cycle stock** | Inventory from ordering/producing in **batches** | ❌ | Parts 4, 6 |
| **Pipeline stock** | Inventory **in transit / in process** | ❌ | Part 6 |
| **Reorder point / order-up-to** | *When* to order and *how much* | ❌ | Part 5 |
| **Lot sizing** | Batch size (EOQ / EPQ / ELSP / FTL) | ❌ | Part 4 |
| **Holding cost gradient** | *Where* it's cheapest to hold | ❌ | Part 2, 6 |
| **Ordering / setup cost** | What drives batch size | ❌ | Part 2 |
| **Capacity constraint** | What happens when you **can't make it fast enough** | ❌ | Part 8 |
| **Production schedule** | The actual campaign sequence | ❌ | Part 9 |
| **Dynamics** | How a demand shock *consumes* buffer & changes the next order | ❌ | Part 10 |
| **Financial view** | Revenue, margin, working capital, turns, GMROI | ❌ | Part 12 |

The spine: *safety stock tells you the floor; lot sizing tells you the sawtooth on top of it;
reorder points tell you the trigger; capacity tells you whether the plan is even feasible;
the simulation shows the whole thing breathing.*

---

## Part 1 — The complete dataset (now costed and capacitated)

### Network (recap)
`Suppliers → Plant → CDC → {RDC A → Stores 1,2 ; RDC B → Stores 3,4}`. Plant makes **Product
A** and **Product B** on **one shared line**. Both products flow to all four stores.

### Demand (per day) and forecast error

| | Store 1 | Store 2 | Store 3 | Store 4 | Σ |
|---|--|--|--|--|--|
| **A** demand / MAPE | 20 / 20% | 30 / 25% | 24 / 15% | 16 / 30% | **90** |
| **B** demand / MAPE | 12 / 18% | 18 / 22% | 14 / 20% | 10 / 28% | **54** |

Daily error std $\sigma_d = 1.25\cdot\text{MAPE}\cdot\mu_d$. FG scrap A 3%, B 2% →
production rates $90/0.97=92.78$/day (A), $54/0.98=55.10$/day (B).

### BOM, suppliers, defect (recap)
A: P1×2 (Sup1), P2×1 (Sup2), P3×4 (Sup3, 5% defect), P4×1 (Sup4), P5×3 (Sup5). 
B: P5×2 (Sup5, **shared**), P6×1 (Sup6), P7×2 (Sup7). Non-P3 parts 2% defect.

### Lead times (mean, std — days)
Suppliers: Sup1 (7,2), Sup2 (10,3), Sup3 (5,1), Sup4 (14,4), Sup5 (8,2), Sup6 (12,3),
Sup7 (6,1.5). Manufacturing: A (2,0.4), B (1.5,0.3). Transit: Plant→CDC (3,0.5),
CDC→RDC A (2,0.4), CDC→RDC B (2.5,0.5), RDC A→S1 (1,0.2), RDC A→S2 (1.5,0.3),
RDC B→S3 (1,0.2), RDC B→S4 (2,0.4).

### Transport / FTL (truck capacity → natural review period)
A truck is the natural batch quantum. Review period $R=\text{truck}/(\text{combined A+B daily
throughput})$:

| Lane | Truck cap | Node combined demand/day | Review $R$ (days) |
|---|--|--|--|
| RDC→Store 1 | 120 | 20+12 = 32 | 3.75 |
| RDC→Store 2 | 120 | 30+18 = 48 | 2.50 |
| RDC→Store 3 | 120 | 24+14 = 38 | 3.16 |
| RDC→Store 4 | 120 | 16+10 = 26 | 4.62 |
| CDC→RDC A | 400 | 50+30 = 80 | 5.00 |
| CDC→RDC B | 400 | 40+24 = 64 | 6.25 |
| Plant→CDC | 800 | 90+54 = 144 | 5.56 |
| Supplier→Plant | — | per part | $R_p=5$ (assumed) |

### Capacity
One line, **16 production-hours/day**. Rates: A **10 units/hr**, B **20 units/hr**. Steady-state
load: A $92.78/10=9.28$h + B $55.10/20=2.76$h $=\mathbf{12.03}$ h/day → **75% utilization**.
Comfortable now — until the peak in Part 8.

### Service level
95% cycle service at all stores → $z=1.645$.

---

## Part 2 — The cost architecture (the foundation the first pass skipped)

Everything downstream — lot sizes, where to hold stock, the constrained-plan tradeoff —
falls out of three cost families. This is the part that was missing.

### 2.1 Part costs and the FG cost build-up

| Part | ₹/unit | | Build-up | Product A | Product B |
|---|--|---|---|--|--|
| P1 | 50 | | Material | 2(50)+1(80)+4(30)+1(120)+3(60) = **600** | 2(60)+1(90)+2(45) = **300** |
| P2 | 80 | | Conversion (line time) | 200 | 100 |
| P3 | 30 | | **FG cost at plant** | **₹800** | **₹400** |
| P4 | 120 | | | | |
| P5 | 60 | | Sell price | ₹1,200 | ₹650 |
| P6 | 90 | | Gross margin/unit | ₹250 (delivered) | ₹160 (delivered) |
| P7 | 45 | | | | |

Conversion is higher for A because it runs at half B's rate (10 vs 20/hr) — more machine-time
per unit.

### 2.2 The value gradient — unit cost rises at every echelon

A unit gains transport + handling value each time it moves downstream:

| Echelon | Product A value | Product B value |
|---|--|--|
| Plant FG | ₹800 | ₹400 |
| CDC (+inbound) | ₹850 | ₹430 |
| RDC (+transit) | ₹900 | ₹460 |
| **Store (shelf)** | **₹950** | **₹490** |

### 2.3 Holding cost per unit-year (carrying rate **25%/yr** = capital + storage + obsolescence)

$$h_{\text{node}} = 0.25 \times \text{unit value at node}$$

| | Plant FG | CDC | RDC | Store |
|---|--|--|--|--|
| **A** | ₹200 | ₹212.5 | ₹225 | ₹237.5 |
| **B** | ₹100 | ₹107.5 | ₹115 | ₹122.5 |

Raw-material holding at plant (25% of part cost): P1 ₹12.5, P2 ₹20, P3 ₹7.5, P4 ₹30,
P5 ₹15, P6 ₹22.5, P7 ₹11.25 per unit-yr.

> **The single most important consequence:** holding the *same physical unit* costs **19% more
> at the shelf than at the plant** (₹237.5 vs ₹200 for A). Combined with risk-pooling, this is
> the quantitative argument to **hold inventory as far upstream as the service target allows** —
> the decoupling-point / postponement decision. Cheap, pooled stock upstream; minimal,
> expensive stock at the edge.

### 2.4 Ordering, setup and shipment costs (what drives batch size)

| Cost | Value | Drives |
|---|--|---|
| Supplier PO (per order) | ₹500 | Raw-material EOQ |
| **Production changeover** (per A↔B switch) | **₹10,000** | Campaign size (EPQ/ELSP) |
| Plant→CDC shipment (per 800-truck) | ₹4,000 | CDC replenishment lot |
| CDC→RDC shipment (per 400-truck) | ₹2,500 | RDC replenishment lot |
| RDC→Store shipment (per 120-truck) | ₹1,200 | Store replenishment lot |
| Stockout penalty (lost margin/unit) | ₹250 (A) / ₹160 (B) | Service-level economics |

The big lever is the **₹10,000 changeover** — it's what makes the plant run monthly campaigns
instead of making a little of everything every day, and it's what creates the large plant
cycle stock you'll see in Part 6.

---

## Part 3 — IBP Demand: the time-phased demand plan

MEIO used demand as a *rate*. Supply planning needs it **time-phased** into buckets. Convert
to **weekly** (×7) over a **12-week** horizon. Steady baseline, with a seasonal peak introduced
in weeks 7–9 (the trigger for the capacity-constrained case in Part 8):

| Weekly demand | Wk 1–6 | **Wk 7–9 (peak)** | Wk 10–12 |
|---|--|--|--|
| Product A | 630 | **945** (+50%) | 630 |
| Product B | 378 | **548** (+45%) | 378 |

This time-phased plan is the **gross requirement** stream that the CDC pulls, which explodes
back through the network. The MAPE figures (Part 1) become the **forecast-error band** around
each bucket — the band MEIO sized the safety stock against, and the band the simulation in
Part 10 draws actuals from.

Two horizons, kept distinct (as in the MEIO doc): the **12-week planning horizon** gives the
time-phased demand and its error; each node's **coverage horizon** (its own net replenishment
time) is what safety stock is sized for — never the full 12 weeks.

---

## Part 4 — Lot sizing at every node

How big is each order/batch? Three different regimes apply at three layers of the network.

### 4.1 Stores and DCs — EOQ vs the truck

Classic EOQ: $Q^\* = \sqrt{2DS/h}$ (D = annual demand, S = order/shipment cost, h = annual
holding). Worked for **Store 2, Product A** (D = 30×365 = 10,950; S = ₹1,200; h = ₹237.5):

$$Q^\*_{\text{S2}} = \sqrt{\frac{2(10{,}950)(1{,}200)}{237.5}} = \sqrt{110{,}653} \approx 333 \text{ units}$$

But the **truck caps at 120**. EOQ (333) ≫ truck (120), so the unconstrained optimum is
infeasible in one shipment. Retail replenishment resolves this the practical way: **ship one
full truck frequently** rather than three trucks rarely — shelf space and freshness both push
the same direction. That recovers the **periodic-review** policy with $R$ from Part 1 (Store 2:
every 2.5 days). *Lesson: when EOQ exceeds the FTL quantum, the truck — not the EOQ formula —
sets the lot, and you ship often.*

### 4.2 Raw materials — supplier EOQ (the upstream lots)

Here EOQ governs cleanly. $S=₹500$/PO, $R_p=5$ days. Worked for all parts:

| Part | $D$/yr | $h$/yr | $\textbf{EOQ}=\sqrt{2DS/h}$ | Cycle stock $Q/2$ |
|---|--|--|--|--|
| P1 | 67,733 | 12.5 | **2,328** | 1,164 |
| P2 | 33,865 | 20 | **1,301** | 651 |
| P3 | 135,462 | 7.5 | **4,250** | 2,125 |
| P4 | 33,865 | 30 | **1,062** | 531 |
| P5 | 141,821 | 15 | **3,075** | 1,537 |
| P6 | 20,112 | 22.5 | **945** | 473 |
| P7 | 40,223 | 11.25 | **1,891** | 946 |

(If a supplier imposes an **MOQ** above its EOQ, the MOQ becomes the lot — a common real-world
override; none assumed binding here.)

### 4.3 The plant — one shared line → the Economic Lot Scheduling Problem (ELSP)

This is the subtle one. You **cannot** independently EPQ each product, because they share one
line — optimizing A's batch in isolation can starve B of capacity. You solve a **common cycle**:
produce both once per rotation, period $T$ chosen to balance total changeover cost against
total holding:

$$T^\* = \sqrt{\frac{2\sum_i S_i}{\sum_i h_i D_i\,(1-d_i/p_i)}}$$

with daily production rates $p_A=10\times16=160$/day, $p_B=20\times16=320$/day, so
$d_A/p_A=90/160=0.5625$ and $d_B/p_B=54/320=0.169$:

$$\sum S_i = 20{,}000,\quad \sum h_i D_i(1-d_i/p_i)= \underbrace{200(32{,}850)(0.4375)}_{2{,}874{,}375}+\underbrace{100(19{,}710)(0.831)}_{1{,}637{,}901}=4{,}512{,}276$$

$$T^\* = \sqrt{\frac{40{,}000}{4{,}512{,}276}} = 0.0942\text{ yr} \approx \mathbf{34\text{ days}}$$

So the plant runs a **~34-day rotation**, producing per cycle:

- **A:** $90 \times 34.4 = 3{,}093$ demand units → produce $3{,}093/0.97 = \mathbf{3{,}189}$ (scrap-grossed)
- **B:** $54 \times 34.4 = 1{,}856$ → produce $1{,}856/0.98 = \mathbf{1{,}894}$

Machine-hours/cycle: A $3{,}189/10=318.9$h + B $1{,}894/20=94.7$h $=413.6$h against
$16\times34.4=550$h available → **75% utilization**, consistent with the daily 12.03h. These
campaign sizes create the plant cycle stock in Part 6 and define the schedule in Part 9.

---

## Part 5 — Reorder points and the replenishment policy at every node

Lot sizing said *how much*; this says *when*. Two policy types, and the network uses both:

- **Continuous review $(s,Q)$** — order fixed $Q$ when inventory **position** drops to reorder
  point $s$. Position = on-hand + on-order − backorders.
  $$s = \mu_d\,L + SS$$
- **Periodic review $(R,S)$** — every $R$ days, order **up to** level $S$. This is what the
  FTL/review cadence forces on stores and DCs:
  $$S = \mu_d\,(L+R) + SS$$

The extra $\mu_d R$ term in $S$ is why periodic review needs more stock than continuous: you
must cover demand until the *next* review **plus** the lead time after it.

Worked for **Product A** at every node (SS values inherited from MEIO):

| Node | $\mu_d$ (A) | $L$ | $R$ | $SS$ | $s=\mu L+SS$ | $S=\mu(L{+}R)+SS$ | Policy |
|---|--|--|--|--|--|--|---|
| Store 1 | 20 | 1.0 | 3.75 | 19.1 | 39.1 | **114** | $(R,S)$ |
| Store 2 | 30 | 1.5 | 2.50 | 34.2 | 79.2 | **154** | $(R,S)$ |
| Store 3 | 24 | 1.0 | 3.16 | 17.0 | 41.0 | **117** | $(R,S)$ |
| Store 4 | 16 | 2.0 | 4.62 | 27.5 | 59.5 | **133** | $(R,S)$ |
| RDC A | 50 | 2.0 | 5.00 | 56.8 | 156.8 | **407** | $(R,S)$ |
| RDC B | 40 | 2.5 | 6.25 | 49.1 | 149.1 | **399** | $(R,S)$ |
| CDC | 90 | 3.0 | 5.56 | 96.9 | 366.9 | **867** | $(R,S)$ |
| Plant FG | 90 | mfg 2.0 | campaign | 76.1 | — | campaign trigger | ELSP |

The plant is the exception: it doesn't reorder against a point, it **produces to the ~34-day
ELSP rotation** (Part 4.3), so its "trigger" is the campaign calendar, not an $s$. Everything
downstream is pull-based order-up-to; the plant is push-based campaign. The boundary between
them — the CDC — is the **decoupling point** where push meets pull.

> Note the **ABC/XYZ texture**: Store 4 (A) has the *highest* MAPE (30%, an X→Z item) yet a
> modest $S$ because its volume is low; Store 2 (mid MAPE, highest volume) carries the largest
> store buffer. Policy should differ by segment — tight auto-replenishment for stable AX items,
> wider review and human eyes on volatile CZ items.

---

## Part 6 — IBP Inventory: total target = cycle + safety + pipeline, and where the money sits

Safety stock is one-third of the story. Total **average inventory** at a node:

$$\bar I_{\text{node}} = \underbrace{\tfrac{Q}{2}}_{\text{cycle}} + \underbrace{SS}_{\text{safety}} + \underbrace{\mu_d\,L_{\text{inbound}}}_{\text{pipeline}}$$

Cycle $=\mu_d R/2$ at review-based nodes (half the order quantity). Pipeline = demand in transit
inbound to the node. Multiply by unit value → **working capital**.

### Product A — inventory and rupees by node

| Node | cycle | safety | pipeline | total units | ₹/unit | **₹ invested** |
|---|--|--|--|--|--|--|
| Store 1 | 37.5 | 19.1 | 20 | 76.6 | 950 | 72,770 |
| Store 2 | 37.5 | 34.2 | 45 | 116.7 | 950 | 110,865 |
| Store 3 | 37.9 | 17.0 | 24 | 78.9 | 950 | 74,955 |
| Store 4 | 37.0 | 27.5 | 32 | 96.5 | 950 | 91,675 |
| RDC A | 125 | 56.8 | 100 | 281.8 | 900 | 253,620 |
| RDC B | 125 | 49.1 | 100 | 274.1 | 900 | 246,690 |
| CDC | 250 | 96.9 | 270 | 617.1 | 850 | 524,535 |
| **Plant FG A** | **1,547** | 76.1 | 180 | **1,803** | 800 | **1,442,080** |
| | | | | | **A subtotal** | **≈ ₹2,817,190** |

### Raw material at the plant

| Part | cycle ($Q/2$) | safety | pipeline ($\mu L$) | total | ₹/unit | **₹ invested** |
|---|--|--|--|--|--|--|
| P1 | 1,164 | 628 | 1,299 | 3,091 | 50 | 154,550 |
| P2 | 651 | 465 | 928 | 2,044 | 80 | 163,520 |
| P3 | 2,125 | 668 | 1,856 | 4,649 | 30 | 139,470 |
| P4 | 531 | 618 | 1,299 | 2,448 | 120 | 293,760 |
| P5 | 1,537 | 1,302 | 3,108 | 5,947 | 60 | 356,820 |
| P6 | 473 | 277 | 661 | 1,411 | 90 | 126,990 |
| P7 | 946 | 284 | 661 | 1,891 | 45 | 85,095 |
| | | | | | **RM subtotal** | **≈ ₹1,320,205** |

Product B (computed the same way; SS by node 10.5 / 18.6 / 12.6 / 16.2 at stores, 31.5 / 31.1
at RDCs, 57.2 at CDC, ~45 plant FG) adds **≈ ₹841,734**, of which Plant FG B alone is ₹421,600.

### The headline: the money is **upstream**, and it's **cycle stock**, not safety stock

| Where it sits | ₹ | Share |
|---|--|--|
| Stores (the field) | ~₹360k | 7% |
| RDCs + CDC | ~₹1.34M | 27% |
| **Plant FG (production batches)** | **~₹1.86M** | **37%** |
| **Raw material** | **~₹1.32M** | **27%** |
| **Total network inventory** | **≈ ₹4.98M** | 100% |

Two things fall out, both invisible in a pure safety-stock analysis:

1. **~64% of working capital sits at or before the plant** — in production batches and raw
   materials — not in the field. The lever on it is **lot sizing** (the ₹10k changeover driving
   3,093-unit campaigns) and **supplier EOQ**, *not* store safety stock. Halve the changeover
   cost and the plant cycle stock drops by ~30% ($\propto\sqrt{S}$) — far more rupees than any
   store-buffer tweak.
2. **Cycle + pipeline dwarf safety stock everywhere.** At the CDC, safety is 97 units of 617
   (16%); the rest is batching and transit. *Safety stock is the cheapest, smallest lever — the
   first pass optimized the least of the three.*

This is also where the **cost gradient pays off**: every unit pushed from shelf (₹237.5/yr to
hold) back to plant (₹200/yr) or held as RM is cheaper *and* poolable — the quantitative case
for a high decoupling point.

---

## Part 7 — IBP Supply: time-phased MRP and the unconstrained production plan

Now the demand plan (Part 3) explodes into a **production and procurement plan**. This is the
**supply heuristic** — rule-based, follows sourcing and lead time, **infinite capacity** (the
capacity reckoning is Part 8). MRP net-requirement logic, per bucket:

$$\text{Net req} = \text{Gross req} + \text{Safety stock} - \text{Projected on-hand} - \text{Scheduled receipts}$$

— note the sign the first pass flagged: **safety stock raises the requirement** (it's a target
to reach, not stock you have).

### 7.1 Master plan — Product A FG at the plant (lot = 3,093 campaign, $SS=76$)

| Week | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|---|--|--|--|--|--|--|--|--|--|--|--|--|
| Gross req | 630 | 630 | 630 | 630 | 630 | 630 | **945** | **945** | **945** | 630 | 630 | 630 |
| Sched receipt | | | 3093 | | | | 3093 | | | | 3093 | |
| Proj on-hand | 1020 | 390 | 2853 | 2223 | 1593 | 963 | 3111 | 2166 | 1221 | 591 | 3054 | 2424 |
| Planned release | →wk3 | | | | →wk7 | | | | →wk11 | | | |

(Opening on-hand 1,650; mfg lead time 2 days ≈ sub-week, so release ≈ same week the receipt is
needed.) Campaigns land in **weeks 3, 7, 11** — roughly the ELSP ~5-week cadence, pulled
slightly tighter by the peak. On-hand never breaches $SS=76$.

### 7.2 BOM explosion → dependent demand for P5 (the critical shared part)

Each A campaign of 3,189 produced units consumes **P5 ×3** = 9,567 (plus B campaigns consume
P5 ×2). So P5's demand is **lumpy** — flat between campaigns, then a spike in each campaign
week — and it must be offset by **Sup5's 8-day lead time** (~1.2 weeks):

| Week (P5) | … | wk 2 | **wk 3 (A campaign)** | wk 4 | … |
|---|--|--|--|--|--|
| Gross req (dependent) | low | low | **~9,567 (+B usage)** | low | |
| Net req (after $SS=1{,}302$, on-hand) | | | large | | |
| Planned **PO release to Sup5** | **placed wk 1–2** (offset 8 days) | | receipt arrives wk 3 | | |

Three things this makes concrete:
- **Dependent demand is spiky** because production is batched — which is *why* the P5 raw buffer
  (1,302) and Sup5's lead time dominate its inventory, exactly as the MEIO LT-term analysis
  predicted.
- **The PO to Sup5 must release ~8 days before the campaign** — lead-time offsetting is what
  turns a net requirement into a dated purchase order.
- **P5 couples A and B**: both campaigns draw the same part. A shock to *either* product's
  schedule moves P5's PO — the shared-supplier single-point-of-failure risk, now visible in the
  planning mechanics, not just the buffer math.

This plan is **feasible only if the plant can physically build it.** Weeks 7–9 ask for 50% more
A. Can the line do it? That is Part 8.

---

## Part 8 — The capacity-constrained version

The unconstrained plan assumed the line could make whatever weeks 7–9 demanded. Check it.

### 8.1 The load-vs-capacity reckoning

Available: $16\times7 = \mathbf{112}$ machine-hours/week. Demand-driven load:

| | A hours | B hours | Total/wk | vs 112h |
|---|--|--|--|--|
| **Steady** (wk 1–6, 10–12) | $9.28\times7=65.0$ | $2.76\times7=19.3$ | **84.3** | 75% ✅ |
| **Peak** (wk 7–9) | $13.92\times7=97.4$ | $4.08\times7=28.6$ | **126.0** | **113% ❌** |

The peak needs **126h/week against 112 available — a 14 h/week shortfall** for three weeks
(≈ 42 machine-hours ≈ **420 units of A-equivalent** that physically cannot be made in-window).
The infinite-capacity heuristic's weeks-7/8/9 campaigns are **infeasible.** IBP's **finite**
heuristic flags the overload; the **optimizer** then chooses how to resolve it — on cost.

### 8.2 The three resolutions and the cost that decides

| Option | Mechanism | Cost |
|---|---|--|
| **Pre-build** | Use slack in wk 4–6 to build ahead, carry into the peak | hold ~420 A-units ~3 wks ≈ **₹6,000** |
| **Overtime** | Add 2 h/day for 21 days (42 OT machine-hrs) at premium | ≈ **₹21,000** |
| **Lost sales** | Don't make ~420 units; forgo the demand | 420 × ₹250 margin ≈ **₹105,000** |

Pre-build is **~3.5× cheaper than overtime and ~17× cheaper than lost sales** → the optimizer
**pre-builds.** (This is precisely the cost-rate-driven tradeoff from the navigation guide:
get the ₹/unit-of-non-delivery wrong and the optimizer would happily choose lost sales or
burn overtime.)

### 8.3 The leveled plan — pre-build is a 4th inventory component

Smooth the load to ≤112h by pulling future production forward:

| Week | 1–3 | **4–6 (pre-build)** | **7–9 (peak)** | 10–12 |
|---|--|--|--|--|
| Demand load (h/wk) | 84 | 84 | **126** | 84 |
| **Leveled run (h/wk)** | 84 | **98** | **112 (max)** | 84 |
| Pre-build stock (machine-hr equiv) | 0 | builds +14/wk → **42** | drawn −14/wk → 0 | 0 |

Pre-build inventory peaks at end of week 6 (~420 A-equivalent units) and is fully consumed by
end of week 9. This **seasonal / pre-build stock sits on top of cycle + safety + pipeline** — a
component that appears *only* when capacity binds. **Capacity-constrained MEIO** therefore holds
*more* than the unconstrained buffer: you pre-position inventory not because demand is uncertain
but because you can't manufacture fast enough at the peak.

> **The conceptual shift:** unconstrained, inventory defends against *variability* (safety
> stock). Constrained, inventory **also** substitutes for *missing capacity* (pre-build). The
> optimizer trades pre-build holding cost against overtime and lost margin — and the right
> answer is entirely a function of the cost ratios in Part 2.

---

## Part 9 — The production schedule

The ELSP rotation (Part 4.3) turned into a calendar. Per ~34-day cycle the line runs **one A
campaign, one B campaign, two changeovers, and slack**:

| Phase | Work | Machine-hours | ≈ Days @16h |
|---|---|--|--|
| Changeover → A | setup (₹10,000) | — | ~0.5 |
| **Campaign A** | build 3,189 units @10/hr | 318.9 | **~20** |
| Changeover → B | setup (₹10,000) | — | ~0.5 |
| **Campaign B** | build 1,894 units @20/hr | 94.7 | **~6** |
| Slack | idle / maintenance buffer | ~137 | ~8 |
| | | **≈ 550h** | **≈ 34** |

A text view of one steady cycle (each block ≈ a few days):

```
│■■■■■■■■■■■■■■■■■■■■│×│▣▣▣▣▣▣│··········│   ← repeats every ~34 days
   Campaign A (20d)  CO  Camp B(6d)  slack
■ = Product A   ▣ = Product B   × = changeover   · = slack/maintenance
```

Annual changeover cost: ~10.7 cycles/yr × 2 changeovers × ₹10,000 ≈ **₹214,000/yr** — a real
S&OP line item, and the reason you don't shrink campaigns to reduce cycle stock without
counting the setup penalty.

**Under the peak (Part 8):** the schedule **compresses** — weeks 4–6 eat into slack to pre-build
(longer effective A run), weeks 7–9 run flat-out at 16h with little idle. The slack you saw in
steady state *is* the pre-build capacity; a plant already at 95% utilization would have nowhere
to pre-build and would be forced onto the expensive overtime/lost-sales options. **Utilization
headroom is itself a seasonal-buffering asset.**

---

## Part 10 — The dynamics: how a demand shock consumes safety stock and reshapes the next order

This is the question the static buffers can't answer. Simulate **Store 2 (Product A)** day by
day under continuous review: reorder point $s=79$, order-up-to $S=154$, lead time $L=2$ days
(order placed end of day $d$ arrives start of $d{+}2$), $SS=34$, forecast $30$/day with error
std $\sigma_d=1.25\times0.25\times30=9.4$/day. Actuals vary around the forecast, with a **demand
spike on day 7**:

| Day | Actual (fcst 30) | Begin OH | +Recv | −Dem | End OH | On-order | Position | Action | SS=34 |
|---|--|--|--|--|--|--|--|---|--|
| 1 | 30 | 154 | — | 30 | 124 | 0 | 124 | — | ok |
| 2 | 32 | 124 | — | 32 | 92 | 0 | 92 | — | ok |
| 3 | 29 | 92 | — | 29 | 63 | 91 | 154 | **order 91** (→d5) | ok |
| 4 | 31 | 63 | — | 31 | 32 | 91 | 123 | — | **brushed** (32<34) |
| 5 | 30 | 32 | **+91** | 30 | 93 | 0 | 93 | — | ok |
| 6 | 28 | 93 | — | 28 | 65 | 89 | 154 | **order 89** (→d8) | ok |
| **7** | **68 ⚡** | 65 | — | 68 | **0** | 89 | 86 | **STOCKOUT −3** | **pierced** |
| 8 | 45 | 0 | **+89** | 45 | 41 | 113 | 154 | **order 113** (→d10) | brushed |
| 9 | 33 | 41 | — | 33 | 8 | 113 | 121 | — | **deep (8≪34)** |
| 10 | 30 | 8 | **+113** | 30 | 91 | 0 | 91 | — | ok |
| 11 | 29 | 91 | — | 29 | 62 | 92 | 154 | **order 92** (→d13) | ok |
| 12 | 31 | 62 | — | 31 | 31 | 92 | 123 | — | brushed |
| 13 | 30 | 31 | **+92** | 30 | 93 | 0 | 93 | — | recovered |

### What the table shows

**Normal operation (days 1–6, 10–13): the sawtooth.** On-hand cycles between a cycle-low near
the reorder point and ~154 after each delivery. Safety stock gets **brushed** on days 4 and 12
(on-hand dips just below 34 while awaiting a delivery) — that is the buffer **doing its job**,
absorbing routine forecast error. Brief dips below the SS line are *expected* ~part of the time;
$s$ is the average position at reorder, not an inviolable floor.

**The spike (day 7): the buffer's tail risk materializes.** Demand of 68 is +38 over forecast —
about **+4σ**. The buffer was sized for 95% service (≈ +1.6σ over the lead-time window), so a
+4σ day **pierces it**: a **3-unit stockout**. This isn't a planning error — it's the **~5% of
cycles the 95% service level explicitly accepts.** To prevent it you'd raise the service level
(more SS, fast-rising cost) or sense the spike early (Part 11) — not "add a bit of buffer."

**The reorder *responds* — the next order is bigger.** Watch the order column: the post-spike
order on day 8 jumps to **113** vs the normal ~90, because inventory position fell further. And
because day-8 demand stays elevated (45), on-hand is drawn to just 8 on day 9 — a **near-second
stockout** while the large replenishment is still in transit. *A single shock reverberates for
more than one cycle*: spike → deeper drawdown → larger reorder → thin coverage until it lands →
recovery by day 13. **This is exactly "how reordering next time is done"** — the policy
self-corrects by ordering up to $S$ from a lower position, automatically pulling more.

### Transient shock vs level shift — the two ways "next time" differs

- **If the spike was noise** (a one-off bulk order): the system rebuilds to the *same* $S=154$;
  $SS$, $s$, $S$ are unchanged; orders return to ~90. Self-healing.
- **If the spike is the start of higher demand** (a level shift): leaving the policy alone means
  **chronic** stockouts, because $s$ and $S$ are still sized for μ=30. The fix is upstream — the
  forecast must rise. If demand sensing lifts μ from 30 → 38, then $\sigma_d$, $SS$, $s$ and $S$
  **all recompute upward** (new $S \approx 38(1.5{+}2.5)+\text{new }SS \approx 195$), and every
  future reorder is permanently larger. *The reorder point is only as good as the forecast
  feeding it — which closes the loop back to IBP Demand.*

### The cascade upstream — bullwhip in the mechanics

Store 2's order stream over the episode was **91, 89, 113, 92** vs a steady ~90 — the day-8
order is **+26%**. That swing doesn't stay local:

- **RDC A** serves Stores 1 + 2. S2's +26% order (plus S1's own variation) arrives as a
  **lumpier, larger draw**, so RDC A's inventory position hits its reorder point ($s=157$)
  **earlier and harder**, triggering an earlier/larger order to the CDC.
- **Amplification compounds:** store demand CV ≈ 9.4/30 = **31%**; after order-up-to batching
  plus the spike, the order stream to the RDC carries a **higher CV (~40–45%)**; the CDC, seeing
  batched orders from *both* RDCs, sees higher still. Order variance grows at each echelon —
  the textbook **bullwhip**, here emerging purely from periodic-review batching and one spike,
  with no irrationality required.
- This is why the **upstream buffers (CDC 97, plant FG 76, P5 1,302) exist** and why the plant's
  pre-build headroom matters: they absorb the amplified, lumpy demand that a single shelf spike
  becomes by the time it reaches the factory.

---

## Part 11 — IBP Response & deployment (executing against the live signal)

The plan (Parts 7–9) is periodic and pre-computed. **Response** is what happens between
planning runs, near real time, when actuals deviate — exactly the day-7 spike.

**Deployment & the push/pull boundary.** The plant **pushes** campaigns into the CDC on the
ELSP schedule. From the CDC down, replenishment is **pull** (order-up-to). The CDC is the
**decoupling point** — push meets pull here, which is also why it carries pooled safety stock
for the whole network.

**Fair-share allocation when constrained.** Suppose right after the spike the CDC can deploy
only **200 units of A**, but RDC A needs **150** (to recover from the Store-2 drawdown) and
RDC B needs its normal **120** — total want 270 > 200. The deployment heuristic allocates
**proportional to need**:

$$\text{RDC A} = 200\times\frac{150}{270}=111,\qquad \text{RDC B}=200\times\frac{120}{270}=89$$

— neither node is fully starved; the shortage is shared. (Alternatives: allocate by
shortfall-to-target, or by service-level priority if RDC A's stores are more critical.)

**Expediting.** Rather than wait for the next scheduled CDC→RDC A truck, Response can **expedite**
an off-cycle shipment to cover the day-9 near-stockout — trading a higher per-unit transport
cost (a partial truck, breaking FTL economics) against the ₹250/unit stockout penalty. The
optimizer expedites only while penalty > expedite premium.

**Demand sensing — catch the spike, don't just react to it.** IBP **demand sensing** reads the
day-7 order/POS signal within its short (days–weeks) horizon and lifts the near-term forecast
*immediately*, so Store 2's **next** order-up-to is sized for elevated demand **proactively** —
shrinking the day-9 thin patch. Sensing is the difference between the policy reacting a cycle
late and anticipating within the cycle. This is also the mechanism that distinguishes a
transient spike from a level shift (Part 10): sustained elevation in the sensed signal is what
trips the re-forecast.

---

## Part 12 — IBP for S&OP: the cycle, the scenarios, and the financial view

Everything above feeds **one monthly decision process**. The peak (Part 8) is precisely what
surfaces in it.

### 12.1 Where the constrained-plan decision gets made
`Product review → Demand review (consensus + sensing) → Supply review (the wk 7–9 overload
appears here) → Reconciliation/pre-S&OP (cost the options) → Executive S&OP (sign off)`. The
capacity overload isn't an IT exception — it's a **business tradeoff** escalated to the exec
meeting with a costed recommendation.

### 12.2 Scenario comparison (the versions/scenarios feature)

| Scenario | Peak service | Incremental cost | Margin impact vs ideal |
|---|--|--|--|
| **Pre-build** (recommended) | ~95% held | +₹6,000 holding | **−₹6,000** |
| Overtime | ~95% held | +₹21,000 premium | −₹21,000 |
| Lost sales / do nothing | ~85% (≈420 units short) | ₹0 out-of-pocket | −₹105,000 |
| **Demand shaping** (shift peak via promo/price timing) | ~95% | marketing cost | flattens load — *attacks the root* |

Pre-build wins on cost; demand shaping is the cross-functional option only S&OP can pull
(move demand *out* of the constrained window rather than chase it with supply). The exec meeting
picks **pre-build now, demand-shaping for next season.**

### 12.3 The financial picture (volume ↔ money — what makes it executive)

| Metric | Calculation | Value |
|---|---|--|
| Annual revenue | A 32,850×₹1,200 + B 19,710×₹650 | **₹52.23M** |
| COGS (plant cost) | A 32,850×₹800 + B 19,710×₹400 | ₹34.16M |
| **Gross margin** | 52.23 − 34.16 | **₹18.07M (34.6%)** |
| Avg inventory investment | from Part 6 | ₹4.98M |
| **Inventory turns** | COGS / avg inventory | **6.9×** |
| Days of inventory | 365 / turns | ~53 days |
| **GMROI** | gross margin / avg inventory | **3.6** (₹3.6 margin per ₹1 of stock) |

Now the Part 6 insight has teeth: since **64% of the ₹4.98M sits upstream in batches and RM**,
the highest-leverage way to lift turns and GMROI is **cutting the ₹10k changeover** (smaller
campaigns → less plant cycle stock) and **tightening supplier lots/lead times** — not squeezing
store safety stock. A 30% changeover-cost reduction lifts turns and frees ~₹400–500k of working
capital; the equivalent attack on store buffers would free a fraction of that.

---

## Part 13 — IBP Control Tower: the alerts that watch the whole loop

Every dynamic event above should be a **monitored exception**, not a surprise. The alert set,
wired to this example:

| Alert | Trips when | Severity | Routes to |
|---|---|--|---|
| **Capacity overload** | weekly load > 112h (wk 7–9) | High | Supply review → confirm pre-build |
| **Projected stockout** | projected on-hand < 0 (Store 2, day 7) | Critical | Response → expedite |
| **Safety-stock breach** | on-hand < SS (days 4, 9, 12) | by depth — day 9 *deep* = High | Response / planner |
| **Forecast bias** | tracking signal \|TS\| > 4 (spike = level shift?) | Med | **Demand** → re-forecast |
| **Pre-build shortfall** | actual pre-build < target end wk 6 | High | Supply → recover |
| **Single-source risk** | P5 / Sup5 dependency (both products) | Strategic | Sourcing → dual-source |

**Case management** turns each into an owned, resolved item: the stockout alert → expedite case;
the bias alert → re-forecast case (which, if confirmed, recomputes $SS$/$s$/$S$ as in Part 10);
the capacity alert → pre-build confirmation case. The Control Tower is the layer that **closes
the loop** — detect deviation, route it to the module that owns the fix, re-plan.

---

## Part 14 — The integrated picture

One network, one dataset, every module — and how they hand off:

```
 IBP DEMAND ──forecast + MAPE──► IBP INVENTORY/MEIO ──buffers──► IBP SUPPLY
  (Part 3,        time-phased        (Part 6: cycle+safety        (Part 7: MRP, BOM
   sensing P11)   demand + error      +pipeline, ₹4.98M)           explosion; Part 8:
       ▲                                                            capacity → pre-build;
       │                                                            Part 9: schedule)
       │                                                                 │
       │                                                            IBP RESPONSE
   re-forecast                                                     (Part 11: deploy,
   on level shift                                                   fair-share, expedite)
       │                                                                 │
       └──────────── IBP S&OP (Part 12: scenarios + financials) ◄────────┤
                              reconciles & decides                       │
                                       │                                 │
                              IBP CONTROL TOWER (Part 13) ◄──────────────┘
                              monitors the whole loop, routes exceptions back up
```

| Module | What it contributed here | Anchor number |
|---|---|--|
| Demand | Time-phased plan + error band; senses the spike | 630/wk, +50% peak |
| Inventory/MEIO | cycle + safety + pipeline at every node | **₹4.98M** total |
| Supply | MRP netting, BOM explosion, lot sizing | campaigns wk 3/7/11 |
| Supply (capacity) | Constrained plan, pre-build | 126h vs 112h → pre-build ₹6k |
| Schedule | ELSP rotation, changeovers | ~34-day cycle, ₹214k/yr setups |
| Response | Deploy, fair-share, expedite | 200 split 111/89 |
| S&OP | Scenarios + financials | GMROI 3.6, turns 6.9× |
| Control Tower | Alerts + cases | stockout/breach/overload |

### The five takeaways the full model delivers (that safety stock alone never showed)

1. **The money is upstream and it's cycle stock.** ~64% of working capital is in production
   batches and raw material; safety stock is the smallest, cheapest layer. The real levers are
   the **₹10k changeover** and **supplier lot/lead time**, not store buffers.
2. **The cost gradient + pooling argue for a high decoupling point.** A unit costs 19% more to
   hold at the shelf than the plant and can't be pooled there — hold cheap, pooled stock upstream;
   keep the edge thin.
3. **Capacity headroom is a buffering asset.** Pre-build is far cheaper than overtime or lost
   sales *here* — but only because the line runs at 75%. A plant at 95% has no pre-build room and
   is forced onto the expensive options. The choice is 100% a function of the **cost ratios**.
4. **A shelf spike amplifies upstream, and a 95% buffer accepts a tail.** The day-7 +4σ spike
   stocked out by design (the accepted 5%), drove a larger next order, and propagated as bullwhip
   to the RDC and CDC. Sensing + Response mitigate within the cycle; a true **level shift**
   requires **re-forecasting** (which recomputes ROP and order-up-to), not just reordering.
5. **Every engine exists to feed the monthly S&OP decision, and the Control Tower keeps it
   honest.** The capacity overload became a costed exec tradeoff; the stockout became an expedite
   case; the bias became a re-forecast — the loop, closed.

> *Safety stock told you the floor. Lot sizing built the sawtooth and revealed where the money
> really sits. Reorder points set the trigger. Capacity decided feasibility and forced the
> pre-build. The simulation showed the whole system breathing — a buffer consumed, an order
> enlarged, a shock rippling upstream — and the modules catching, costing, and closing it.*
