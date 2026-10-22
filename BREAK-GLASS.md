# BREAK-GLASS — the committed bypass spec (NEXTGEN-CI.md (d), red-team patch S7 / premise P4)

D2 (single required `gates`) survives **only** with this spec committed. Without it the
fail-closed stack is brittle and the honest move under incident pressure is an illegal
bypass; with it the estate freeze-on-typo path is closed. The ledger law applies to the
bypass itself: a bypass is a recorded event, not an absence of one.

## A ruleset bypass is legal iff ALL four hold

1. **Actor = repo-admin.** The bypass must be executed by a repo-admin actor. An actor
   without that role cannot bypass and must page instead.
2. **The squash message carries `gates-bypass=<bead-id>`.** The bead is the pre-existing
   incident/authorization record; the stamp links merge to cause. No bead id, no legal
   bypass.
3. **A bypass-reason artifact is uploaded.** The artifact holds the reason text, the
   observed `gates` failure evidence, and the timestamp. It rides the run's provenance
   convention (see README: `tests/runs/...` provenance law).
4. **A P1 bead files.** The bypass opens debt, not forgiveness: a P1 bead for the repair
   of whatever forced the bypass, filed at bypass time.

Any bypass missing any of the four is a falsifier (spec (f) 9): "a ruleset bypass with
no bead stamp or artifact" means the fail-closed stack is brittle and the break-glass
itself needs redesign — the week is spent on redesign, not on blame.

## The trip wire

**Two legal bypasses on one repo within 30 days trip a design-review bead.** Two exits
through the glass in a month is a statement that the contract, not the incident, is the
problem. The design-review bead re-opens the spec section that produced the brittle
surface — it is not closable by filing more bypass paperwork.

## What break-glass is NOT

- It does not retire the ledger: every merged-over-red commit still owes its markers and
  the merged ledger still recomputes via the compare API pinned to event shas (S6).
- It does not authorize branch protection edits as a workaround; a ruleset edit that
  removes the required `gates` name outside the migration law (ruleset edit + legacy
  emitter rename in the same commit) is itself a bypass-shaped event and carries the
  same four-part stamp or the same falsifier.
- It is not self-service for agents: filing the P1 bead and recording the stamp is
  scripted; executing the GitHub-side bypass is a repo-admin action (owner holds the
  seal on ruleset changes).

## Cheat sheet (what the squash message and receipts must look like)

```
title: fix(whatever): hotfix the thing

body of the squash message ...

gates-bypass=est-xxxxx
```

- [ ] actor checked: repo-admin
- [ ] `gates-bypass=<bead-id>` present in the squash message
- [ ] `bypass-reason` artifact uploaded on the merge commit's provenance path
- [ ] P1 bead filed (repair), linked from the incident bead

30-day window check: `gh api` the commit log for `gates-bypass=` on the repo; count ≥ 2
→ file the design-review bead before the next bypass is even discussed.
