# Runbook: the code-check store

Where the per-release code-check summaries live and how they are kept
current (camp-tools#79, camp-index#532). The summaries feed the "Code
check" column, the chips and the badge JSON; they are a report, never a
gate (DESIGN.md D23).

## Where

`checks/<component>.json` in camp-index, one document per released
plugin, bot-written like `metrics/`. Each version's summary records the
checker versions it was computed with under `facets` (`code` for php -l
and phpcs, `amd` for the AMD file-set, staleness and rebuild verdicts).
Publish copies the store into `dist/checks`, so `/checks/<component>.json`
on the site is the store as of that publish.

## Who writes it

- The scheduled **Checks refresh** workflow (02:41 UTC daily, or by
  dispatch) runs `camp checks-refresh . --budget N`: summaries that are
  missing first, then those whose facet is behind its checker version,
  stalest document first, up to the budget (60 by default). Documents the
  store lacks but the live site has are imported, not recomputed.
  Documents for entries that no longer have releases are removed.
- **Publish** computes only a brand-new release the store does not know
  yet, so a release's check line appears with the release; the next
  refresh imports that summary into the store. Publish never writes the
  store and never recomputes the archive.

## After a checker bump in camp-tools

Bump only the facet that changed (`CODE_VERSION` or `AMD_VERSION` in
`camp/checks.py`). Nothing happens at publish; the refresh job drains the
stale facet over the following nights at the budget, each run committing
what it did. To drain faster, dispatch the workflow with an empty budget
(everything pending) or a larger one; a cancelled run loses only the
releases it had not committed. The page may show one facet from before
the bump and one from after until the drain completes; each summary says
which.

## Checks

- Pending work: `camp checks-refresh . --budget 0` prints the counts and
  computes nothing.
- A document that looks wrong: delete it from `checks/` and let the next
  refresh recompute it (or import it from the live site if that copy is
  right).
