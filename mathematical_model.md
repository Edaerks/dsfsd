# Vending Machine Replenishment & Repair Routing — Mathematical Model

## Problem Description

Given a snapshot at `mid_day_time`, we re-route trucks from that point onward to minimize total cost. Trucks service vending machines (replenishment and/or repair) subject to time windows, travel times, and operational constraints.

---

## Sets & Indices

| Symbol | Description |
|--------|-------------|
| $\mathcal{M}$ | Set of machines, indexed by $i$ |
| $\mathcal{M}^F \subseteq \mathcal{M}$ | Set of **failed** machines (those with `failed_at` field) |
| $\mathcal{M}^R \subseteq \mathcal{M}$ | Set of machines that still require service after `mid_day_time` |
| $\mathcal{K}$ | Set of trucks, indexed by $k$ |
| $\mathcal{N}$ | Set of network nodes, indexed by $n$ |
| $\mathcal{A}$ | Set of arcs $(n_1, n_2) \in \mathcal{N} \times \mathcal{N}$ |
| $0$ | Depot node |

---

## Parameters

| Symbol | Description |
|--------|-------------|
| $t_{n_1, n_2}$ | Travel time on arc $(n_1, n_2) \in \mathcal{A}$ |
| $[e_i, l_i]$ | Time window for machine $i$ (earliest, latest service start) |
| $s_i^{rep}$ | Replenishment service duration for machine $i$ |
| $s_i^{fail}$ | Failure repair service duration for machine $i \in \mathcal{M}^F$ |
| $d_i$ | Demand rate (priority weight) of machine $i$ |
| $\lambda_f$ | Cost weight for unrepaired failures |
| $\lambda_d$ | Cost weight for demand-related cost |
| $T^{mid}$ | Mid-day time (re-routing decision point) |
| $T^{end}$ | End of operational day |
| $\text{node}(i)$ | Network node where machine $i$ is located |
| $P_k$ | Position/node of truck $k$ at time $T^{mid}$ (determined from original route) |
| $T_k^{avail}$ | Earliest available time of truck $k$ after $T^{mid}$ (if en-route, must finish current visit first) |
| $C_k$ | If truck $k$ is currently committed to a machine at $T^{mid}$, that machine's ID (otherwise $\emptyset$) |

---

## Decision Variables

| Variable | Domain | Description |
|----------|--------|-------------|
| $x_{ijk}$ | $\{0, 1\}$ | 1 if truck $k$ travels directly from machine $i$ to machine $j$ (including depot as source/sink) |
| $y_{ik}$ | $\{0, 1\}$ | 1 if machine $i$ is assigned to truck $k$ |
| $r_i$ | $\{0, 1\}$ | 1 if failed machine $i \in \mathcal{M}^F$ is **repaired** |
| $v_i$ | $\{0, 1\}$ | 1 if machine $i$ is **visited** (replenished) |
| $a_i$ | $\mathbb{R}_{\geq 0}$ | Arrival time at machine $i$ |
| $w_i$ | $\mathbb{R}_{\geq 0}$ | Service start time at machine $i$ |
| $\delta_i$ | $\mathbb{R}_{\geq 0}$ | Departure time from machine $i$ |

---

## Objective Function

$$\min \quad \lambda_f \cdot \sum_{i \in \mathcal{M}^F} d_i \cdot (1 - r_i) \quad + \quad \lambda_d \cdot \sum_{i \in \mathcal{M}^R} d_i \cdot (1 - v_i)$$

**Component 1 — Failure cost:** Penalizes each failed machine left unrepaired, weighted by its demand rate.

**Component 2 — Demand cost:** Penalizes each machine left unvisited (not replenished), weighted by its demand rate.

> **Note:** If the problem requires penalizing *lateness* rather than binary visit/no-visit, an alternative formulation is:
>
> $$\lambda_d \cdot \sum_{i \in \mathcal{M}^R} d_i \cdot \max(0,\ w_i - e_i) \cdot v_i$$
>
> which charges a cost proportional to how long after the earliest service time the machine is actually served.

---

## Constraints

### 1. Assignment & Routing

**(1a) Each machine is assigned to at most one truck:**

$$\sum_{k \in \mathcal{K}} y_{ik} \leq 1, \quad \forall\, i \in \mathcal{M}^R$$

**(1b) A machine is visited if and only if it is assigned:**

$$v_i = \sum_{k \in \mathcal{K}} y_{ik}, \quad \forall\, i \in \mathcal{M}^R$$

**(1c) Flow conservation — each truck enters and leaves each visited machine exactly once:**

$$\sum_{j \in \mathcal{M}^R \cup \{0\}} x_{ijk} = y_{ik}, \quad \forall\, i \in \mathcal{M}^R,\ \forall\, k \in \mathcal{K}$$

$$\sum_{j \in \mathcal{M}^R \cup \{0\}} x_{jik} = y_{ik}, \quad \forall\, i \in \mathcal{M}^R,\ \forall\, k \in \mathcal{K}$$

**(1d) Each truck departs from its starting position and returns to depot:**

$$\sum_{j \in \mathcal{M}^R} x_{0jk} \leq 1, \quad \forall\, k \in \mathcal{K}$$

$$\sum_{j \in \mathcal{M}^R} x_{j0k} \leq 1, \quad \forall\, k \in \mathcal{K}$$

