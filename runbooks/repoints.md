# Runbook: source repoints

How the registry changes the `source` repository a listing points at.
Companion to removals.md and seeding.md. Repoints carry the most
judgment of the three: a source URL is the listing's identity claim,
and since OIDC trusted publishing the repository id behind it is the
credential that authorizes releases. Wrong repoints are how a name gets
hijacked; the process below is deliberately conservative.

## Principles

- **Who owns the act depends on tier.** Tier 0 listings are
  scanner-owned: the registry repoints them as a registry act. Claimed
  listings (tier 1+) belong to their maintainer: the registry never
  rewrites their source; the maintainer changes it by pull request, or
  the registry flags and waits. Automated rename detection follows the
  same split (enrich auto-canonicalizes tier 0 renames and only
  records `renamed-to` in metrics for claimed entries).
- **A repoint is the human checkpoint for publishing identity.** The
  publish service refuses on id-match/name-mismatch until the listing
  is repointed, by design: every repository rename or transfer gets
  exactly one human look. A repoint that updates the URL but not the
  recorded `source-repo-id` leaves tokenless publishing broken or,
  worse, anchored to the wrong repository.
- **Classify, then decide; never repoint on a URL's say-so.** The
  crosscheck taxonomy is the vocabulary: same-owner rename, owner
  alias, shared history, independent, directory-dead. Only some of
  these are repoint material, and shared history cuts both directions
  (a successor organization keeping its fork is a KEEP, not a
  repoint).

## The two routine shapes

**1. Rename or transfer of the same repository** (GitHub 301s the old
name forever, so only the canonical-URL check ever notices; a move
once stayed invisible for months). Tier 0: enrich fixes these
automatically. Claimed: the maintainer confirms, or the
`renamed-to` flag in metrics is the prompt to ask them.

**2. Requested repoint with evidence** (the listing points at a copy,
a dead fork, or a predecessor). Evidence that has carried a repoint,
from the filter_fontawesome precedent (camp-index#149): the old
moodle.org directory listed the requested repository; the requested
repository is a true fork of the original author's; the currently
listed repository is an unrelated copy, dead for years; the requester
is the requested repository's top committer. No single item decides;
the picture does.

## Verify before acting

1. **Read the whole thread**; establish which shape this is. A
   "repoint" from someone with no connection to either repository is a
   name dispute (NAMESPACE.md), not a repoint.
2. **Tier check.** Claimed listing: the maintainer's PR is the vehicle;
   the operator's job is at most to verify and comment. Tier 0:
   continue.
3. **Build the classification picture:** are the two repositories the
   same party (rename/alias)? Does one derive from the other
   (shared-history probe)? Is the currently listed one alive? What did
   the old directory list? Who commits to the requested one?
4. **Check both repositories' ledger records** for markers (opted-out,
   name-collision, copy) that contradict the requested direction.

## Execute

1. Tier 0: edit the entry's `source` to the new URL as a registry
   commit. Claimed: the maintainer's PR carries this edit; the
   registry side is steps 2 to 4 after their merge. Nothing else in
   the entry changes by hand either way.
2. Refresh the observed data so nothing stale survives the move:

       camp refresh-metrics . <component>

   This re-fetches metrics from the new source and keeps the recorded
   repository id current. Standing rule since camp-tools#9: run it
   after every repoint merge, no exceptions.
3. For claimed entries whose repoint arrived by maintainer PR:
   `camp fill-repo-ids . <component>` re-resolves the OIDC identity
   anchor against the new source (the overwrite case is exactly what
   the named-component form is for).
4. Validate the entry, commit as the registry identity referencing the
   issue, push (mid-scan push rule applies), and verify the plugin
   page shows the new source.

## Close

State the evidence that carried the decision, plainly and completely;
repoints are the closes most likely to be re-read in a later dispute.
If the requester is the plugin's author, point at claiming.

## Escalate

- Any claimed listing where the maintainer is not the requester.
- Shared-history cases: derivation alone does not decide direction
  (successor organizations legitimately hold forks; the shared-history
  review queue exists for exactly this).
- Both repositories alive and independent: that is a collision under
  NAMESPACE.md, board territory.
- Evidence pictures weaker than the #149 precedent.
- Anything where the repoint would move a name away from a responsive
  party without their word.

## Precedents

- filter_fontawesome (camp-index#149): the evidence-picture standard
  for requested repoints, executed with refresh-metrics after.
- logstore_xapi: the months-invisible rename that motivated canonical
  URL checking in enrich.
- local_mail: canonical repository shadowed by a copy listing; fixed
  as a repoint plus refresh-metrics (the origin of the standing rule).
- Open LMS cluster (shared-history review): fourteen in-house platform
  variants of community plugins, verdict KEEP; shared history is not
  itself repoint evidence.
- webservice_restful (#52): when the parties agree the name moves, the
  clean mechanism was removal plus re-seed rather than a repoint;
  consider it for full handovers.
