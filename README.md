# camp — design documents

**camp** is a proposed community-governed plugin repository for Moodle:
open, mirrorable, cryptographically verifiable, and not controlled by any
single company. Start here:

- **[The RFC](rfc-community-plugin-repository.md)** — the full proposal,
  open for community comment in
  [Discussions](https://github.com/camp-registry/camp-docs/discussions)
- **[AUTHORS.md](AUTHORS.md)** — the plugin developer's path from claimed
  listing to verified releases (steady state: push a git tag)
- **[DESIGN.md](DESIGN.md)** — the running log of implementation decisions
  (D1–D21) and their rationale
- **[MIRRORING.md](MIRRORING.md)** — how to run a mirror
- **[REPO-SPLIT.md](REPO-SPLIT.md)** — how the development monorepo became
  these repositories (historical)

The implementation lives at
[camp-index](https://github.com/camp-registry/camp-index) (the registry
data), [camp-tools](https://github.com/camp-registry/camp-tools) (the CLI),
and [moodle-tool_camp](https://github.com/camp-registry/moodle-tool_camp)
(the Moodle client plugin).

Status: **pre-launch, request for comments.** The trust model, tiers, and
security pipeline described in the RFC are implemented and tested; keys,
governance, and the public index open with the launch checklist in the RFC's
roadmap (§11).
