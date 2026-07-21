# RFC: A Community-Governed Plugin Repository for Moodle

**Status:** Draft for community comment (revision 10)
**Author(s):** the camp project — https://github.com/camp-registry
**Date:** July 2026
**Feedback:** https://github.com/camp-registry/camp-docs/discussions

---

## 1. Summary

We propose building an open, community-governed repository for Moodle plugins: a place to publish, discover, verify, and install plugins that is not controlled by any single company, cannot be paywalled, and cannot disappear if any one server or organization does.

The design in brief: plugin metadata lives in a public Git repository that anyone can clone or mirror; every published plugin ZIP is automatically verified to match its public source code; releases are cryptographically signed and recorded in a public, tamper-evident log; and distribution is a static, mirrorable file tree that any university or hosting provider can replicate with a single sync job. A small Moodle admin plugin lets site administrators browse and install plugins from the repository using Moodle's existing installation machinery, with no core changes required to start.

The repository also provides something the current directory never has: a coordinated security disclosure process for third-party plugins, with signed security advisories that automatically warn every administrator running an affected version.

The design deliberately builds on Moodle's own new direction: Moodle 5.2 introduces native Composer-based plugin installation for Composer-managed sites. This repository doubles as a Composer repository, extending that official channel with source verification, signing, review tiers, and security advisories that surface in `composer audit`. These are capabilities Packagist alone does not provide.

This RFC asks the community: is this worth building together, and will you help?

## 2. Background and motivation

### 2.1 What is changing

Moodle HQ has announced the Moodle Marketplace, launching mid-2026 as the replacement for the community Plugins Directory.

### 2.2 Why this is structural, not incidental

The Marketplace transition highlights problems the community has lived with for years, which no policy adjustment can fix:

- **Single point of control.** One organization decides what is listed, on what terms, and can change those terms at any time, as the transition itself demonstrates.
- **Single point of failure.** If the official directory is down, degraded, or discontinued, plugin discovery and updates break for every Moodle site in the world.
- **Review bottleneck.** Manual approval has historically depended on one or two individuals, with hundreds of submitted plugins waiting in the queue. The cause is not lack of goodwill: centralized human review does not scale.
- **No security advisory channel.** There is no strong mechanism today for privately reporting plugin vulnerabilities, coordinating fixes, or warning administrators running affected versions.

The Moodle community of institutions, developers, and service providers built the plugin ecosystem. This proposal is about giving that ecosystem infrastructure the community actually controls. Beyond its direct utility, the repository also serves as the ecosystem's permanent archive: every version of every listed plugin is preserved, so nothing is lost in the Marketplace transition or any future one.

## 3. Goals

1. **Resilience.** No single server, organization, or person whose failure or capture breaks the system. Full mirrors must be trivial to operate.
2. **Trust and security.** Stronger, verifiable guarantees than the current directory offers: every artifact provably built from public source, every release signed, every signing event publicly auditable, and a working security disclosure and advisory pipeline.
3. **Discoverability.** A good browsing and search experience for admins, and eventually installation from within Moodle itself.
4. **Low operating cost.** The steady-state system should be runnable by a small volunteer team on donated or cheap infrastructure.

### Non-goals

