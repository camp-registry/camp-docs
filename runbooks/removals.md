# Runbook: listing removal requests

How the registry handles a maintainer asking for their plugin's listing
to be removed. This operationalizes the RFC §4.4 no-questions-asked
removal promise and NAMESPACE.md's discovery-is-not-reservation
principle. It is written so that any operator, human or automated, can
execute the routine path and knows exactly when to stop and escalate.

Requests arrive as issues on camp-index via the removal-request form
(title "Removal request: ..."). Nineteen removals were processed under
this process before it was written down; the special cases below are
that case law.

## What qualifies

Only **unclaimed Tier 0 listings with no releases** are removed
outright. The other shapes are different requests:

- **Claimed (tier 1+):** the entry is the maintainer's own file. They
  edit or delist it by pull request; point them at their entry and
  close. The registry does not edit claimed entries on request.
- **Has releases:** published history is never deleted (append-only
  ledger, RFC §4.2). The available act is `status: delisted`, which
  keeps the archive intact and stops serving the listing. This is a
  registry act with its own judgment; escalate.
- **Not listed at all:** nothing to do; say so and close.

`camp opt-out` enforces all three refusals itself. Do not work around
a refusal by editing files directly.

## Verify before acting

1. **Read the whole issue thread**, not just the title. Confirm what is
   actually being asked; a "removal" is sometimes a rename, a
   supersession note, or a source repoint in disguise.
2. **The requester controls the source repository.** Compare the issue
   author's account to the listing's `source` owner. Same account is a
   pass. An organization repo needs the requester to be a visible
   member of the org (precedent: mod_readseed, where the org itself had
   already deleted the repo). Anything murkier, escalate.
3. **Read the entry file**: tier 0, `releases: []`, source matches the
   repository the requester controls.
4. **Sanity-check the motive when the plugin looks alive** (stars,
   recent activity). Removal of an actively-used plugin is still
   honored, but state the facts in the close comment for the record
   (precedent: quizaccess_safeexambrowser, removed as superseded by
   core with a for-the-record note).

## Execute

From a current camp-index checkout, with camp-tools on PYTHONPATH:

    camp opt-out . <component> --reason "camp-index#<issue>"

What the tool does: deletes the listing file, writes a permanent
`opted-out` marker keyed by the source repository (discovery never
re-evaluates an opted-out repo), and sweeps the scan ledger for other
records carrying the same component:

- **Same owner** (the requester's own variant/mirror repos, ledgered as
  copy/exists/needs-review): these inherit the opt-out automatically.
  The request covers the author's plugin wherever their own
  repositories carry it (precedent: theme_dennis and its Moodle 4.4
  variant repo, camp-index#171).
- **Different owner:** reported in the output and left alone. Removal
  frees the name; it does not adjudicate who may claim it next
  (NAMESPACE.md governs what happens if someone does). Read these
  lines and use judgment: a copy-farm account poised to inherit a
  freed name is worth escalating before it happens.

Then:

5. **Timing rule:** never push a ledger-touching commit while a
   discovery scan is running (the scan pushes the ledger at the end of
   a multi-hour sweep; a mid-flight rebase conflict kills the run).
   Check the scan workflow's recent runs first. Scans are monthly on
   the 3rd, plus manual dispatches.
6. Commit as the registry identity
   (`camp project <noreply@camp-registry.org>`), message referencing
   the issue, and push. The publish workflow runs on plugins/ changes
   and removes the page.
7. Verify the plugin page returns 404 on camp-registry.org.

## Close

Comment and close the issue. Plain register: confirm the removal, note
the permanent marker, and state the reversal path - a seed request
lifts the marker, no questions asked in either direction. If the sweep
touched variant repos or flagged other owners' records, say so plainly.

## Escalate to the registry lead instead of proceeding

- Ownership unclear, or requester is not the listed source's owner.
- The entry is claimed or has releases (see shapes above).
- A different-owner ledger record would inherit a freed name you would
  not hand it (judgment call, goes to a human).
- The request is really a dispute (two parties, one name): that is
  NAMESPACE.md territory, never a removal.
- Anything the tools refuse for a reason not listed here.

## Related precedents not covered above

- **Name handover after removal** (webservice_restful, camp-index#52):
  removal plus immediate re-seed from the successor repository via a
  targeted scan. Two acts, both recorded; the removal does not imply
  the re-seed.
- **Bundled subplugins** (camp-index#130/#131): subplugins that ship
  inside a parent plugin's releases do not carry their own listings;
  removal on those grounds cites the precedent rather than the
  maintainer-request rationale.
