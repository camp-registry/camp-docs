# Design decisions

Running log of implementation decisions and their rationale, referenced to
the RFC. Newest last.

## D1: Python for registry tooling, moodle-plugin-ci for plugin checks

Plugin-quality checks reuse `moodlehq/moodle-plugin-ci` unchanged — it is
what authors already run, so registry results match author CI results.

Registry tooling (validation, deterministic builds, verification, Composer
metadata, later signing) is Python. The deciding factor is RFC §4.3:
`python-tuf` is the TUF reference implementation and `sigstore-python` is
the mature Sigstore client — the exact stack PyPI uses. PHP has no
production repository-side TUF implementation, so PHP registry tooling
would mean reimplementing security infrastructure by hand.

## D2: The canonical artifact is defined by our packer, not `git archive`

`git archive --format=zip` output can vary across git versions, which would
silently break "anyone can rebuild the artifact byte-identically" (RFC §4.2,
Appendix A "reproducible build"). Instead, `camp build`:

1. takes the file set and contents from `git archive --format=tar` at the
   tagged commit (this honours the author's `.gitattributes export-ignore`
   rules), then
2. re-packs into a normalized ZIP: sorted paths, plugin-folder prefix, all
   timestamps = the commit's committer time (UTC), permissions normalized
   to 0644/0755, fixed deflate level, no extra fields.

Symlinks and special files are rejected outright — they have no safe
meaning in a ZIP Moodle will extract.

