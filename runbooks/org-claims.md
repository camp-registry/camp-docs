# Runbook: organization claims

Claiming every listed plugin an organization owns, from its
`<org>/camp-claim` manifest (`camp-claim.yml` at the root; format in
AUTHORS.md "Claiming as an organization"). The registry runs the sweep;
the org's only artifact is the manifest. Requests arrive through the
**Organization claim request** issue form on camp-index.

## Qualification

- The manifest exists at `<org>/camp-claim/camp-claim.yml` and parses
  (`camp org-claim` validates and refuses with a specific message if
  not — relay that message to the requester).
- The requester can act for the organization: public membership, or the
  request comes from an account that owns/administers org repositories.
  Same standard as claim-PR auto-merge; anything less legible, verify
  by hand before running.
- Authorization is control of the org's repositories, so the sweep only
  ever touches entries whose `source` lives under that org — the tool
  enforces this; there is nothing to check per entry.

## Procedure (first sweep, on the request issue)

1. `camp org-claim INDEX <org> --dry-run` — read the report before
   writing anything. Sanity-check the claimed count against
   expectations from the issue.
2. Review conflicts (entries already claimed individually). The sweep
   never touches them. If the manifest and the existing claim disagree,
   that is a human conversation on the request issue, not an overwrite
   in either direction.
3. Run without `--dry-run`. The command claims each entry (maintainers,
   security-contact, labels, tier 1, canonical key order, an
   `org-claim: <org>` stamp) and anchors `source-repo-id` for the
   claimed set.
4. `camp validate` the touched entries; commit as the project identity,
   one commit for the sweep, message referencing the request issue.
5. Enroll the org: add it under `orgs:` in `discovery/org-claims.yml`
   (enrolled date + request issue number), same commit or a second one.
6. Push (publish deploys), verify a couple of entry pages show Tier 1,
   reply on the issue with the claimed count, any conflict list, and
   that manifest edits now flow automatically; close.

## After enrollment: the watch

The org-claim-watch workflow re-runs the sweep daily for every enrolled
org, so manifest edits (maintainer changes, new security contact, label
fixes) reach that org's stamped entries without a new request. The
stamp is the safety boundary: the watch updates manifest-owned fields
(maintainers, security-contact, labels) on `org-claim: <org>` entries
only — tier and releases are never touched, individually claimed
entries are never touched, and un-claiming is never automatic. Anything
the sweep refuses to do lands in a dedup'd "Org claim watch: entries
needing review" issue; triage those against the escalation list below.
A newly listed plugin under an enrolled org (fresh discovery or seed)
is claimed by the next day's sweep automatically — by design.

## Escalate to the lead

- Conflicts where the manifest contradicts an existing individual claim.
- Orgs whose listed sources are split across owners (personal +  org
  repos): the sweep claims only the org-owned subset; whether the rest
  belongs in the manifest's org is a judgment call.
- Manifest labels that look wrong for what the plugins visibly are
  (label heuristics exist in CI, but a sweep bypasses per-PR review —
  spot-check a few entries).
- GitLab groups (not yet supported by the tool).

## Precedents

- catalyst: the request that motivated the mechanism (179 entries at
  build time, zero conflicts on dry-run).
