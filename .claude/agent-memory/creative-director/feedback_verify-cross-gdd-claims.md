---
name: feedback-verify-cross-gdd-claims
description: Specialist "missing signal/schema/dependency" blockers may be stale — verify against the actual file before ruling
metadata:
  type: feedback
---

When a specialist (systems-designer, qa-lead, etc.) flags a blocker of the form
"depends on X which doesn't exist" or "schema/signal Y is missing/undefined,"
verify it against the actual file on disk before accepting it into the verdict.

**Why:** Specialists review the target GDD in isolation and infer the state of
upstream/downstream GDDs from references, not from reading them. Some of these
claims are stale or partially wrong. Adjudicating on a false "missing dependency"
either rejects a sound design or lets the author waste a revision cycle.

**How to apply:** Before finalizing a synthesis verdict, grep/read the named
upstream file. Distinguish three outcomes: (a) genuinely missing — blocker stands;
(b) exists but owned elsewhere — reframe who must act; (c) exists and satisfies the
claim — drop the blocker. Example from GDD #16 review: `get_nested` genuinely
absent from GDD #4 (blocker stands); GDD #22 absent but it only owns the delta
*value table* — the corruption *mechanism* is fully in GDD #4 §4.1 (reframe, not
block the whole pillar). See [[feedback-pillar-vs-engineering-blockers]].