Consequence: repos without `export-ignore` rules ship dev files
(`.github/`, dotfiles) in their ZIPs, exactly as tagged. That is the
guarantee working as intended ("the artifact is exactly the public
source"); trimming is the author's call via `.gitattributes`, and the
planned author-side release Action will scaffold a sensible default.

## D3: Index entry files are named by full component name

`index/plugins/<plugintype>/<component>.yml`, e.g.
`index/plugins/mod/mod_cmi5launch.yml`. The type directory is technically
redundant with the component prefix, but keeps directory listings browsable
at scale (~2000 plugins); the full component name as filename makes results
of `grep -r mod_cmi5launch` unambiguous. The validator enforces placement.

## D4: Ledger immutability is enforced structurally in CI

RFC §4.2 says releases are never rewritten. Rather than trusting review,
`camp ledger-check` requires the base version's release list to be a
byte-identical prefix of the head version's — existing records cannot be
edited, reordered, or removed in any PR that passes CI. Revocation happens
via signed advisories (RFC §5.3), a separate object, not by editing history.

## D5: Registry CI runs the static check subset only

RFC §4.2 names "code checker, PHPLint, version.php sanity, privacy API
presence, basic security lint" as the trust floor. Full PHPUnit/Behat
matrices belong to the author's own CI (where mod_cmi5launch, for example,
already runs them). Registry PR latency matters: publishing must be minutes,
not the review-queue days the RFC positions itself against (§10, "instant CI
publishing vs. review queues").

## D6: Composer package vendor = first maintainer's GitHub handle

`davidpesce/moodle-mod_cmi5launch`, matching the RFC §6.1 example shape
`<vendor>/moodle-<component>`. Component names are the registry's authority;
Composer names are a projection of them. Plugins that already publish to
Packagist under a different name can add an explicit override field when the
need first arises (deliberately not speculatively designed now).

## D7: The registry targets Moodle 4.5 LTS through 5.2+

Confirmed 2026-07-10. The CI matrix pins the endpoints of that range
(MOODLE_405_STABLE + MOODLE_502_STABLE); intermediate branches are assumed
covered by the endpoints for the static trust-floor checks. The range's ends
map to the two integration channels: 4.5 sites use the client plugin
(RFC §6.2), 5.2+ Composer-managed sites can use the Composer channel
(RFC §6.1). Note this is the *registry's* tooling target — individual
plugins declare their own `supported-moodle` range in the ledger, which may
extend below 4.5 (mod_cmi5launch supports 4.3+, for example).

## D8: Tier 0 discovery is conservative by construction

The seeding scanner (`camp scan`, RFC §4.4) writes a discovered listing only
when a repository passes all of: not a fork (canonical source only), GPL-
family license detected by GitHub, and a root `version.php` whose
`$plugin->component` (or legacy `$module->component`) parses as valid
frankenstyle. Repo description becomes the entry's `summary`; discovery date
is recorded for provenance. Existing components are never overwritten
(first-registered wins, RFC §8). Consequences accepted: plugins in
subdirectories or monorepos are missed, and missing/unrecognized licenses
exclude some genuine plugins — both err on the side of not listing rather
than mislisting. Schema change: `labels` and `security-contact` are required
only from Tier 1 up, since discovered listings have no author declarations
yet.

## D9: TUF learnings from the Phase 2 spike

`tools/experiments/tuf_prototype.py` runs the full lifecycle over real camp
artifacts with python-tuf 7 / securesystemslib 1.4 (needs the
`securesystemslib[crypto]` extra): per-role ed25519 keys, targets covering
every dist file (ZIPs, packages.json, the website), sign → serialize →
verify-from-root → tamper-reject. Takeaways for the real design: the
metadata layer is genuinely small (the prototype is ~130 lines), the hashes
TUF signs are the same sha256 values the index ledger already records (no
double bookkeeping), and the hard parts are exactly where the RFC says they
are — key ceremonies, thresholds, expiry/re-signing cadence — not the code.
The production layout should sign `targets` over the artifact tree and let
snapshot/timestamp automate freshness, with root held offline at 2-of-N.

## D10: Heuristics and security lint are warn-only in CI

`camp lint-labels` and `camp audit` run on every index PR but never block
it. Rationale: the RFC frames heuristics as flagging "likely misdeclarations
for human attention" (§4.7), and legitimate plugins do everything the audit
looks for (VPL shells out to run student code; AI plugins store API keys).
Real-world calibration on the _MoodleDEV corpus: `local_ai_manager`
correctly suggests `external-account, paid-service` (API keys + Google
endpoints), simple blocks come back clean, and the three seeded Tier 1
plugins produce zero audit findings. Blocking on these signals would train
authors to game the patterns; surfacing them trains reviewers to read them.

## D11: Trusted publishing (OIDC) — design, pending GitHub infrastructure

The bootstrap release flow (templates/author-release.yml) authenticates
with a personal-access token, which is exactly the long-lived credential
RFC §4.3 says to eliminate. The replacement, implementable only once the
index lives on GitHub:

1. The author's release workflow requests a GitHub OIDC token and uses
   Sigstore keyless signing (Fulcio) to attest the built artifact — binding
   the artifact hash to the *workflow identity* (repo, ref, workflow path),
   with the attestation logged in Rekor automatically.
2. The release PR carries the attestation bundle alongside the ledger
   record.
3. Index CI verifies the bundle: the certificate's repository claim must
   equal the entry's `source`, the ref must be the release tag, and the
   attested digest must equal `zip-sha256`. A PR from anyone whose CI
   identity doesn't match the registered source repo fails mechanically —
   name ownership (RFC §8) becomes cryptographically enforced per release.
4. No secrets exist anywhere in the flow: nothing to steal from authors,
   nothing for camp to rotate.

Trade-off accepted: this ties trusted publishing to forges with OIDC
issuers (GitHub today, GitLab has an equivalent). Authors elsewhere keep
the PR flow with human review of provenance.

## D12: Client plugin trusts TLS + published hash until the TUF client lands

tool_camp v0 verifies artifacts against the SHA-256 in fetched
packages.json — integrity against mirror/CDN tampering, but the metadata
itself is only TLS-protected. The Phase 2 client adds TUF metadata
verification (root pinned at plugin install time, rotation via the TUF
root-update flow) so a compromised origin server can't serve altered
metadata either. The repository side (`camp tuf sign`) already produces
what that client will consume.

## D13: Every scan evaluation is recorded, including rejections

`index/discovery/scan-ledger.yml` records each repository the discovery
scan has ever evaluated: outcome, human-readable reason (e.g. "license:
MIT", "no parseable version.php at root of branch main"), first-seen and
last-checked dates. Committed in the index repo, so rejection decisions
are as auditable as listings. Uses: re-scans skip recently-checked repos
(default 30-day recheck window, `--recheck-days`) instead of re-fetching
them; "why isn't X listed?" has a recorded answer; and the bad-license
bucket is an outreach list — those authors are one LICENSE file away from
eligibility. The ledger never affects installed artifacts; it only gates
what discovery re-evaluates. `camp scan-report` summarizes it.

## D14: Advisories are index objects; revocation is metadata-side, not history-side

Advisories live at `index/advisories/<component>/<CAMP-YYYY-NNNN>.yml`
(schema/advisory.schema.json): severity, Composer-syntax affected-version
constraint, optional fix version, optional `revoke: true`, and the
coordinated-disclosure timeline. Effects are mechanical and tested:
`camp composer` omits revoked versions from packages.json and publishes all
advisories in Packagist-compatible shape (security-advisories.json) so
`composer audit` warns; `camp site` replaces the "no published advisories"
card. The release ledger is never edited — revocation removes a version
from installation *channels* while the archive keeps the artifact for
forensics, exactly as RFC §5.3 specifies. The client plugin consuming
packages.json inherits revocations automatically; per-site warnings for
already-installed versions remain on the D12 roadmap. No real advisories
are committed pre-launch: test fixtures only, because a plausible-looking
advisory against a real plugin would be misinformation.

## D15: GPL-compatible permissive licenses are listed, with the license surfaced

Decision (David, 2026-07-11): repos under GPL-compatible permissive
licenses (MIT, Apache-2.0, BSD family, ISC, Unlicense, CC0, Zlib, WTFPL)
are accepted at Tier 0 rather than rejected, with the detected SPDX id
recorded in the entry and shown as a badge ("Apache-2.0 · GPL-compatible");
GPL-family remains the unmarked norm. Still rejected: no detectable
license, NOASSERTION (unclassifiable — a future content-based re-check
could recover GPL texts GitHub can't fingerprint), and CC-BY-SA (one-way
compatibility only). Empirical note: of ~150 MIT/Apache repos in the sweep,
only ~10 were actual Moodle plugins (e.g. matrix-org/moodle-mod_matrix);
the rest were downloaders, scrapers, and tooling correctly rejected for
having no version.php. Scanner hardening from the same session: transient
fetch errors (rate limits, 5xx) are never recorded as rejections, and
version.php fetches use the authenticated contents API when a token is
present — an anonymous raw-host burst previously mislabeled ~150 repos.

## D16: GitLab discovery, and licensing from the version.php header

`camp scan-gitlab` mirrors the GitHub scanner against gitlab.com's project
API (unauthenticated works; GITLAB_TOKEN for headroom), searching the
frankenstyle prefixes and applying the same acceptance gates, name-match
guard, and ledger. Two GitLab-specific findings:

- GitLab's license detector returns null for almost all Moodle plugins,
  because the convention is to carry the GPL grant as a per-file header
  comment, not a LICENSE file. So license classification reads the Moodle
  header in the version.php we already fetch (the same text classifier used
  for GitHub NOASSERTION recovery). This is legitimate — the in-source GPL
  header is the authoritative license statement for Moodle code.
- GitLab's nested groups mean namespace.path is the immediate subgroup
  (e.g. "moodle"); the maintainer handle uses namespace.full_path
  ("academic-moodle-cooperation/moodle") instead. The schema's gitlab
  maintainer field permits "/".

First sweep: 110 plugins added (index 389 → 499), heavy on the Academic
Moodle Cooperation and dne-elearning/magistere groups that mirror to
GitLab. Ledger keys are namespaced "gitlab.com/<path>" to avoid collision
with GitHub "owner/repo" keys.

## D17: License is read from the version.php header on both platforms

Both scanners now fetch version.php *before* the license gate and classify
the license from its Moodle GPL header when the platform's own detector
finds none. Root cause: the majority of Moodle plugins ship no LICENSE file
— the GPL grant is a per-file header comment — so GitHub reports
`license: null` and GitLab reports none. The old license-first gate wrongly
rejected all of them as "none detected" (danmarsden/moodle-local_recompletion
was the reported example: no LICENSE file, default branch MOODLE_405_STABLE,
GPL-3.0 clearly in the header). Re-running GitHub discovery with the fix
recovered 256 real plugins (index 499 → 755) and dropped bad-license from
502 to 56 — the remainder being genuine non-GPL or the no-LICENSE-no-header
long tail. The "none detected" repos that were actually non-plugins
(downloaders, scrapers) now correctly land in no-version-php instead.

## D18: Discovery recall — frankenstyle-prefix queries sorted by recency

adamjenkins/moodle-mod_aiescape (0 stars, no topics, pushed the same day)
was never even seen by the scanner: the GitHub queries were topic-based
plus a broad "moodle in:name" that matches 27k+ repos, and GitHub hard-caps
search at 1000 results — sorted by stars, a 0-star plugin is unreachable.
Fix: the default query set now includes one targeted "moodle-<type>_ in:name"
search per frankenstyle prefix (mod, local, block, …), each sorted by
`updated` rather than `stars`. Prefix corpora are far smaller (most <1000,
fully paginable) and recency-sorting surfaces exactly the new/low-star tail
that star-sorting buries. GitLab discovery already used prefix queries, so
it was unaffected. See D19 for the >1000 prefixes.

## D19: Date-window sharding for prefixes past the 1000-result cap

The prefixes over the cap turned out to have very different tails, measured
by the pushed-date at rank ~1000 (sorted by recent activity): mod_ cut off
at 2018-09 and block_ at 2020-06 — their unreachable tail is genuinely
abandoned (only ~565 of each pushed since 2024, well within reach). local_
was the real gap: the catch-all type has 1,187 repos pushed since 2024
alone — more than the entire reachable window — so plugins active within
the last ~18 months were being missed.

Fix: when a name query's total_count exceeds 1000, `_date_windows` bisects
its pushed-date range (2010→today) until each window matches < 900, and the
scanner paginates every window. Adaptive, so it self-adjusts as any prefix
grows past the cap rather than hardcoding which types need it. Overlap at
window boundaries is deduped by the seen-repos set. Cost is a handful of
count requests per oversized prefix; the search API's 30/min limit is
handled by the existing backoff.

Correction to the D18 "abandoned tail" note: that analysis measured the
cutoff at the 1000-result boundary, but the comprehensive sweep actually ran
with `--limit 300`, so mod_/block_ were capped at rank ~300 — far shallower
than 1000. block_integrityadvocate (a real, maintained proctoring block)
sat at rank 331, just past that. The reported gap was the --limit, not the
API cap. Applying date-sharding to block_ and mod_ (previously only local_
had it) recovered the full reachable set: block_ 385→1054, plus mod_.

Robustness fix from the same run: a socket-read TimeoutError during a sweep
was uncaught (the network helpers caught URLError/HTTPError but not a bare
TimeoutError/OSError from resp.read()), crashing the whole scan mid-mod_.
_request, _fetch_component, and _gitlab_request now retry network blips a
few times with backoff and degrade to a transient/status-0 result instead
of raising. Regression-tested.

## D20: Four trust tiers — "claimed" split out, security audits are badges

The original three-tier ladder (0 discovered / 1 source-verified / 2
reviewed) conflated two events in old Tier 1: the author claiming the
listing and the first release passing verification. A claimed listing with
no verified release yet had nowhere to sit, and the claim funnel that
drives outreach (RFC §4.4 Tier 0) was invisible in the data. The ladder is
now 0 discovered / 1 claimed / 2 source-verified / 3 human-reviewed. Each
tier answers one question: does it exist, is someone accountable, does the
artifact match its source, have humans read the code.

Consequences: the installation floor moves from tier ≥ 1 to tier ≥ 2
(claiming alone must not make a plugin installable) — enforced in
composer.py's package gate and defensively in the client's
`get_installable()` via `max(2, mintier)`, which also transparently
migrates stored mintier=1 settings from the old numbering (old 1 and new 2
both mean source-verified). Schema: releases are forbidden below tier 2 (a
release enters the ledger only by passing verification, which is what makes
the plugin tier 2); labels + security-contact stay required from tier 1 up
— that boundary was always really the claim boundary (D8).

Rejected: a "security validated" tier. A volunteer review board cannot
honestly certify security, and audits are per-version while tiers are
plugin-level. Version-specific or third-party attestations (e.g. a
professional security audit of one release) will be per-release badges,
not tiers.

## D21: Client multi-repository support — merge in PHP, bind at install

RFC §6.3 (the RPM model) is implemented client-side only: the registry
needs no changes, because "a repository" is just the static file format,
and the merge rules are site policy. tool_camp's repos setting is an
ordered textarea (`name|url|token=…|mintier=N`) rather than a repeated
form: Stage 1 is an alpha admin tool, and a parseable one-line-per-repo
format is easier to audit, copy between sites, and migrate than a custom
UI. The old single repourl is migrated to `default|<url>` in upgrade.php.

Selection is component-keyed, not package-name-keyed: different
repositories may wrap the same component under different Composer vendor
names, so collisions are detected via extra.camp.component. Source
bindings (component → repo, recorded at install) live in plugin config as
JSON; a bound component whose repository disappears from the list is
offered from nowhere — silence, not fallback, because falling back to
another repository is exactly the dependency-confusion move the binding
exists to prevent. Advisory feeds union across repositories with per-repo
failure isolation (one storefront being down must not suppress another's
warnings). Verified with a stubbed-Moodle PHP harness covering parsing,
priority, shadowing, binding, per-repo mintier and bearer tokens.

## D22: Artifacts are a derived cache today; the archive needs real storage

Where the ZIPs live, as of the Pages-era pipeline: nowhere permanent.
`camp artifacts` rebuilds every ledger release from its tagged source on
each publish, hash-gates against the ledger, and ships the tree in the
Pages deployment, which the next deploy replaces. This is coherent
because builds are deterministic (D2): source tag + ledger hash fully
determine the bytes, so artifacts are cache, not canon. Two consequences
accepted for now: CI rebuilds everything every publish (the materializer's
keep-if-hash-matches optimization needs a persistent output dir), and
there is no hosted copy of any artifact beyond the current deployment.

The soft spot this leaves: the permanent-archive promise (RFC §2.2) is
not yet backed by permanent storage. "Rebuildable from upstream" fails
exactly when archival matters — deleted repos, suspended accounts, moved
tags. At that point the ledger proves what the artifact *was* but nothing
can produce it.

Decision: the artifact-era hosting move is two components, not one.

1. **Archive** (the commitment): append-only object storage. Each ZIP is
   written once at verification time — as part of the release merge, not
   the publish — and never deleted. Revoked versions stay (withdrawn from
   distribution, preserved for forensics, RFC §5.3). This must exist by
   (or shortly after) the first Tier 2 release: the first verified ZIP is
   the first thing the registry has promised to keep forever.
2. **Serving origin** (the convenience): rsync-capable host or bucket+CDN
   that the publish step populates from the archive instead of rebuilding,
   and that mirrors sync from (MIRRORING.md). Replaceable at any time;
   mirrors and the archive make it non-critical.

Until the archive exists, the registry's continuity rests on upstream
availability plus reproducibility — acceptable pre-launch with an empty
ledger, and flagged here so the hosting decision is made with the
archival requirement in view rather than as a serving-cost question only.

## D23: The code checker is warn-only — style is quality, not trust

Decided when the registry's first real release PR (mod_cmi5launch 1.1.0)
was blocked by ~58 auto-fixable phpcs findings in code whose artifact had
just been rebuilt and hash-verified perfectly. camp's promise is
"source-verified, not human-reviewed" (RFC §4.4): style violations do not
weaken the artifact-matches-source guarantee, and blocking on them would
keep a large share of legitimate, widely-used community plugins out —
the friction landing on exactly the authors the registry courts. The old
directory's prechecker reported findings; humans decided.

The floor therefore splits on what a failure *means*:

- **Hard-blocking (trust):** deterministic rebuild + hash verification,
  moved-tag detection, append-only ledger, schema validation, malware
  scan, PHP lint (parse errors), moodle-plugin-ci validate + savepoints.
- **Warn-only (quality):** moodle-plugin-ci phpcs, the disclosure-label
  heuristics and security lint (already warn-only per D10).

Same philosophy as D10: surfacing findings trains reviewers; blocking on
them trains authors to game the patterns. Code-quality judgment is what
Tier 3 human review is *for*. Findings stay visible in every release
PR's checks, so the information is public even though it doesn't gate.

## D24: Third-party review display is registry-level, not author-declared

Decided when MDL Shield shipped a published-reviews feed (July 2026).
Author-declared endpoint badges have survivorship bias: an author
declares the badge after a good report and quietly doesn't after a bad
one, so an opt-in badge can never be evidence. The registry therefore
fetches the reviewer's published-reviews feed once per publish and
renders the result itself on every plugin page: a trust-strip shield
when the review covers the release camp currently offers (an
unqualified signal or none), a grade pill per reviewed release in the
versions table, and a Project row carrying the full detail.

The display honours two boundaries that are not camp's to move:

- **The reviewer's privacy model.** Publication is opt-in on their
  side, so absence from the feed is genuinely ambiguous — unreviewed,
  reviewed-but-private, and unknown are indistinguishable by design.
  Absence renders as nothing; the registry never shows "not reviewed".
- **The reviewer's subject.** Reviews cover the moodle.org
  distribution of a version, which is not byte-identical to camp's
  tag-built artifact (and occasionally differs meaningfully). Every
  rendered review says so.

Matching keys on frankenstyle component plus the `$plugin->version`
integer — the identifier Moodle itself trusts; release strings drift.
Consistent with D-series precedent (RFC §4.6), visitors load nothing
from the reviewer's servers: the feed is fetched and sanitized at
publish time, and camp draws its own chips. Author-declared badges for
the same reviewer are suppressed as redundant; the declared-badge
mechanism remains for future allowlisted hosts.

- Where advisories live in the index tree and their Composer projection
  (`composer audit` format) — RFC §5.3/§6.1.
- Listing ingestion pipeline (sanitized markdown, image re-encoding) and
  where rendered listing content is published — RFC §4.1.
- Advisory signing: advisories are validated objects but not yet
  individually signed; they gain signatures with the TUF rollout (they are
  covered by `camp tuf sign` today as part of the published tree).
- The BRANCHES table in moodleversions.py: confirm 5.1/5.2 first-release
  version codes against upstream before they become load-bearing.
