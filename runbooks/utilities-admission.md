# Admitting a utility listing

Utilities (RFC §4.8, camp-docs#4) are curated: no scanner seeds them, and every admission is one deliberate registry act. The request arrives by issue form or PR.

## Qualification — all four, no discretion on the first two

1. **Public claims anchor.** A public repository whose control can prove a claim (`source-repo-id` recorded at admission). For closed-source tools the docs/issue-tracker repository qualifies and the entry must say so (`closed-source: true`).
2. **Monitorable distribution channel.** The source host is GitHub/GitLab, or the entry declares `release-channel: scheme:ref` with a scheme camp-tools implements. `camp validate` enforces this; if the scheme is missing, the answer is "not yet — adapter first", never a hand-maintained release row. Monitorable does not mean has-releases; a git-only tool qualifies with an absent release row.
3. **Established community footprint.** Directory-era listing, moodlehq stewardship, or comparable evidence of real use. When in doubt, escalate to the lead; the section is meant to hold tens of entries, not hundreds.
4. **Free tier for commercial tools.** Genuinely usable without payment. Labels carry the commercial facts (`freemium`, `paid-service`, etc.).

## Consent

Ask the maintainer before listing while the section is small. Their "yes" is consent to a curated listing, not a claim; the entry launches without `claimed:` and they claim by PR whenever they wish (same authorship proof as plugin claims; claim sets the `claimed:` date and maintainers).

## Mechanics

- Entry at `utilities/<slug>.yml`; run `camp validate` on it; fill `source-repo-id` at admission.
- Do not hand-write `metrics` beyond an initial `latest-release` if useful — the next enrich run overwrites the block either way.
- Closed-source entries: `license: Proprietary`, `closed-source: true`, no repo metrics expectations, release row only via a declared channel.

## Escalate to the lead

- Footprint judgment calls; anything that smells like a vendor seeking a directory listing.
- Slug disputes (NAMESPACE.md procedure).
- Requests to widen the category or install vocabularies, or to add a release-channel adapter (those are camp-tools changes).
- Removal requests follow the removals runbook, minus the tier/release considerations.
