---
name: causal-entropic-optionality
description: >
  Decision-making procedure based on Wissner-Gross & Freer's "Causal Entropic
  Forces" (Phys. Rev. Lett. 110, 168702, 2013) — select actions by maximizing
  the diversity of future reachable paths/states over a time horizon, rather
  than by maximizing expected value directly. Use when designing or executing
  an action-selection step (trading decisions, agent planning, game/strategy
  AI, resource allocation) where the user wants to "maximize future
  optionality", "keep options open", "find the path that preserves the most
  future freedom of action", or explicitly invokes "causal entropic forcing",
  "Wissner-Gross's equation for intelligence", or "entropic force" decision
  making. Not a replacement for domain risk limits or hard constraints — it's
  an objective/heuristic layered on top of them.
---

# Causal Entropic Optionality

## Core idea

Wissner-Gross & Freer showed that a simple physical rule — push the system
toward whichever present state keeps the **largest diversity of future
paths open**, evaluated out to a horizon τ — spontaneously produces
sophisticated-looking adaptive behavior (centering in a box, balancing an
inverted pendulum, tool use, cooperation) with no explicit goal other than
"maximize reachable future diversity."

Key correction to a common misreading: the force does not search for "tools"
or any other category directly. Tool use and cooperation *emerged* as
side effects of maximizing path diversity — because grabbing a tool or
cooperating happened to unlock more future states than not doing so. Apply
the same discipline here: score candidates by how much future diversity they
unlock, never by whether they look like "finding a tool."

Formally: `F(X₀, τ) = T_c ∇_X S_c(X, τ)|_{X₀}`, where `S_c` is the path
entropy (log-diversity of accessible future paths) of macrostate X over
horizon τ, and `T_c` is a strength parameter. Two free parameters only:
horizon τ (how far ahead to look) and strength T_c (how much this objective
dominates vs. other goals/constraints). The translation to an LLM decision
procedure below maps τ → "how many steps deep to roll forward" and T_c →
"how much weight this optionality score gets vs. direct expected value or
hard constraints."

## The procedure (run at each decision point)

This is a per-decision procedure, not a persistent background loop — re-run
it fresh every time a decision point is reached. There are two ways to
execute it; pick based on what's available.

1. **Enumerate candidate actions** at the current state. Bounded, not
   exhaustive — use judgment on how many are worth considering (typically
   3-7; fewer for cheap/fast decisions, more for high-stakes/low-frequency
   ones).
2. **Roll forward τ steps per candidate** — for each candidate action, work
   out what states/further actions become reachable next, across a few
   plausible branches (not just the expected branch). Depth and branching
   are also judgment calls: deeper/wider for high-stakes or slow-changing
   domains, shallow for fast/cheap ones.
3. **Score each candidate by reachable-future diversity**, not by expected
   value of that single candidate. A candidate that forecloses future
   branches (even if its direct expected value looks good) scores worse
   than one that keeps many viable continuations open.
4. **Apply hard constraints** (risk limits, budget, domain rules, safety
   bounds) as filters before or after scoring — these are not part of the
   entropic force itself and always take precedence over the optionality
   score.
5. **Pick the action maximizing future diversity** subject to those
   constraints, and act.
6. **Repeat fresh at the next decision point.** Don't treat step 5's winner
   as locked in for future steps — re-enumerate and re-score as new
   information arrives.

## Execution mode A — multi-agent (when agent infrastructure is available)

When the host system can actually spawn parallel workers (the Agent tool, a
Workflow, or the target system's own simulator/backtester), use real
execution instead of imagined rollouts:

- Treat each candidate action from step 1 as a branch to actually simulate.
- Dispatch one agent/worker per candidate (or per candidate × scenario) to
  carry the rollout forward τ steps and report back what further
  actions/states became reachable.
- Compare the returned branch reports and score as in step 3.
- This scales to wider enumeration and deeper rollouts than a single LLM
  turn can hold in context, at the cost of actually running the work —
  budget it accordingly (time, API cost, compute).

## Execution mode B — single-LLM virtual rollout (default, no infrastructure)

When there's no agent/simulator to dispatch to — the common case when this
skill fires mid-conversation — do the rollout as an explicit internal
thought experiment instead of actually executing anything:

- For each candidate, write out (even briefly) the 2-3 most plausible next
  branches and what they in turn open up or foreclose.
- This must be done per candidate, not skipped in favor of intuition — the
  value of the method is in comparing enumerated futures, not in a vibes-based
  diversity guess.
- Keep breadth × depth small enough to stay legible (judgment call, no
  fixed cap) — if the enumeration is getting too large to reason about
  clearly, that's a signal to either narrow the candidate set or switch to
  Mode A instead of pushing through with a sloppier rollout.
- Conclude with an explicit comparison across candidates before picking.

## Applying this to a concrete domain (e.g., trading)

The skill is domain-general; the domain supplies the state space, the
candidate actions, and the constraints:

- **Candidate actions**: whatever moves are legal at this decision point
  (enter/exit a position, hedge, resize, hold, rebalance, etc.)
- **"Future diversity"**: breadth of viable follow-up positions/strategies
  reachable from each candidate, given plausible market branches — not raw
  expected return.
- **Hard constraints**: risk limits, capital constraints, regulatory rules —
  always filter/override the optionality score, never compete with it as
  just another scored factor.
- A hedge or diversifying trade may score well under this method precisely
  because it preserves more future viable paths, not because "hedging" is
  specially favored — don't hardcode domain tactics into the scoring.
