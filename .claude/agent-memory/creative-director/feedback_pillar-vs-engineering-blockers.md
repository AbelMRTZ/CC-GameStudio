---
name: feedback-pillar-vs-engineering-blockers
description: Split review blockers into vision/pillar (I own) vs engineering correctness (defer to leads); reserve MAJOR REVISION for pillar contradictions
metadata:
  type: feedback
---

When synthesizing an adversarial GDD review, sort the blockers into two buckets
before assigning a verdict:
- **Pillar/vision blockers** — the design contradicts or fails to deliver a stated
  pillar or the Player Fantasy. These are mine to rule on and they are what should
  drive a MAJOR REVISION verdict.
- **Engineering-correctness blockers** — degenerate formula inputs, missing
  validation, undefined behavior, untestable ACs, missing upstream methods. Real
  and must be fixed, but they are NEEDS REVISION territory and belong to the
  systems-designer / lead programmer / qa-lead to resolve, not me.

**Why:** A pile of 20+ specialist blockers reads as catastrophic, but most are
often correctness nits on a structurally sound design. Treating them all as equal
weight leads to over-harsh verdicts and demoralizes the author. Conversely, a
single genuine pillar contradiction can be buried under engineering noise and get
missed. The verdict should be anchored to pillars, not to blocker count.

**How to apply:** State the verdict's rationale in terms of pillars. If the design
serves the pillars and the blockers are all engineering correctness, the verdict is
NEEDS REVISION (fixable), not MAJOR REVISION. Reserve MAJOR REVISION for when the
design as written would ship a pillar broken. See
[[feedback-verify-cross-gdd-claims]] and [[project-demons-of-dravaryn]].
