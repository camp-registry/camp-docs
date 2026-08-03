# Runbook: seeding requests

How the registry adds a listing for a plugin discovery has not found,
or has previously rejected. Companion to removals.md; same contract:
the routine path is executable by any operator, and every judgment
call is an explicit stop.

Requests arrive as issues on camp-index via the seed-request form.
Seeding also happens registry-initiated, from the crosscheck missing
worklist and from `camp dependency-report` (a dependency declared by a
listed plugin but absent from the index arrives with built-in evidence
of relevance; components Moodle bundles are excluded automatically).

## Principles

- **Seeding creates a Tier 0 discovered listing, nothing more.** It
  records that a public repository carries a component name; per
  NAMESPACE.md, it grants nothing. The close comment always points the
  author at claiming.
- **The scanner is the gate, not the operator.** A seed is a targeted
  run of the same discovery pipeline as the monthly sweep: version.php
  must parse, the license must be GPL-compatible (file or header), the
  repository name must plausibly match the declared component. An
  operator never hand-writes an entry file to get around a scanner
  refusal.
- **There is no repository age or popularity bar.** Nine-day-old repos
  with zero stars have been seeded (the OER exchange suite,
  camp-index#118 through #124). Activity signals are advisory data for
  verification priority, not admission criteria.

## Verify before acting

1. **Read the whole thread.** Some "please add my plugin" requests are
   really claims (plugin already listed), repoints (listed under the
   wrong repo), or collisions (name already held by someone else).
2. **The component is not already listed.** If it is, and the request
   points at a different repository, stop: that is a repoint or a name
   dispute, not a seed (see repoints.md and NAMESPACE.md).
3. **Check the scan ledger for the repository.** An `opted-out` marker
   means the author previously asked for removal; a seed request from
   that same author lifts the marker, no questions asked (delete the
   ledger record before scanning, and say so in the close). An
   opted-out marker plus a request from someone else escalates.
4. **The repository is public and is the plugin's canonical home** as
   far as the requester represents it. The scanner's checks handle the
   rest.

## Execute

From a current camp-index checkout, with camp-tools on PYTHONPATH:

    camp scan . --query "repo:OWNER/NAME" --recheck-days 0

`--recheck-days 0` forces re-evaluation of a repository the ledger has
seen before (the common case for requests: the sweep already rejected
it once, usually for a license GitHub could not classify). GitLab
sources use `camp scan-gitlab` with the project path as the term.

Read the outcome the scanner records:

- **written:** the listing exists. Proceed to commit.
- **no-version-php / bad-license:** the repository does not currently
  qualify. Reply with the specific reason and what fixes it (add a
  license file or GPL header; put version.php at the repo root); the
  request can be re-run after they push. Do not override.
- **needs-review:** the repo name does not correspond to the declared
  component. A human decides (RFC §8); escalate with the details.
- **exists / copy / name-collision:** the component is already held.
  This is not a seed; see step 2.

Then commit as the registry identity, push (respecting the mid-scan
push rule in removals.md), and confirm the plugin page renders.

## Close

Point at the new listing and at the claim path (AUTHORS.md step 1);
Tier 0 is an invitation, and requesters are usually the maintainer.
If the scanner refused, the close states the reason and the fix, and
invites a fresh request after the push.

## Escalate

- needs-review outcomes (name/component mismatch).
- Any already-listed component (repoint or dispute in disguise).
- An opted-out marker for the repository when the requester is not the
  author who opted out.
- Bulk seeding (whole-org or list imports): batch admission is a
  registry-lead decision, not an operator act. Precedent: a 203-repo
  eligible batch was deliberately held rather than auto-admitted.

## Precedents

- Adam Jenkins' three requests (camp-index#84 to #86) set the shape:
  verify, targeted scan, commit, close with claim pointer.
- OER exchange suite (#118 to #124): seven form-validated requests in
  one pass; new names, young repos, no age policy invented.
- block_course_checker_info (#147): seeded straight off the crosscheck
  missing worklist when the requester surfaced it.
- webservice_restful (camp-index#52): re-seed from a successor
  repository immediately after a removal freed the name; two recorded
  acts, and the removal did not imply the re-seed.
- tool_imageoptimize / tool_imageoptimizer (#84 area, #87): seeding a
  successor and removing the stale predecessor are separate requests,
  each verified on its own.
