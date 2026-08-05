# Runbook: the camp-tools release train

How a camp-tools change reaches production. Companion to removals.md,
seeding.md and repoints.md. The registry's CI and the author templates
all pin an exact tag, so nothing ships until the train runs; equally,
nothing on main is live just because it merged.

## Principles

- **Tags are immutable.** A tag, once pushed, is never moved or
  deleted. A broken release gets a new patch version and the broken
  tag simply stays undeployed (v0.2.38 is the precedent: unusable when
  pip-installed, superseded by v0.2.39, never reused).
- **The pins are the deployment.** camp-index workflows and the author
  release templates install camp-tools by tag. Until the pin sweep
  lands, production runs the old version regardless of what main says.
- **Local tests cannot catch packaging.** Tests run from the source
  tree; a data file missing from package-data imports fine locally and
  breaks every pinned consumer (the v0.2.38 lesson). Hence the real
  pip-install check below, mandatory whenever the release adds or
  moves a packaged data file.

## Pre-flight

1. Full test suite green from the camp-tools source tree.
2. Real packaging check: pip install the source tree into a scratch
   virtualenv and, from outside the source tree, import the package,
   load every packaged data file the release touches, and run
   `--help` on any new CLI surface.
3. For changes with a live data surface (site rendering, scanner
   behavior), verify once against the real index locally before
   tagging.

## Ship

1. Bump `__version__` in camp/__init__.py; commit as `vX.Y.Z`; push.
2. Tag `vX.Y.Z`; push the tag.
3. Pin sweep in camp-index: grep for the previous tag first, then sed
   every occurrence to the new one. Counts drift as workflows and
   templates gain version references (the template freshness checks
   carry the literal too), so trust the grep, not a remembered number.
   Commit and push.
4. Publish: a push that touches plugins/ or advisories/ triggers it;
   a pins-only, templates-only or docs-only push does not (the paths
   filter), so dispatch publish.yml manually in that case.
5. Verify live: version.json reports the new version, and spot-check
   whatever surface the release changed.

## Record

1. GitHub Release on the tag: notes drafted from the commits and the
   issues closed since the previous tag, edited for reading, published
   with `gh release create`. camp-tools and moodle-tool_camp carry
   releases; camp-index and camp-docs do not (continuous data and
   documentation).
2. Close the shipped issues with evidence: what shipped, where it is
   verified, and any follow-ups split out.
