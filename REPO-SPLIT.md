# Repository split plan

This monorepo is a development convenience (see README, "starts as a
monorepo… designed to split"). This document is the concrete plan to break
it into the multi-repo topology the RFC's architecture implies, and the
exact code/config changes each split requires. Execute it when the GitHub
org is created — that is the natural moment, because `index/` then wants to
be the public PR target, `tool_camp` wants its own installable repo, and
`tools/` wants its own release stream.

Nothing here is required to keep developing; the monorepo works as-is. This
is a checklist for the day you go public.

## Target topology

```
<org>/camp-index          the registry data — public, PR-driven, mirror-cloned
<org>/camp-tools          the `camp` CLI (Python) — versioned, pip-installable
<org>/moodle-tool_camp    the Moodle client plugin (PHP) — installable, self-listed
<org>/camp-docs           (optional) RFC, DESIGN, MIRRORING, governance, policies
```

Rationale for the boundaries:

- **camp-index** must be small and fast to clone (RFC §4.1: "mirroring the
  registry's brain is `git clone`"), and it receives thousands of author
  PRs — it cannot be entangled with tooling code or test fixtures.
- **camp-tools** is versioned software with its own release cadence,
  installed by the index CI and by authors. Standard to separate a tool
  from the data it operates on.
- **moodle-tool_camp** *must* be its own repo: it is a Moodle plugin, so its
  `version.php` has to sit at the repo root, it follows the
  `moodle-<component>` naming convention, and it gets listed in camp's own
  index (dogfooding).
- **camp-docs** is optional; the docs can also just ride along in
  camp-index.

## What moves where

| Current path | Destination repo | Notes |
|---|---|---|
| `index/plugins/`, `index/discovery/` | **camp-index** | becomes the repo root `plugins/`, `discovery/` |
| `.github/workflows/index-pr.yml` | **camp-index** | CI that validates PRs |
| `templates/author-release.yml` | **camp-index** | authors copy it; lives with the thing it targets |
| `docker-compose.yml`, `docker/`, `Makefile` | **camp-index** | local preview / reference deployment of the rendered index |
| `schema/` | **camp-tools** | becomes package data (see change T1 below) — the schemas are the code's contract, not registry data |
| `tools/camp/`, `tools/tests/`, `tools/experiments/`, `tools/pyproject.toml` | **camp-tools** | becomes the repo root |
| `client/tool_camp/*` | **moodle-tool_camp** | contents become the repo root |
| `docs/` | **camp-docs** or **camp-index** | design notes, mirror guide |
| `dist/` | — | generated; stays gitignored, never moves |

## The only real code change: schema loading (T1)

Everything else is moving files and updating references. The one genuine
code coupling is how the tools locate the schemas:

`tools/camp/validate.py:11`
```python
SCHEMA_DIR = Path(__file__).resolve().parent.parent.parent / "schema"
```

This reaches *up and out* of the `tools/` tree to `camp/schema/` — which
breaks the moment `schema/` and `tools/` are in different repos. Fix by
folding the schemas into the tools package as package data:

1. `git mv schema/*.json tools/camp/schema/` (schemas live inside the package)
2. change the line to:
   ```python
   SCHEMA_DIR = Path(__file__).resolve().parent / "schema"
   ```
3. in `pyproject.toml`, ship the JSON:
   ```toml
   [tool.setuptools.package-data]
   camp = ["schema/*.json"]
   ```

`advisory.py` imports `SCHEMA_DIR` from `validate` and needs no further
change; `site.py`/`ingest.py` call validate's functions, not the constant.
After this, the index repo needs no `schema/` directory at all — running
`camp validate` uses the tool's bundled schemas.

## Cross-repo references to update

| File | Change |
|---|---|
| `camp-index/.github/workflows/index-pr.yml` | `pip install ./tools` → `pip install camp-tools` (from PyPI) or `pip install "git+https://github.com/<org>/camp-tools@<tag>"` — two occurrences |
| `camp-index/Makefile` | `CAMP ?= tools/.venv/bin/camp` → `CAMP ?= camp` (installed on PATH) |
| `camp-index/templates/author-release.yml` | set `CAMP_INDEX_REPO: <org>/camp-index`; the `pip install ./index-repo/tools` step becomes `pip install camp-tools` |
| `camp-tools/*/schema/*.json` `$id` | replace `https://camp.invalid/schema/...` with the real published schema URLs |
| `moodle-tool_camp/` | add a standard `moodle-plugin-ci` workflow (reuse the one from any listed plugin as a template) |
| all `README`/`DESIGN` sibling-directory links | become inter-repo links |

## Preserving history

To carry each subtree's git history into its new repo (rather than starting
fresh), use `git subtree split` or `git-filter-repo` per directory:

```bash
# example for the tools repo
git subtree split --prefix=tools -b split-tools
# then push split-tools as the main branch of the new camp-tools repo
```

Do the `schema/` move (T1) *before* splitting, so the schema history lands
in camp-tools with the rest of the package. If preserving history is not
worth the effort, a clean initial commit per repo is fine — this is
pre-launch.

## Suggested order

1. Create the org; decide the final project name (the trademark/name pass in
   RFC §8 gates the public name, and the schema `$id` URLs and repo names all
   bake it in — do it once).
2. **camp-tools first**: apply T1 (schema move), split with history, publish
   a tagged release (and to PyPI if desired). Everything else depends on it.
3. **camp-index**: split; drop `tools/` and `schema/`; point CI and Makefile
   at the published camp-tools; open it for PRs.
4. **moodle-tool_camp**: extract `client/tool_camp/` to root; add its CI;
   submit it to camp-index as its own first Tier 2 (source-verified) listing.
5. **camp-docs** (optional): move `docs/` + the RFC.
6. Sweep all cross-repo references (table above) and the schema `$id` URLs.

## What does NOT change

- The `camp` CLI's commands and behavior — `validate`, `scan`, `verify`,
  `site`, `composer`, etc. all already take an explicit `index_dir`
  argument, so pointing the tool at a *separate* index checkout works today.
  That argument is the seam the whole split rides on; it is why the monorepo
  can become multi-repo cheaply.
- The index entry / listing / advisory formats.
- The trust model, CI check list, or client plugin behavior.
