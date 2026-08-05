# Registry runbooks

The procedures for the registry's routine operations. Each runbook is
self-contained: what qualifies, how to verify before acting, the exact
commands, and an explicit list of what escalates to the registry lead
instead of being decided in the runbook.

If you are new to registry operations, read a runbook fully before
working an item of its kind, and treat the escalation lists as hard
boundaries: they mark the judgments this project has decided belong to
the lead, usually because a precedent was expensive to establish.

- **removals.md** — listings removed at their maintainer's request:
  qualification shapes, ownership verification, the opt-out sweep, and
  reversal.
- **seeding.md** — adding listings for plugins discovery has not
  admitted: the scanner is the gate, never hand-write entries past a
  refusal.
- **repoints.md** — changing the repository a listing points at: the
  most judgment-heavy act, since a source URL is a listing's identity
  claim and its publishing credential.
- **release-train.md** — how a camp-tools change reaches production:
  tags, pins, packaging checks, publish, release notes.

The tooling is built so that policy lives in commands, not in operator
memory: when a runbook and the CLI disagree, that is a bug in one of
them; file it.
