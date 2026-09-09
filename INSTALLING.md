# Installing plugins from camp

For site administrators. Three ways to install a plugin listed on camp, in order of how much they automate: Composer, the tool_camp client plugin, or a ZIP download. Whichever you use, the package is the same source-verified artifact: rebuilt from the maintainer's tagged source, byte-compared, and recorded with its SHA-256 in the public index.

Only source-verified releases (Tier 2 and above) are installable. Discovered and claimed listings without a verified release show where the source lives, and nothing more.

## Composer (Moodle 5.2 and later, Composer-managed sites)

Moodle 5.2 supports installing plugins with Composer (MDL-87473): a plugin is a Composer package of type `moodle-<plugintype>`, and Moodle HQ's [`moodle/composer-installer`](https://github.com/moodle/composer-installer) places it in the right directory. camp publishes a standard Composer repository, so a Composer-managed site installs verified plugins with two commands of one-time setup and one command per plugin.

### One-time setup

From your Moodle root (the directory holding `composer.json`):

```
composer config repositories.camp composer https://camp-registry.org
composer config allow-plugins.moodle/composer-installer true
```

The first line adds camp as a package source. The second lets the installer run; Composer refuses to execute any plugin without an explicit allow entry, and every camp package depends on this one (it is fetched from Packagist, where HQ publishes it).

### Installing a plugin

Every plugin page shows its install command, for example:

```
composer require bfh/moodle-quiz_archive
```

Package names are `<vendor>/moodle-<component>`, where the vendor is the GitHub account of the plugin's first listed maintainer and the component is Moodle's own name for the plugin (`mod_quiz`, `block_html`). The plugin page has the exact name, and a Copy button.

To install a specific release instead of the newest, pick it in the plugin page's release list; the command updates with the version pinned:

```
composer require bfh/moodle-quiz_archive:"5.2-patch3"
```

Versions follow Composer's grammar. Where a maintainer uses the Moodle community's `-rN` scheme (v5.2-r3, "third release for 5.2"), camp publishes it as `5.2-patch3`, which orders the same way. The maintainer's own version string is kept in the package metadata under `extra.camp.version`, and the download and source reference still point at the maintainer's real tag.

Updates work as for any Composer package: `composer update bfh/moodle-quiz_archive`, then run Moodle's upgrade (`php admin/cli/upgrade.php`) as you would after any plugin install.

### Which Moodle versions a release supports

camp records the Moodle branches each release declares in `extra.camp.supported-moodle`, and the plugin page shows them. Composer only enforces them if `moodle/moodle` is itself a package in your project; on a plain checkout it will happily install a release for a newer branch, so check the plugin page's Moodle range before pinning.

### Security advisories

camp's advisories are published in Composer's own format, so Composer treats them exactly as it treats Packagist's. `composer audit` lists any advisory affecting an installed camp package, with camp's identifier, severity and link (the same record as on the [advisories page](https://camp-registry.org/advisories/)), and exits non-zero, which is what you want in a deployment pipeline. Composer 2.10 and later also refuse to install or update to a version with an open advisory unless you say otherwise; the controls are `policy.advisories.block`, `policy.advisories.ignore-id` for an advisory whose risk you have accepted, and `policy.advisories.ignore` for a whole package (older Composer versions use the `audit.*` keys). Plugins whose maintainers have marked them moved appear as abandoned with the successor named, under `policy.abandoned`.

Versions withdrawn by an advisory are removed from the repository outright, so they cannot be installed even when pinned.

### Verifying a download yourself

Composer keeps downloaded ZIPs in its cache (`composer config cache-files-dir`). Each package version's SHA-256 is published in the repository metadata under `extra.camp.zip-sha256` and on the plugin page's verification ledger, so any ZIP can be checked against the public index with `sha256sum`. Composer's own checksum field is not used, because it supports SHA-1 only.

### Name clashes with Packagist

A few maintainers also publish to Packagist under the same package name. With the setup above, camp is listed before Packagist and wins for any name both carry, so you get the verified build. To prefer Packagist for a specific package, mark the camp repository non-canonical (`composer config repositories.camp '{"type":"composer","url":"https://camp-registry.org","canonical":false}'`) and Composer resolves across both as usual.

### Mirrors and removal

A mirror serves the same file tree, so pointing the repository entry at a mirror's URL is the only change ([MIRRORING.md](https://github.com/camp-registry/camp-docs/blob/main/MIRRORING.md)). To stop using camp: `composer config --unset repositories.camp`. Installed plugins stay where they are.

## tool_camp (Moodle 4.5 and later, any site)

Most Moodle sites are not Composer-managed. For them, [tool_camp](https://github.com/camp-registry/moodle-tool_camp) is an admin tool plugin, installed once through Moodle's normal ZIP upload, that browses one or more camp-format repositories from inside Site administration, installs through Moodle's own plugin deployment, verifies every download against the repository's SHA-256 before writing a file, and warns when an installed plugin has an open advisory. It enforces a site policy you set: a minimum trust tier, an optional release cooldown, and filtering to your Moodle branch. tool_camp is alpha; its README has the current state.

## ZIP download (any site)

Every verified plugin page has a Download ZIP button for each release. Install the file through Site administration, Plugins, Install plugins, as with any plugin ZIP. The SHA-256 on the page's verification ledger lets you confirm the file before uploading it.

## Where the metadata lives

For anyone building tooling: the Composer repository is `https://camp-registry.org/packages.json` (packages inline, plus per-package files under `/p2/`, plus `security-advisories.json` as a whole feed). Every artifact URL is under `https://artifacts.camp-registry.org/`. The formats are documented in the [RFC](https://github.com/camp-registry/camp-docs/blob/main/rfc-community-plugin-repository.md), section 6.
