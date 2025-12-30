# Command locations (short) ✅

- `&get -n` — [src/base/abci/abc.c:1236](src/base/abci/abc.c#L1236) — registers `Abc_CommandAbc9Get`
- `&put` — [src/base/abci/abc.c:1237](src/base/abci/abc.c#L1237) — registers `Abc_CommandAbc9Put`
- `&st` — [src/base/abci/abc.c:1270](src/base/abci/abc.c#L1270) — registers `Abc_CommandAbc9Strash`
- `&syn2` — [src/base/abci/abc.c:1308](src/base/abci/abc.c#L1308) — registers `Abc_CommandAbc9Syn2`
- `&sweep` — [src/base/abci/abc.c:1334](src/base/abci/abc.c#L1334) — registers `Abc_CommandAbc9Sweep`
- `&if -g -K 6` — [src/base/abci/abc.c:1342](src/base/abci/abc.c#L1342) — registers `Abc_CommandAbc9If`
- `&nf` — [src/base/abci/abc.c:1351](src/base/abci/abc.c#L1351) — registers `Abc_CommandAbc9Nf`
- `&dch` — [src/base/abci/abc.c:1377](src/base/abci/abc.c#L1377) — registers `Abc_CommandAbc9Dch`

## Core struct definitions (short) 🧭

- `Gia_Man_t` — [src/aig/gia/gia.h:96](src/aig/gia/gia.h#L96) — `typedef struct Gia_Man_t_ Gia_Man_t;` / `struct Gia_Man_t_`
- `Gia_Obj_t` — [src/aig/gia/gia.h:76](src/aig/gia/gia.h#L76) — `typedef struct Gia_Obj_t_ Gia_Obj_t;` / `struct Gia_Obj_t_`
- `Abc_Frame_t_` — [src/base/main/mainInt.h:60](src/base/main/mainInt.h#L60) — `struct Abc_Frame_t_`
- `Abc_Ntk_t` — [src/base/abc/abc.h:115](src/base/abc/abc.h#L115) — `typedef struct Abc_Ntk_t_ Abc_Ntk_t;` (body at [abc.h:153](src/base/abc/abc.h#L153))
- `Abc_Obj_t` — [src/base/abc/abc.h:116](src/base/abc/abc.h#L116) — `typedef struct Abc_Obj_t_ Abc_Obj_t;` (body at [abc.h:128](src/base/abc/abc.h#L128))

## ABC command-script examples 🧾

- `ra i10.aig`
- `&get -mn`
- `&put -v`
- `quit`

## AIG format (brief) 📄

- Header: `aig M I L O A` — M = max variable index, I = #inputs, L = #latches, O = #outputs, A = #AND gates.
- Following sections (ASCII lines): inputs (one literal per input), latches (per-latch literals), outputs (one literal per output), then A lines for AND gates (each: `lhs rhs0 rhs1`).
- Literals are integers: even = variable, odd = negation of the variable (literal n is var n/2; odd means inverted).
- For full details and corner cases (symbol table, comments), see the AIGER spec: http://fmv.jku.at/aiger
  Run and capture outputs (shell example):

```bash
# create timestamped files
TS=$(date +%Y%m%d_%H%M%S)
./abc -f scripts/demo_node_assoc.abc    > demo_1_${TS}.txt 2>&1
./abc -f scripts/demo_node_assoc_gia.abc > demo_1_gia_${TS}.txt 2>&1
```

# Personal Notes

### Cut

What a cut is (in an AIG / logic network)

A cut of a node n is a set of nodes L = {l₁,…,l_k} (called leaves) such that every path from any primary input (PI) to n passes through at least one leaf.

So the leaves “separate” n from the PIs. If k ≤ K, it’s a K-feasible cut.

Example 1 (tiny AIG)
a b c (PIs)
\ / \
 x=AND(a,b)
\ /
n=AND(x,c)

Cuts for n:

{n} is a cut (trivial, leaf is the node itself).

{x, c} is a cut: every PI→n path hits x or c.

{a, b, c} is a cut: every path hits one of the PIs.

Not a cut:

{x} alone is not a cut because the path c → n reaches n without going through x.

Example 2 (K-feasible meaning)

If K=2, then {x,c} is 2-feasible, but {a,b,c} is not (it has 3 leaves).

### K-feasible Cuts

Setup: a small logic cone

Assume this logic feeding node N:

a b c d e (PIs)
\ / \ /
n1 n2
\ /
n3
|
N

Where:

n1 = AND(a, b)
n2 = AND(c, d)
n3 = AND(n1, n2)
N = AND(n3, e)

Goal

Break N into K-feasible cuts.
Let K = 3.

Step 1: Trivial cut

Every node has this:

Cut(N) = { N }

✔ size = 1 ≤ K

Step 2: One level down

Fanins of N:

Cut(N) = { n3, e }

✔ size = 2 ≤ K
All PI→N paths go through either n3 or e.

Step 3: Expand n3

Fanins of n3 are n1 and n2:

Candidate cut = { n1, n2, e }

✔ size = 3 ≤ K
This is a valid 3-feasible cut.

Interpretation:

The Boolean function at N can be expressed purely in terms of (n1, n2, e).

Step 4: Try expanding further (fails K-limit)

Expand n1 → {a,b} and n2 → {c,d}:

Candidate = { a, b, c, d, e }

❌ size = 5 > K
❌ Not K-feasible

So this cut is rejected for K=3.

Summary of valid K-feasible cuts (K=3)
Cut leaves Size Valid
{N} 1 ✔
{n3, e} 2 ✔
{n1, n2, e} 3 ✔
{a,b,c,d,e} 5 ✘
What ABC does with this

For each valid cut:

Compute Boolean function F_N in terms of the cut leaves

Canonicalize / decompose it

Build a Dec_Graph

Convert it to AIG via Dec_GraphToAig()

Conceptually:

N(a,b,c,d,e)
→ N(n1,n2,e)
→ N(n3,e)

Each arrow is a cut-based abstraction.

Key intuition (important)

A K-feasible cut limits how much logic you “see at once” so optimization and decomposition stay tractable.

This is why cuts are central to:

rewriting

technology mapping

decomposition

DAG sharing

If you want, next I can show:

how Boolean functions are computed from a cut

or how ABC enumerates cuts efficiently (without exponential blowup)

what is e? one lineCandidate cut = { n1, n2, e }

e is a primary input (PI) that directly feeds node N and must be included as a leaf because paths e → N bypass n1 and n2.

great yes can you show me how boolean functions are computed from a cut and how abc enumerates cuts? short res for each

1. How the Boolean function is computed from a cut (short)
   Given a cut L = {l₁,…,l_k} of node n, ABC treats each leaf as a variable and symbolically evaluates the cone from the leaves to n (using AIG ops or truth tables) to get

2. How ABC enumerates cuts (short)
   ABC does bottom-up cut enumeration:

PIs start with the trivial cut {PI}.

For an AND node with fanins u,v, merge cuts of u and v.

Keep unions whose size ≤ K (K-feasible).

Prune dominated cuts (supersets with no benefit) and keep only a small best set per node.
