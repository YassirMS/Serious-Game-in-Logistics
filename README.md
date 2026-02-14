# SeGaLog: Global Supply Chain Optimization via Mixed-Integer Programming

**€96M profit over 3 periods (Each period represents one year). 27.7% net margin. 100% demand satisfaction.**

## Results

| Metric | Value | Impact |
|--------|-------|--------|
| **Net Profit (3 periods)** | €96.5M | 27.7% margin on €348M revenue |
| **Demand Fulfillment** | 100% | Zero stockouts across 6 regions, 3 products, 3 periods |
| **Solve Time** | 30s | <0.5% optimality gap (HiGHS solver) |
| **Strategic Investments** | 2 factories | 1 Medium (AFR) + 1 Large (AMS) at T+0 |
| **CO₂ Integration** | Direct cost | Carbon pricing embedded in objective function |

**Capacity evolution:**
- T+0: 517,500 units (2.5% margin over demand)
- T+1: 784,500 units (16.1% margin) — investments from T+0 operational
- T+2: 790,000 units (2.0% margin) — capacity utilization at 98%

---

## Problem

Global industrial company (SeGaLog) faces multi-continental supply chain planning across **6 regions** (Africa, North America, South America, Asia, Europe, Oceania), **3 products** (low/mid/high-end), and **3 time periods**.

**Constraints:**
- Satisfy 100% demand (no stockouts allowed)
- Variable production costs (labor €10-50/h by region)
- Asymmetric customs duties and container loss risks
- Factory capacity limits with TRS (synthetic yield rate)
- Carbon emissions from production (energy mix) and transport

**Strategic decisions:**
- Which factories to open/close (Small: 50K, Medium: 100K, Large: 250K units/period)
- Production allocation by factory
- Distribution flows across continents
- Timing of investments (1-period delay for new factories)

---

## Solution

Mixed-Integer Linear Program (MILP) minimizing total cost: production + transport + customs + factory operations + CO₂ emissions.

### Mathematical Model

**Decision variables (864 total):**
- Factory operations: `u[i,s,t]` (open), `v[i,s,t]` (close), `N[i,s,t]` (active count)
- Production: `x[i,p,t]` (units manufactured)
- Distribution: `y[i,j,p,t]` (units shipped), `n[i,j,p,t]` (containers)

**Objective function:**
```
min Z = Σ[production_cost + transport_cost + customs + factory_ops + CO₂_cost]
```

**Key constraints:**
1. **Demand satisfaction:** `Σ y[i,j,p,t] × (1 - ρ[i,j,t]) ≥ D[j,p,t]` (loss-adjusted arrivals ≥ demand)
2. **Capacity limits:** `Σ x[i,p,t] × h[p] ≤ Σ N[i,s,t] × CAP[s] × TRS[i,t]` (labor-hours ≤ capacity)
3. **Factory evolution:** `N[i,s,t] = N[i,s,t-1] + u[i,s,t-1] - v[i,s,t]` (1-period investment lag)
4. **Container logic:** `y[i,j,p,t] ≤ n[i,j,p,t] × Q[p]` (fractional containers rounded up)

### Strategy Implementation

**Hybrid demand approach:**
- **T+0:** Nominal demand (504,450 units) — initial system balanced at 97.5% utilization
- **T+1, T+2:** Pessimistic demand (upper CI bound: +10%, +16%) — risk-averse capacity planning

*Rationale:* Factories purchased at T+0 become operational only at T+1. Initial capacity (517,500 units) cannot handle T+0 pessimistic demand (529,672 units), so we use nominal. Subsequent periods provision for uncertainty via investments.

---

## Tech Stack

**Solver:** HiGHS (open-source MILP solver via `scipy.optimize.milp`)  
**Language:** Python  
**Problem size:** 864 variables (486 integer, 378 continuous), 558 constraints  
**Algorithm:** Branch-and-bound with cutting planes and presolve

---

## Key Decisions

### Investment Strategy (T+0)
- **1 Medium factory in Africa (100K units/period)** → Low labor cost (€12/h), serves local demand
- **1 Large factory in South America (250K units/period)** → Export hub for high-end products (P3)

**Why South America?**  
Labor cost (€15/h) + proximity to North America market + lower customs duties = optimal for P3 exports to Asia (16,404 units at T+1, 18,238 at T+2).

### Distribution Network Evolution

**T+0:** North America acts as primary hub  
- Exports to Africa, Asia, Oceania  
- Europe handles local demand + P3 exports to Asia

**T+1-T+2:** South America becomes dominant exporter  
- Combined capacity (Medium + Large = 280K units)  
- Supplies Asia with P3, Oceania with mixed products  
- North America shifts to Europe exports at T+2 (1,599 P3 units)

