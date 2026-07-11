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

## Open items carried forward

- Where advisories live in the index tree and their Composer projection
  (`composer audit` format) — RFC §5.3/§6.1.
- Listing ingestion pipeline (sanitized markdown, image re-encoding) and
  where rendered listing content is published — RFC §4.1.
- Advisory signing: advisories are validated objects but not yet
  individually signed; they gain signatures with the TUF rollout (they are
  covered by `camp tuf sign` today as part of the published tree).
- The BRANCHES table in moodleversions.py: confirm 5.1/5.2 first-release
  version codes against upstream before they become load-bearing.
