# The Locked Archive – Escape Room Solver

**An AI agent that plans its way out of an escape room**, modelled twice with two formalisms:
1. **Classical planning in PDDL**, solved with **Fast Downward**, comparing blind search with the hFF and hadd heuristics.
2. **Situation Calculus and IndiGolog**, with online execution, **reactive controllers** and **exogenous events**.

Individual project for the *Planning and Reasoning* course, Master of Science in Engineering in Computer Science, Sapienza University of Rome.

📊 [Slides (PDF)](docs/presentation.pdf) · 📈 [Full results](RESULTS.md) · 🧾 [IndiGolog execution transcripts](indigolog_transcripts.md)

---

## The domain

An agent is trapped in a multi-room archive. To reach the exit it must **explore rooms, collect items, read clues to learn door codes, combine items into keys, and unlock doors**.

| Instance | Rooms | What makes it interesting |
|---|---|---|
| Easy | 3 | One key lock and one code lock (sanity check) |
| Medium | 5 | A **mandatory combine** (`item_x + item_y → key`) to reach the exit |
| Hard | 8 | A hub with two branches: a **cheap west path** that needs a combine puzzle (door cost 3) and an **expensive east path** with a decoy (door cost 5). The cost-optimal plan is hidden behind the puzzle |

Room maps are in [`slides/maps/`](slides/maps).

## Part 1: PDDL with Fast Downward

- **`domain_adl.pddl`**: `combine` uses a **conditional effect** (`forall`/`when`) and an **existential precondition** to decide which item is produced
- **`domain_strips.pddl`**: the same action fully grounded, so that the delete-relaxation heuristics (hFF, hadd), which do not support ADL, can be used
- Action costs through `:action-costs`, with a **different `move-cost` per door**

### Results (all 9 runs executed)

| Instance | Config | Plan length | Plan cost | Expanded states |
|---|---|---:|---:|---:|
| Easy | Blind | 6 | 2 | 9 |
| Easy | hFF | 6 | 2 | 7 |
| Easy | hadd | 6 | 2 | 7 |
| Medium | Blind | 14 | **4** | 38 |
| Medium | hFF | 14 | **4** | 19 |
| Medium | hadd | 16 | **6** | 18 |
| Hard | Blind | 17 | **13** | 169 |
| Hard | hFF | 12 | **16** | 15 |
| Hard | hadd | 12 | **16** | 15 |

**Key finding: inadmissible heuristics trade optimality for speed.** On the hard instance, blind search finds the optimal west-branch plan (cost 13) but expands 169 states. hFF and hadd expand about **10 times fewer states** but settle for the east branch: fewer actions (12 vs. 17), higher cost (16). The same effect appears with hadd on the medium instance.

```bash
./fast-downward.py pddl/domain_adl.pddl    pddl/problems/problem_hard.pddl        --search "astar(blind())"
./fast-downward.py pddl/domain_strips.pddl pddl/problems/problem_hard_strips.pddl --search "astar(ff())"
./fast-downward.py pddl/domain_strips.pddl pddl/problems/problem_hard_strips.pddl --search "astar(add())"
```

## Part 2: Situation Calculus and IndiGolog

The same domain as a **basic action theory** (fluents, preconditions, successor-state axioms) for the `indigolog_plain.pl` interpreter.

**Two controllers**
- **Simple Controller**: one offline `search/1` over a nondeterministic program, then execution
- **Reactive Controller**: `prioritized_interrupts/1` re-plans from the current situation whenever the world changes. It adapts to two exogenous events without any hand-written special cases:
  - `hint_revealed`: a code becomes known, so the agent skips reading the clue
  - `door_jams`: a door becomes blocked, so the agent re-routes through the other branch

**Reasoning tasks** (`reasoning_tasks.pl`), using the interpreter's own predicates:
- **Legality**: execute a sequence and reject illegal ones (for example, unlocking before picking up the key)
- **Projection**: query a fluent after a sequence with `holds/2`, without executing it
- **Regression**: check whether a goal is reachable from the current situation with `search/1`

**Design notes.** IndiGolog has no numeric cost metric, so on the hard instance the controllers may pick either branch: the west branch is cheaper in PDDL cost but needs more primitive actions. This difference between the two formalisms is discussed in [`RESULTS.md`](RESULTS.md). The results also document **8 interpreter pitfalls** found and fixed while running on the course VM (successor-state-axiom conventions, `exog_occurs/1` semantics, variable grounding before negation, and others).

**Running it**: copy the `indigolog/` folder into the IndiGolog distribution (for example `examples/locked_archive/`), then load the interpreter configuration followed by an instance:
```bash
swipl ../../config.pl main_hard.pl
```
```prolog
?- indigolog(control_reactive).
```

## Repository structure

```
pddl/
├── domain_adl.pddl, domain_strips.pddl
├── problems/              # easy / medium / hard (+ STRIPS variants)
└── experiment_logs/       # raw Fast Downward output for all 9 runs
indigolog/
├── escape_room_domain.pl  # action theory
├── controllers.pl         # simple + reactive controllers
├── instances/             # easy / medium / hard
├── main_*.pl              # entry points
├── reasoning_tasks.pl     # legality, projection, regression
└── demo_exog_harness.pl   # scripted exogenous events for the live demo
slides/maps/               # room layouts (SVG)
docs/presentation.pdf
RESULTS.md                 # full results and discussion
indigolog_transcripts.md   # raw execution transcripts
```

## Author

**Bogdan Andrei Tutuianu** – [GitHub](https://github.com/bogX2)
