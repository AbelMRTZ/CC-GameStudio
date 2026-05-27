---
name: feedback-pre-vs-gate
description: Hard QA gate rule — branch selection and reputation logic tests must pass before Vertical Slice narrative content is authored for Kingdom 2
metadata:
  type: feedback
---

Branch selection and reputation formula unit tests (CA-NPC-004 through CA-NPC-010 in GDD #15) are a hard gate before Vertical Slice narrative content for Kingdom 2 is written.

**Why:** If these logic tests don't pass first, Kingdom 2 dialogue content would be authored on top of unverified branch selection and reputation mechanics — any logic bug found later would require content rewrite, not just code fix. Shift-left principle: catch logic before content is layered on top.

**How to apply:** When sprint planning for Vertical Slice begins, check that all CA-NPC-004 through CA-NPC-010 tests exist and pass. If they don't, block Kingdom 2 narrative authoring work — escalate to producer if schedule pressure arises.

Related: [[project-gdd15-acs]]