### No Closures
Zero factories closed across all periods—all existing facilities remain profitable. Optimization prioritizes opening over closing, leveraging geographic cost advantages.

---

## Architecture

### Problem Structure
```
6 Regions × 3 Products × 3 Periods
         ↓
54 Demand Requirements
         ↓
Optimization Loop:
├─ Factory operations (162 decisions)
├─ Production allocation (54 decisions)
├─ Distribution flows (324 decisions)
└─ Container logistics (324 decisions)
         ↓
Global Minimum Cost
- Satisfies all demand
- Respects capacities
- Accounts for CO₂ pricing
```

### Cost Breakdown (3 periods)
- **Production:** Labor (region-dependent: €10-50/h) + fixed cost
- **Transport:** Container cost + customs (11% of valorization) + loss risk (1-5% inter-region)
- **Factory ops:** Fixed cost by size + investment (€3-15M) - salvage (€1-5M)
- **Carbon:** Energy mix coefficient × (factory emissions + transport emissions) × CO₂ price

---

## What I Learned

**1. Temporal dynamics in strategic planning**  
The 1-period investment delay fundamentally changed optimization strategy. Naive approaches purchase capacity in-period, failing when T+0 pessimistic demand exceeds initial capacity. Our hybrid nominal/pessimistic approach aligns with industrial reality: you can't build factories overnight.

**2. Integer variables drive complexity exponentially**  
486 integer variables (factory decisions, container counts) made this NP-hard. Without preprocessing (HiGHS eliminated 30% of redundant constraints), solve time would exceed hours. Continuous relaxation bounds proved critical for branch-and-bound efficiency.

**3. Multi-objective trade-offs via pricing**  
Integrating CO₂ as a cost (€X/ton) rather than a separate objective creates implicit Pareto solutions. Varying CO₂ price (€50 vs €500/ton) shifts factory locations—low carbon price favors Asia (cheap labor, high emissions), high price favors Europe (renewables). This pragmatism beats pure bi-objective approaches for executive decision-making.

**Challenges:**

- **Demand uncertainty modeling:** Pure deterministic pessimistic approach leaves money on the table. Attempted two-stage stochastic programming but computational burden (5+ scenarios × 864 variables = intractable). Settled on upper confidence bound as practical compromise.
  
- **Asymmetry in real-world data:** Customs duties τ[i,j] ≠ τ[j,i] in reality (import tariffs differ from export), but data provided was symmetric. Used 2026 geopolitical estimates to adjust key routes (e.g., US-China tariffs).

- **Container loss risk:** Probabilistic constraint `y[i,j] × (1 - ρ[i,j])` models expected arrivals, but doesn't capture variance. High-risk routes (Africa-Oceania: 5% loss) got penalized indirectly via cost, not explicitly via chance constraints.

---

## Validation

**Demand satisfaction:** All 54 demand constraints met with equality (zero slack)  
**Capacity respect:** No factory exceeds TRS-adjusted capacity (98% utilization max at T+2)  
**Temporal consistency:** T+0 investments (Africa Medium, South America Large) confirmed operational at T+1 via factory count tracking  
**Financial audit:** Revenue (€348M) - Costs (€252M) = Profit (€96M), 27.7% margin

---

## Use Cases

**Manufacturing:** Multi-plant production scheduling with capacity expansion  
**Consumer Goods:** Global distribution network optimization (P&G, Unilever scale)  
**Automotive:** Parts sourcing with JIT constraints and emission regulations  
**Pharmaceuticals:** Compliant multi-region supply chains with GMP facilities  

---

## Complexity Analysis

**Variables:** 864 (486 integer, 378 continuous)  
**Constraints:** 558 (108 equality, 450 inequality)  
**Problem class:** NP-hard MILP  
**Solve time:** 30 seconds (HiGHS on Intel i7)  
**Scalability:** Linear growth with regions/products; exponential with integer variables

---

## Policy Implications

**Customer-first philosophy:** 100% demand satisfaction non-negotiable (competitive markets = lost sales are permanent)  
**Economic pragmatism:** CO₂ cost integrated, not minimized independently (carbon pricing reflects societal cost)  
**Risk-averse capacity planning:** Upper confidence bound provisioning prevents stockouts in high-uncertainty periods (T+1: +10%, T+2: +16%)  
**Geographic neutrality:** No political bias—optimal locations emerge from cost/emissions data (Africa chosen despite typical Western hesitancy)

---

## Author

Yassir Masfour  
Supervised by: Prof. Olivier Devise  
Sigma Clermont, 2025-2026

---

## Citation

```bibtex
@techreport{segalog_optimization_2026,
  author = {Masfour, Yassir},
  title = {SeGaLog: Global Supply Chain Optimization via Mixed-Integer Programming},
  institution = {Sigma Clermont},
  year = {2026},
  type = {Project Report}
}
```