$$\sum_{j \in \mathcal{M}^R} x_{0jk} = \sum_{j \in \mathcal{M}^R} x_{j0k}, \quad \forall\, k \in \mathcal{K}$$

---

### 2. Time Feasibility

**(2a) Service cannot start before arrival:**

$$w_i \geq a_i, \quad \forall\, i \in \mathcal{M}^R$$

**(2b) Service start respects time window (earliest time):**

$$w_i \geq e_i \cdot v_i, \quad \forall\, i \in \mathcal{M}^R$$

> If the time window lower bound is a **hard** constraint. If it is soft, move this into the objective as a penalty.

**(2c) Service must complete before the time window closes:**

$$w_i + s_i^{rep} + s_i^{fail} \cdot r_i \leq l_i + M \cdot (1 - v_i), \quad \forall\, i \in \mathcal{M}^R$$

> **Note on soft time windows:** Based on the example solution, the time window may be a **soft** constraint (especially for repair operations). If so, replace the hard constraint with a penalty term in the objective, or relax it only for repair:
>
> $$w_i + s_i^{fail} \cdot r_i + s_i^{rep} \leq l_i + M \cdot (1 - v_i) + \text{slack}_i$$

**(2d) Departure time includes all service durations:**

$$\delta_i = w_i + s_i^{rep} \cdot v_i + s_i^{fail} \cdot r_i, \quad \forall\, i \in \mathcal{M}^R$$

**(2e) Travel time linking between consecutive visits (subtour elimination with time):**

$$a_j \geq \delta_i + t_{\text{node}(i),\text{node}(j)} - M \cdot (1 - x_{ijk}), \quad \forall\, i,j \in \mathcal{M}^R,\ i \neq j,\ \forall\, k \in \mathcal{K}$$

**(2f) First visit from starting position:**

$$a_j \geq T_k^{avail} + t_{P_k, \text{node}(j)} - M \cdot (1 - x_{0jk}), \quad \forall\, j \in \mathcal{M}^R,\ \forall\, k \in \mathcal{K}$$

**(2g) Return to depot before end of day:**

$$\delta_i + t_{\text{node}(i), 0} \leq T^{end} + M \cdot (1 - x_{i0k}), \quad \forall\, i \in \mathcal{M}^R,\ \forall\, k \in \mathcal{K}$$

---

### 3. Repair Constraints

**(3a) Only failed machines can be repaired:**

$$r_i = 0, \quad \forall\, i \notin \mathcal{M}^F$$

**(3b) Repair requires a visit:**

$$r_i \leq v_i, \quad \forall\, i \in \mathcal{M}^F$$

---

### 4. Committed Visit Constraint

If a truck $k$ is en-route to machine $C_k$ at $T^{mid}$, that visit must be completed:

$$y_{C_k, k} = 1, \quad \forall\, k \in \mathcal{K} \text{ where } C_k \neq \emptyset$$

---

### 5. Variable Domains

$$x_{ijk} \in \{0, 1\}, \quad y_{ik} \in \{0, 1\}, \quad r_i \in \{0, 1\}, \quad v_i \in \{0, 1\}$$
$$a_i, w_i, \delta_i \geq 0$$

---

## Determining Initial Conditions at $T^{mid}$

For each truck $k$, examine its original route at `mid_day_time`:

1. **If the truck is idle at the depot:** $P_k = 0$, $T_k^{avail} = T^{mid}$, $C_k = \emptyset$.

2. **If the truck is servicing a machine $m$:** The truck must complete the service. $P_k = \text{node}(m)$, $T_k^{avail} = \delta_m^{original}$, $C_k = m$.

3. **If the truck is traveling between two nodes:** The truck must complete travel to its destination. If traveling to machine $m$: $P_k = \text{node}(m)$, $T_k^{avail} = a_m^{original}$, $C_k = m$ (must complete the planned visit).

A machine is in $\mathcal{M}^R$ (still requires service) if it has **not yet been fully serviced** by $T^{mid}$ in the original route.

---

## Verification with Example (`instance_0`)

**Given:** 1 truck, 4 machines, `mid_day_time = 120`

**At $T^{mid} = 120$:**
- Truck 0 has completed: M3 (dep=75), M0 (svc_start=115, dep=140 → in service at t=120)
- Truck is servicing M0 at t=120 → must finish M0 first: $T_0^{avail} = 140$, $P_0 = \text{node}(0) = 1$

**Remaining machines:** $\mathcal{M}^R = \{1, 2\}$ (M0 and M3 already done/in-progress)

**Failed:** Machine 2 ($\text{failed\_at} = 90$, $s^{fail} = 30$)

**Solution re-routes:** After M0 → visits M2 (Repair + Replenish) → M1 (Replenish) → Depot

This confirms the model: the solution prioritizes repairing the failed machine (M2) and reorders visits to minimize total cost.

---

## Big-M Value

Set $M$ to a sufficiently large value:

$$M = T^{end} + \max_{(n_1, n_2) \in \mathcal{A}} t_{n_1, n_2}$$

---

## Summary

This is a **Mixed-Integer Linear Program (MILP)** combining:
- Vehicle Routing Problem with Time Windows (VRPTW)
- Repair scheduling for failed machines
- Re-routing from mid-day state

The model can be solved with standard MILP solvers (Gurobi, CPLEX, SCIP, OR-Tools).
