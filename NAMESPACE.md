# Component names: allocation, collisions, and disputes

How camp allocates Moodle component names ("frankenstyle", e.g.
`mod_example`), what happens when two plugins want the same one,
and how disputes are resolved. This operationalizes RFC §8 (component
namespace) and §9 (governance). Modeled loosely on PyPI's PEP 541 and
npm's package-name dispute policy, scaled to this ecosystem.

## Why the registry has to care

Moodle component names are a flat namespace: a site cannot install two
plugins with the same component, the index keys every listing by
component (`plugins/<type>/<component>.yml`), and the client and Composer
channels resolve installs by component. One listing per name is a
technical constraint, not a policy preference. The moodle.org Plugins
Directory historically acted as the name registry and its approval queue
caught collisions before they shipped; with that gone, collisions arrive
here. The registry's first collision (July 2026) involved two unrelated
plugins that independently chose the same name.

**Before naming a new plugin, search camp-registry.org for the component
name you want.** The registry is now the ecosystem's de facto
availability check.

## Principles

1. **Discovery is not reservation.** A Tier 0 listing exists because the
   scanner found a public repository. It records that a repo uses a
   name; it does not grant the name. Crawl order allocates nothing.
   (Precedent: a seeded listing that had raced to a fork was later
   repointed to the plugin's canonical repository.)

2. **Names are earned by publication, held by use.** The strength of a
   party's stake in a name, in increasing order:
   - a public repository using the name (Tier 0, informational only);
   - a claimed listing (Tier 1, accountability established);
   - verified releases through the registry (Tier 2+);
   - a real install base: sites that would break if the name changed.

   The registry allocates a contested name to the party with the
   materially stronger stake; between comparable stakes, first to
   publish a working release under the name prevails.

3. **A claim is not a transfer.** Claiming a listing requires control of
   the **listed source repository**. A claim from a different repository
   that happens to use the same component name is not a claim; it is a
   collision, and it routes to the process below. This closes both doors:
   squatting (reserve a name by creating a repo) and claim-jumping (take
   a name by claiming someone else's listing).

4. **Disputes are resolved publicly** (RFC §8): in issues on camp-index,
   labelled `name-dispute`, decided by the review board (during
   bootstrap, by the maintainers), with the reasoning written down
   either way. No private adjudication.

5. **Renaming is the cheap exit.** Before a plugin has an install base, a
   component rename costs almost nothing; afterwards it breaks upgrades
   for every site running it. Collisions are therefore resolved eagerly,
   at claim time rather than after distribution, and the party that
   renames gets help with the mechanics and keeps its listing under the
   new name.

## Procedure

**Detection.** A collision is recorded when the scanner finds a second
repository declaring an already-listed component, or when anyone reports
one. The colliding repository is never silently dropped: it is noted on
the ledger and a `name-dispute` issue is opened on camp-index.

**Case A: the listed party is an unclaimed Tier 0.** The registry posts
a notice issue on the listed repository explaining the collision and the
options (keep-and-dispute, rename, or request removal), with a
**14-day** response window.

- No response, or the listed party concedes: the name is uncontested.
  The listing's `source` is repointed to the other project on a verified
  claim; the previous repository may list under a different component
  name at any time.
- The listed party wants the name: proceed to Case B.

**Case B: both parties actively want the name.** The review board
decides in the public `name-dispute` issue, weighing in roughly this
order:

1. install base and distribution under the name (strongest);
2. verified releases through the registry;
3. first public working release under the name (tag date of the earliest
   installable release, not repository creation date);
4. evidence the project is a maintained, working plugin (activity,
   maturity, responsiveness);
5. bad faith disqualifies outright: reserving names without a working
   plugin, mass registration, registering a name to block or impersonate
   another project, or typosquatting (RFC §8's similarity check applies
   to new names generally).

The criteria are ordered, not mechanical: the board may depart from
them with written reasons (e.g. an earlier tag from an abandoned
experiment does not outweigh a maintained plugin with users).

**Aftermath.** The losing party renames; the registry assists and lists
the renamed plugin normally. The resolution stays linked from the
`name-dispute` issue. Decisions can be revisited if made on facts that
turn out to be wrong.

## Squatting

A component name may not be held by anything that is not a working
plugin. The board may reclaim a name whose listing exists only to occupy
it. Departed authors are the exception, by standing policy: a maintainer
who leaves via `status: moved` keeps their component name against their
return (AUTHORS.md). Published history is a real stake; an empty repo is
not.