- Replacing moodle.org community forums, documentation, or the Moodle core security process (core vulnerabilities are always routed to Moodle HQ's existing process; this project handles **third-party plugins only**).
- Hosting paid/commercial plugin sales. The project is commerce-neutral: it neither sells plugins nor excludes plugins whose authors sell services or premium versions elsewhere, and it imposes no exclusivity: authors are free to list in this repository, the official Marketplace, Packagist, or all three. The focus is being the best possible home for free and open plugins. The architecture does, however, deliberately permit commercial storefronts (including the official Marketplace) to operate **on top of** the registry layer (using its namespace, verification, signing, and advisories, with payments and entitlements as their own layer), following the neutral-registry model of Packagist and Private Packagist; §6.3 specifies how a site combines several such repositories side by side. Registry governance and neutrality are not for sale in that arrangement; distribution rails are.
- Forking Moodle itself.

## 4. Design overview

The architecture combines patterns proven by the AUR/Homebrew (Git-native index), F-Droid (verified builds, signed static repository), and modern package registries (npm, PyPI) for signing and transparency.

None of this is exotic: it is the direction the wider package-registry world has publicly converged on. In 2026, Packagist.org, PHP's own registry, responded to a wave of supply chain attacks by shipping a public transparency log, version immutability, and malware-feed integration, and stated that its long-term direction is hosting immutable build artifacts with build provenance and Sigstore attestations verified client-side, benchmarked against the OpenSSF Principles for Package Repository Security and SLSA's dependency track. PyPI and npm made the same moves earlier (mandatory MFA, signed provenance, staged publishing). This project builds that foundation for the Moodle ecosystem from day one rather than retrofitting it after an incident.

### 4.1 The index: a Git repository as the source of truth

Each plugin is described by one small metadata file (component name, maintainer, source repository URL, security contact, disclosure labels (§4.7), and a list of released versions with supported Moodle versions and content hashes). All metadata files live in a single public Git repository, "the index."

- Publishing a plugin or a new version = opening a pull request against the index.
- The entire registry history is auditable, diffable, and clonable by anyone in seconds.
- Mirroring the registry's brain is `git clone`.

**Listing content lives in the plugin's own repository.** Following the pattern proven by F-Droid (Fastlane metadata) and Packagist (composer.json), descriptive content (description, category, screenshots, documentation links, and declared disclosure labels) is read from a small manifest directory in the plugin repo itself (e.g. `.camp/listing.yml` plus a `screenshots/` folder). Authors update their listing with an ordinary commit; the index entry shrinks to essentially "this component name belongs to this repository," and the release tooling scaffolds a starter manifest for repos that lack one.

This split is also a security boundary:

- **Descriptive content** (from the repo) is treated as untrusted input: descriptions are sanitized markdown with no raw HTML; images are restricted to raster formats (no SVG) and re-encoded by CI (decoded, resized, stripped of metadata, and re-emitted), neutralizing malformed or polyglot files.
- **Authoritative content** (component-name ownership, tier status, verification results, and security advisories) lives only in the index and is computed or arbitrated by the registry, never read from the repo at face value. Disclosure labels sit in between: declared in the manifest, validated by CI heuristics and Tier 3 review (§4.7).
- **Listing content is pinned at release.** CI ingests the manifest and screenshots at the same tagged commit it builds the ZIP from, and records their hashes in the signed release metadata. A later compromise of the source repository therefore cannot silently alter the description or screenshots shown for an already-published version; the listing itself is covered by the verification guarantees of §4.2 and §4.3.

### 4.2 Automated verification: the trust floor

On every submission, CI automatically:

1. Clones the plugin's public source repository at the declared tag.
2. Builds the distribution ZIP deterministically from that tag.
3. Confirms the ZIP hash matches the submitted metadata.
4. Runs the standard Moodle plugin checks (code checker, PHPLint, version.php sanity, privacy API presence, basic security lint).
5. Runs malware and behavioral scanning on the artifact (following Packagist.org's integration of malware detection feeds into package metadata); for plugins dual-published to Packagist, its malware flags are consumed as an additional signal.

Only submissions passing all checks are merged. This guarantees, for **every** plugin in the repository, including brand-new, unreviewed ones, that *the artifact you install is exactly what is in the public source repository*. Because Moodle plugins are interpreted PHP with no compilation step, this guarantee is cheap for us and for authors, and it is stronger than what the current directory provides.

**Releases are immutable.** The release ledger is append-only: a published version is never rewritten. If an upstream tag is moved or force-pushed after publication, the recorded commit and hash no longer match, the mismatch is rejected, and the maintainer is alerted. Mistakes are fixed by releasing a new version, never by overwriting an old one. This is the same property Packagist.org enforced in 2026 after re-tagging featured in nearly every recent supply chain attack on the PHP ecosystem.

### 4.3 Signing and transparency

- All repository metadata is cryptographically signed following **TUF** (The Update Framework), a mature, independently audited standard (used by PyPI's design work, Docker, and automotive updates) built on the assumption that servers and even some keys *will* eventually be compromised, and designed to survive that. In practice this means: separate keys for separate duties, important actions requiring multiple signatures, and root keys kept offline by several different people in different places.
- Every release/signing event is additionally published to **Rekor**, sigstore's public append-only transparency log (the same infrastructure npm and PyPI use). Anyone can audit the full history; a compromised key or infrastructure cannot rewrite the past without detection, and maintainers can subscribe to alerts if their identity is ever used unexpectedly.
- Author publishing uses **trusted publishing** (OIDC): releases flow from the author's CI (e.g. a GitHub Action on tagging) without long-lived passwords or API keys that can be stolen.

### 4.4 Trust tiers

- **Tier 0: Discovered (metadata only).** The ecosystem is seeded by scanning public sources (GitHub/GitLab naming conventions and topics, and the existing directory's published source URLs) for GPL-licensed Moodle plugins. A discovered listing is a search result, not a distribution: name, description, license, and a link to the author's canonical repository. **No artifacts are hosted at this tier**, and any author may have their listing removed on request, no questions asked. Discovered listings double as outreach: each carries a "claim this plugin" path for its author.
- **Tier 1: Claimed.** The author (or a community submitter, with the author notified) has claimed the listing: ownership of the source repository is proven, and the listing gains what only an author can provide — a security contact, declared disclosure labels (§4.7), and consent to host artifacts. A claimed listing is accountable but not yet installable: no release has passed verification. This tier exists so the registry can distinguish "nobody has engaged" from "the author is on board, first verified release pending", and so the claim funnel that drives outreach is visible in the data.
- **Tier 2: Verified (automatic).** Releases flow through the automated checks in §4.2. Available for installation immediately. Clearly labelled: "source-verified, not human-reviewed."
- **Tier 3: Reviewed (human).** A plugin version is promoted after independent code review by at least **two** members of a community review board, each signing the promotion with their own key. No single reviewer, however trusted, can promote a plugin alone. Reviews and reviewer identities are public.

Each tier answers one question: does it exist (0), is someone accountable for it (1), does the artifact provably match its public source (2), have humans read the code (3). Signals that are version-specific or come from outside the registry's own process — for example a professional **security audit** of a particular release — are deliberately *not* tiers: they are displayed as per-release badges, recording who attested what about which version, without implying the whole plugin carries that guarantee forever.

**Administrators choose the tier they trust.** Installation is only possible from Tier 2 upward — that is the registry's floor, since below it there is no verified artifact to install. Above the floor, the policy is the site's: both the client plugin and the Composer metadata expose the tier, and an administrator sets a minimum installation tier ("Tier 2 or above" vs. "Tier 3 only") the same way they set the cooldown policy below. Institutions that today require "approved" plugins get an equivalent (and more transparent) signal by requiring Tier 3.

We are explicit about what Tier 2 does and does not promise: it proves the artifact matches its public source; it does not prove the source is benign. Two mitigations apply. First, sites can opt into a **minimum release age ("cooldown") policy**, such as "only install releases older than 48 hours", honored by the client plugin and expressed in the Composer metadata; malicious releases are typically detected and revoked within hours of publication, so a cooldown lets cautious sites sit out the risky window entirely. Second, before launch the project will have a published incident-response plan and a signed revocation mechanism (§5.3) so that a malicious or vulnerable plugin can be pulled from installation quickly, with a public post-mortem.

### 4.5 Distribution: static and mirrorable

The published repository is a plain file tree: plugin ZIPs, metadata, signatures, advisories, and a generated website for browsing and search. The same generator also emits Composer repository metadata (see §6.1), so the repository is natively consumable by both Moodle's admin tooling and Composer-managed sites. There is no database or application server in the download path.

- A full mirror is one `rsync` (or `git clone` + object fetch) on a cron job.
- Clients verify signatures, so a mirror does not need to be trusted: a malicious or compromised mirror can at worst serve stale or missing files, never tampered ones.
- We will seek 3–5 institutional mirror operators (universities, Moodle partners) in different countries at launch.

### 4.6 Privacy by design

An update check inherently reveals which plugins a site runs; collectively, that is a map of the security posture of thousands of institutions. The project commits to data minimization: no per-site accounts or registration, no retention of installed-plugin lists, aggregate statistics only, and a published privacy policy stating exactly what is logged and for how long. Mirrors see only anonymous file downloads.

### 4.7 Disclosure labels

Every plugin must declare a set of standardized disclosure labels, treated exactly like the supported-Moodle-versions field: required, machine-readable, filterable, and displayed as badges on the website and in the client plugin. This follows the model proven by F-Droid's "Anti-Features": labels inform rather than disqualify. The initial label set:

- **`fully-free`**: complete functionality with no paid components or external accounts.
- **`freemium`**: free to install, but some features are paywalled or require a paid unlock.
- **`paid-service`**: requires a subscription, API key, or account with an external commercial service to function (in full or in part).
- **`external-account`**: requires a third-party account even if free (relevant for privacy and procurement review).
- **`donation-supported`** and **`commercial-support-available`**: optional, author-promotional.

Administrators can filter searches by label ("show only fully-free plugins"), and institutions can encode label policies in procurement rules. Labels are validated for presence by CI at submission; automated heuristics (e.g., detecting API-key or license-key settings in the code) flag likely misdeclarations for human attention, and Tier 3 review verifies labels as part of promotion. Deliberate mislabeling is treated as a trust violation: correction, and delisting for repeat offenses.

This is disclosure, not gatekeeping: freemium and service-backed plugins are welcome; the requirement is simply that administrators know what they are installing before they install it.

## 5. Security reporting and advisories (third-party plugins only)

This section describes a capability the official directory has never offered: an end-to-end pipeline from private vulnerability report to automatic administrator warning. Reports concerning Moodle core are always redirected to Moodle HQ's existing security process; this project's scope is strictly third-party plugins.

### 5.1 Reporting channels

- **Per-plugin security contact.** Every plugin's metadata must include a `security-contact`: an email address or, preferably, a link to the source platform's private vulnerability reporting (GitHub and GitLab both provide this natively). CI validates the field exists.
- **Hosted fallback intake.** For unresponsive maintainers or reporters who prefer a single door: a well-known address and simple form operated by the project. Reports are encrypted to the security triage team's published keys (distributed in the signed repository metadata, so reporters can verify they are encrypting to the genuine team) and visible only to a triage team of 2–3 named members, never a single individual.

### 5.2 Coordinated disclosure policy

Standard coordinated disclosure: the maintainer is notified privately; an embargo is agreed (default 90 days, negotiable); the fix is released; then the advisory is published. If a maintainer is unresponsive or declines to fix, the security team may publish the advisory after the embargo expires and mark affected versions in the repository regardless.

### 5.3 Signed advisories and revocation

Advisories are signed objects in the repository, exactly like plugin metadata. This closes the loop:

- The Moodle client plugin warns every administrator running an affected version ("plugin X version Y has a published vulnerability; upgrade to Z"). No tracking is required for this: consistent with §4.6, the registry never knows which plugins a site runs. The client periodically downloads the complete signed advisory feed — a small, anonymous static-file fetch that works against any mirror — and compares it locally against the site's own installed plugins. Matching happens on the site, where that knowledge already lives, never on the server. Composer-managed sites get the same result from `composer audit`, which performs the identical local comparison.
- For severe cases, a signed revocation removes the affected version from installation entirely while preserving it in the archive for forensic purposes.
- After disclosure, the full timeline becomes public.

### 5.4 Triage team governance

Because the triage team holds sensitive information pre-disclosure: multiple named members with fixed terms, a public log of advisory counts and time-to-resolution (process transparency without leaking embargoed details), and a strict published rule against sharing or acting on embargoed information. Recruitment will target university security teams, who have professional incentive and existing expertise.

## 6. Moodle integration

Integration is two-pronged, reflecting the split in how Moodle sites are now managed.

### 6.1 The Composer channel (Moodle 5.2+, Composer-managed sites)

Moodle 5.2 introduces native Composer-based plugin installation (MDL-87473): plugins declare a `composer.json` with a `moodle-[plugintype]` package type, and the official `moodle/composer-installer` places them in the correct directory. This is a sanctioned installation path that does not run through the Marketplace.

Because a Composer repository is fundamentally a static metadata file tree, this repository publishes Composer metadata alongside its own format at negligible extra cost. Site administrators on Composer-managed sites can then install verified plugins directly:

```
composer config repositories.community composer https://<repo-domain>
composer require <vendor>/moodle-mod_example
```

What this adds over plain Packagist: download URLs point at this repository's source-verified, signed artifacts; Moodle-version compatibility is expressed in Moodle's own versioning scheme (which maps poorly to semver); review-tier status is visible in the metadata; and the project's security advisories (§5) are served in Composer's advisory format, so affected versions surface automatically in `composer audit`. With Composer 2.10's dependency-policy framework, a unified mechanism covering vulnerability advisories, malware-flagged versions, and abandoned packages, this repository's advisories and revocations can do more than warn: under an administrator's policy they **block installation**, and site policies can also enforce the cooldown period described in §4.4.

The project remains a good citizen of the wider ecosystem: authors are encouraged to publish to Packagist as well, and the release tooling (§12) generates a correct `composer.json` for plugins that lack one.

### 6.2 The client plugin (everyone else)

Most Moodle sites are not Composer-managed and will not be for years. Moodle's built-in update checker and one-click installer call a hardcoded moodle.org endpoint, so for this majority:

- **Stage 1 (launch, no core changes):** a small admin-tool plugin, installed once via standard ZIP upload, that lets admins browse the community repository, verifies signatures, installs/updates plugins through Moodle's existing deployment machinery, and surfaces security advisories.
- **Stage 2 (optional, deprioritized):** an API-compatible endpoint implementing Moodle's documented update-check JSON format, for consortiums willing to apply a host override; less important now that Composer provides a sanctioned alternative path.
- **Stage 3 (proposal to core):** submit a patch making the update/installation provider URL an admin setting. We commit to proposing this publicly through the standard core contribution process; its acceptance or rejection will itself be informative.

### 6.3 Multiple repositories and storefronts

The non-goals in §3 state that commercial storefronts (including the official Marketplace) may operate on top of the registry layer. This section makes the mechanics concrete. The model is the one Linux administrators already know from RPM/APT: a site enables a small, explicitly chosen set of repositories — say, this community repository, an official Marketplace repository, and a Moodle partner's customer-only repository — and the client merges them under rules that keep trust separable.

- **Anyone can be a repository.** The published format is a static file tree (`packages.json`, artifacts, `security-advisories.json`); serving it requires no relationship with this project, only the format. A partner distributing paid GPL plugins to its customers is simply a repository behind an access token, exactly like a vendor RPM repository.
- **Trust is per-repository and never bleeds across.** Each configured repository carries its own credentials and, as TUF lands (§4.3), its own pinned trust root — the analog of importing a vendor's signing key. A tier is an assertion *by its repository*: "Tier 3" from this registry means two community reviewers signed the promotion; from another repository it means whatever that operator's process is. Clients display tiers attributed to their source, and minimum-tier policy is set per repository — a site may demand Tier 3 from the community registry while accepting a contracted partner's repository wholesale, because the contract is the trust basis there.
- **The administrator arbitrates across repositories.** Within one repository, names are arbitrated by its registry (§8). Across repositories, collisions are resolved by the administrator's priority ordering, with shadowing always visible ("also available from…") and per-component pins for overrides. The choice is never made silently.
- **The source-binding rule (normative).** Once a component is installed from repository X, updates for it are offered only from repository X until the administrator explicitly re-pins it. A higher version number in another repository is not an update; it is at most a visible shadow warning. This forecloses cross-repository dependency confusion — the attack where a public repository publishes an inflated version of a private component to hijack its updates — and follows the rule Composer 2 and APT pinning converged on after real-world incidents.
- **Advisories union; revocation is scoped.** The client fetches every enabled repository's advisory feed and matches all of them against all installed plugins, whatever their origin: warnings are cheap and should not depend on which door a plugin came through. Revocation, by contrast, is scoped to the publishing repository — no repository can pull another's package, only warn about it.
- **The boundary of the model.** This design is for a handful of administrator-chosen repositories. It is *not* open federation: with hundreds of auto-discovered sources there is no arbiter, and namespace integrity — the registry's core security function — collapses. The client will not discover repositories on its own.
- **Listings can move — in either direction.** With no exclusivity (§3), an author who takes a plugin to another repository simply stops publishing here, and may mark the listing **moved** with a successor pointer. A moved listing keeps everything the community already relies on: published versions stay in the archive, stay installable, and stay covered by advisories; the component name remains the author's (§8) if they return. Clients surface the pointer ("new versions publish at …") so sites go quiet with an explanation, never silently — but *following* the plugin stays an explicit administrator action (add the repository, re-pin the component), because automatic follow-the-plugin would reopen exactly the takeover hole that source binding closes. In Composer metadata the pointer is emitted as the standard abandoned-with-replacement field, so `composer` warns natively. Dual-listing needs none of this ceremony: both repositories simply list the component, and the site's priority order decides.

Composer-managed sites (§6.1) get these semantics natively — multiple repository entries with scoped credentials and canonical-repository behavior are built into Composer. The client plugin's job is to deliver the same model to the non-Composer majority.

## 7. Translations

Plugins listed only in this repository lose access to AMOS, Moodle's official translation infrastructure. We treat this as an opportunity rather than a loss: the release pipeline will generate draft language packs via machine translation automatically on every release, with human corrections submitted as ordinary pull requests. This provides day-one coverage in far more languages than volunteer throughput allows, with community review where it matters.

## 8. Naming, trademark, and namespace policy

- **Naming.** "Moodle" is a registered trademark of Moodle Pty Ltd. The project will adopt a distinct name, using "for Moodle" only descriptively (as in "a plugin repository for Moodle"), and will obtain legal review of the name and domain before public launch, through the fiscal host where possible. Name proposals are invited in the discussion thread.
- **Component namespace.** Moodle plugins are identified by component names ("frankenstyle", e.g. `mod_attendance`), and the official directory has historically acted as the name registry. To protect users from collisions and typosquatting, this repository's policy is: (1) any component name registered in the official directory maps only to that plugin's canonical source; (2) new names are first-come but subject to an automated similarity check and human sign-off; (3) disputes are resolved publicly by the governance body.

## 9. Governance

Structure follows adoption (we will not design a constitution for a community that doesn't exist yet), but the principles are fixed from day one:

1. **Founding stewardship (launch → ~6 months).** A named group of 3–5 founding stewards hold the offline root keys (geographically distributed, threshold-based so no individual or minority can act alone) and operate the infrastructure. All decisions and signing events are public.
2. **Elected review board (as Tier 3 launches).** Reviewers are recruited publicly; promotion requires two independent reviewer signatures; board membership rotates on fixed terms.
3. **Fiscal host and formal home (within year one).** We will seek a nonprofit fiscal host (e.g. Software Freedom Conservancy, Open Collective, or a European equivalent given the Moodle community's geography) so that donations, domains, and trademarks belong to the project, not to any founder, and so that distribution happens under an organizational legal umbrella with standard open-source warranty disclaimers.
4. **The exit guarantee.** Because the index is Git, the artifacts are static files, all tooling is free software (GPL/Apache), and signing follows open standards, the community can fork the *entire repository*, content, history, and infrastructure included, if governance ever fails. This property is deliberate. We ask to be trusted, but we build so that trust is not required.

## 10. Risks and mitigations

We prefer to name the hard problems ourselves:

| Risk | Mitigation |
|---|---|
| Trademark challenge over naming | Distinct name, descriptive use only, legal review pre-launch (§8) |
| Moodle HQ hostile platform changes (hardcoded endpoints, core-level restrictions) | Two independent channels: the Composer path is HQ's own sanctioned mechanism and pulls from any configured repository by design; the Stage 1 client plugin uses only standard, supported plugin APIs, and blocking it would require blocking all manual plugin installation. Institutional adoption raises the political cost of hostile changes. |
| Component-name collisions / typosquatting | Namespace policy in §8; automated similarity checks |
| Author backlash over unsolicited Tier 0 indexing | Metadata-only listings (no artifact hosting), prominent labeling, instant no-questions removal on request, and "claim this plugin" as the primary outreach channel |
| First malicious plugin through Tier 2 | Honest labelling, malware scanning in CI (§4.2), opt-in cooldown policy (§4.4), pre-launch incident-response plan, signed revocations, public post-mortems (§5.3) |
| Update-check privacy exposure | Data minimization by design, published privacy policy (§4.6) |
| Economic gravity pulls authors to the paid Marketplace | Be the better home for free plugins: instant CI publishing vs. review queues, security pipeline, archival permanence; remain commerce-neutral |
| Reviewer/triage volunteer bandwidth (the failure mode of most web-of-trust systems) | Automation as the trust floor; human review as a scarce, targeted resource; recruit institutionally, not just individually |
| Trust-infrastructure key mismanagement | TUF threshold design, offline roots, documented ceremonies, and an external audit of the signing path before launch |

## 11. Roadmap

| Phase | Timeframe | Deliverables |
|---|---|---|
| 0. RFC & consensus | now | This document, public discussion, name selection + legal review, recruit founding stewards & mirror hosts |
| 1. Foundation | 2–4 weeks | Metadata schema, index repo, CI verification pipeline, ecosystem seeding scan |
| 2. Distribution | ~1–2 months | Signed static repo, Composer repository metadata, browse/search website, transparency logging, first mirrors |
| 3. Launch | ~2–4 months | Catalog seeded with Tier 0 discovered listings + claimed (Tier 1) and source-verified (Tier 2) plugins, Moodle client plugin (Stage 1), public trust + privacy + disclosure policies, incident-response plan |
| 4. Maturity | 4+ months | Review board & Tier 3, security triage team, fiscal host, external signing-path audit, core patch proposal (Stage 3) |

## 12. What we are asking of the community

- **Plugin authors:** would you list your plugin? What would make publishing painless? (Target: a single GitHub/GitLab Action that releases on `git tag`, generating a valid `composer.json` and a starter listing manifest automatically for plugins that lack them.)
- **Institutions:** would you mirror? Would your policies accept Tier 2/Tier 3 labels? Would you contribute reviewer or security-triage time?
- **Developers:** help build the CI pipeline, the website, and the Moodle client plugin.
- **Everyone:** critique this design, especially the trust model and the risk table. And propose names.

## 13. Open questions

1. Should Tier 3 review target plugins (all versions after first approval, like the current directory) or each version?
2. How should plugin *adoption* work when a maintainer disappears (the current directory's adoption programme is a genuinely good pattern worth keeping)?
3. Refinements to the disclosure label set (§4.7): are more granular labels needed (e.g., distinguishing ads, telemetry, non-GPL dependencies), and should any label carry installation warnings rather than badges alone?
4. Default embargo length and severity thresholds for the disclosure policy (§5.2).
5. Which machine-translation pipeline and which languages at launch (§7)?

## Appendix A: Glossary

- **TUF (The Update Framework):** an open standard for securing software repositories. Its core ideas: assume compromise will happen; split signing power across multiple keys with different jobs; require multiple signatures for sensitive actions; keep the most powerful keys offline. CNCF-graduated and independently audited.
- **Rekor / sigstore:** free public infrastructure (Linux Foundation) providing an append-only, tamper-evident log of signing events. Used by npm, PyPI, and Kubernetes. Lets anyone verify *when* something was signed and detect after-the-fact tampering or misuse of identities.
- **OIDC / trusted publishing:** releasing packages by proving identity through a short-lived login token from CI (e.g. GitHub Actions) instead of storing long-lived passwords or API keys that can leak.
- **Threshold signatures ("2-of-N"):** an action is only valid if at least 2 (of N) authorized key-holders sign it, so no single person can act unilaterally and no single stolen key is catastrophic.
- **Reproducible/deterministic build:** producing a byte-identical artifact from the same source every time, so anyone can independently confirm an artifact matches its source.
- **Coordinated disclosure / embargo:** the practice of reporting a vulnerability privately to the maintainer first, agreeing a period to develop a fix, and publishing details only after users can protect themselves.
- **Frankenstyle / component name:** Moodle's naming scheme for plugins (`plugintype_pluginname`, e.g. `mod_quiz`), which uniquely identifies a plugin within a Moodle site.
- **Composer / Packagist:** Composer is PHP's standard dependency manager; Packagist is its default public package registry. Moodle 5.1+ ships with Composer-managed dependencies, and Moodle 5.2+ supports installing plugins via Composer. A "Composer repository" is a metadata format any server (including a static file host like this project) can publish so that `composer require` works against it.
- **`composer audit`:** a built-in Composer command that checks installed packages against security advisories published by the configured repositories; this is the delivery mechanism for this project's advisories on Composer-managed sites.

## Appendix B: Sources for §2.1

All claims in §2.1 are drawn from Moodle HQ's own published materials and should be re-verified (and linked) at time of posting, as terms may change:

- Moodle Marketplace onboarding site (marketplace.moodle.com): Plugins Directory shutdown and migration, transaction fee schedule, Stripe Connect requirement, paid-service plugin policy, Provider Terms of Use requirement.
- Moodle.org community announcement, "New Moodle Marketplace: onboarding is open for paid plugins" (February 2026).

The convergent-direction claims in §4 are drawn from: Packagist blog, "An update on Composer & Packagist supply chain security" (May 2026), blog.packagist.com.
