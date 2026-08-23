# The p-Center Problem

Minimax facility location, formulated as an integer program and solved with Gurobi. Given a set of
points and the distance between every ordered pair, choose exactly `p` of them as centers and assign
every remaining point to one center, so that the **largest** point-to-center distance is as small as
possible.

That objective is the right one whenever the binding requirement is a guarantee rather than an
average: ambulance and fire station siting, cell tower placement, disaster relief depots, or field
service territories with a maximum response time. It contrasts with the p-median problem, which
minimizes total distance and will accept one badly served outlier in exchange for a better mean.

A single notebook covers the formulation, three instances of increasing size, an analysis of where
the solver spends its time, and a measurement of how the algebraic form of one constraint changes the
strength of the linear relaxation.

---

## The model

**Decision variables.** $y_j \in \lbrace 0,1 \rbrace$ if point $j$ is opened as a center, $x_{ij} \in \lbrace 0,1 \rbrace$
if non-center point $i$ is assigned to center $j$ for $i \neq j$, and $\eta \ge 0$ is the covering
radius, meaning the largest realized assignment distance.

$$
\begin{aligned}
\min \quad & \eta \\
\text{s.t.}\quad
& \sum_{j} y_j = p && \text{open exactly } p \text{ centers}\\
& y_i + \sum_{j \neq i} x_{ij} = 1 && \text{point is either a center or assigned to one}\\
& x_{ij} \le y_j && \text{linking to open centers}\\
& d_{ij} x_{ij} \le \eta && \eta \text{ dominates every realized distance}\\
& x_{ij}, y_j \in \lbrace 0,1 \rbrace, \quad \eta \ge 0.
\end{aligned}
$$

Stated directly the problem is a nested $\min \max \min d_{ij}$, which is not linear. The radius
constraint removes the outer $\max$ by bounding every assignment distance by $\eta$, and the
assignment constraint removes the inner $\min$ by ensuring each point is served by exactly one
center. Both nested operators collapse into linear constraints this way.

---

## The data

Each instance file gives $n$, $p$, and the full $n \times n$ distance matrix:

```
N: <number of points>

P: <number of centers>

DIST: [
<row 1 of the distance matrix>
...
<row n of the distance matrix>
]
```

The files are read exactly as supplied, with no edits to the source. The parser tokenizes on
whitespace rather than relying on line breaks or column positions, so the differences in line
wrapping between the three files do not matter, and it checks that exactly $n^2$ values were
recovered before any model is built.

These are synthetic distance matrices rather than points in a plane. All three are asymmetric, so
$d_{ij}$ is read directionally as the distance incurred when point $i$ is served by center $j$, and
the formulation assumes neither symmetry nor the triangle inequality.

---

## Results

| Instance | n | p | Radius | Status | Runtime |
| --- | ---: | ---: | ---: | --- | ---: |
| `PC001.dat` | 10 | 3 | **257** | Optimal | 0.07 s |
| `PC002.dat` | 100 | 15 | **684** | Optimal | 23.59 s |
| `PC003.dat` | 200 | 25 | **1085** (best found) | Time limit, gap 26.73% | 600.08 s |

PC001 opens centers at points 5, 7 and 10, and the radius is set by a single pair, point 8 served by
center 7 at distance 257. PC003 ends at the ten minute limit with a lower bound of 795, so its result
is a best known radius rather than a proven optimum.

Every radius is recomputed from the raw distance matrix and asserted against the solver objective, so
no reported result rests on Gurobi's own output alone. Runtimes come from a single machine and will
vary with hardware, solver version and thread count; the objective values and LP bounds will not.

### Where the runtime goes

The PC002 solve is instrumented with a callback that records the incumbent and the bound throughout
branch-and-bound, because Gurobi retains only the final objective and gap once a solve ends.

| Milestone | Time | Fraction of runtime |
| --- | ---: | ---: |
| First feasible solution found | 0.53 s | 2.2% |
| Optimal solution first found | 21.84 s | 92.6% |
| Gap first below 10% | 21.84 s | 92.6% |

Almost the whole solve is search rather than proof. The bound had already climbed to 621 while the
incumbent was still 702, so finding the optimum closed the gap to 9.21% on its own and the proof that
followed took 1.75 seconds. Stopping at 15 seconds would have returned 702, only 2.6% above optimal,
but stopping at a 10% gap saves nothing, since the run reaches that threshold only by finding the
optimum.

### Formulation strength

The radius constraint above can also be written once per point instead of once per pair:

$$\sum_{j \neq i} d_{ij}\, x_{ij} \le \eta \qquad \forall\, i$$

The two are equivalent over the integers, since exactly one term in the sum is nonzero for any
feasible $x$, but they behave very differently in the LP relaxation, measured below. Both forms are
implemented and selected by the `Aggregated` flag on `build_model`.

Solving the LP relaxation of both forms of constraint (4) gives:

| Instance | Rows (4) | LP bound (4) | Rows (4a) | LP bound (4a) | Improvement |
| --- | ---: | ---: | ---: | ---: | ---: |
| `PC001.dat` | 191 | 22.54 | 111 | 88.56 | 3.9x |
| `PC002.dat` | 19,901 | 17.18 | 10,101 | 284.66 | 16.6x |
| `PC003.dat` | 79,801 | 16.04 | 40,201 | 356.98 | 22.3x |

Gurobi largely finds this on its own. The PC002 solver log reports a root relaxation matching the
aggregated bound rather than the per-pair one, and presolve cuts the model down to 10,088 rows,
almost exactly what (4a) states outright. So writing it aggregated buys a smaller model rather than a
bound the solver would have missed.

What neither form fixes is the distance that remains: on PC002 the aggregated bound still sits at
under half the optimum in the results table above. That gap is intrinsic to the minimax structure
rather than an artifact of how the constraint is written, and it is why PC003 does not close within
the time limit.

---

## Requirements

| Library | Used for |
| --- | --- |
| `gurobipy` | mixed-integer programming |
| `numpy` | distance matrix handling |
| `jupyterlab` | running the notebook |

The `pip install gurobipy` package ships with a size-limited licence covering models up to 2,000
variables and 2,000 constraints. PC001 builds 101 variables and 191 rows and runs under it, but PC002
and PC003 build 10,001 and 40,001 variables and need a full or academic licence. Academic licences
are free from [Gurobi](https://www.gurobi.com/academia/academic-program-and-licenses/).

---

## Running it

```bash
git clone https://github.com/JayeshYevale/p-Center-Problem.git
cd p-Center-Problem
pip install -r requirements.txt
jupyter lab
```

The data files sit beside the notebook, so it runs top to bottom with no path configuration. Outputs
are committed, so the models and results can be read without installing a solver.

---

## Repository layout

```
p-center-problem/
├── p_center_problem.ipynb
├── PC001.dat
├── PC002.dat
├── PC003.dat
├── requirements.txt
├── LICENSE
└── README.md
```

---

## Further reading

- Daskin, M. S. (2013). *Network and Discrete Location: Models, Algorithms, and Applications*, 2nd ed. Wiley.
- Elloumi, S., Labbé, M., & Pochet, Y. (2004). A new formulation and resolution method for the p-Center problem. *INFORMS Journal on Computing*, 16(1), 84 to 94. Recasts the problem as a sequence of set-covering feasibility questions solved by binary search over the distinct distances, which is how large instances are handled in practice.
- Kariv, O., & Hakimi, S. L. (1979). An algorithmic approach to network location problems. I: The p-centers. *SIAM Journal on Applied Mathematics*, 37(3), 513 to 538.

---

## Licence

MIT, see [LICENSE](LICENSE).
