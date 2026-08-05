# Registry roles

Who does what in the registry, how someone takes on a role, and where
each role's authority ends. Three roles form a ladder; each admits from
demonstrated work in the one below, though direct entry stays possible
for people with an established record elsewhere in the Moodle community.

Applications for any role are private (volunteer@camp-registry.org, or
a direct message to the lead in the CAMP Matrix room) and are never
published, accepted or not. Appointments are announced publicly when
someone accepts a role. Declining or leaving a role carries no public
record beyond the appointment ending.

## Contributors

Anyone, no privileges, no application. Contribution is showing up on
the work itself: evidence on the review queues, seeding verification,
documentation and display-name fixes, client testing, mirrors. The
contribute page on the site lists the current lanes. This work is also
how the registry gets to know people: admissions to the roles below
point at it.

## Registry operations

The day-to-day help: working claim and release pull requests, removal
and seed requests, repoint evidence, author questions, and issue
triage across the registry repositories.

- **Procedure**: the public runbooks (camp-docs runbooks/) are the
  operations manual. Each runbook ends its judgment at an explicit
  escalation list; anything on those lists goes to the lead. An
  operator never needs to guess where their authority ends.
- **Access**: GitHub triage permission on camp-index (label, respond,
  assign, close). No code write. Registry acts that change the index
  (removals, repoints, seeds) are prepared by the operator as pull
  requests and merged by the lead; the operator's work product is a
  prepared decision, the lead's click is the act.
- **Bar**: a track record of correct queue work as a contributor, or
  an established identity in the Moodle community; organization
  two-factor enforcement applies.
- **Admissions**: private application, decided by the lead against
  this bar; outcome announced on acceptance.
- **Tenure**: lapses after sustained inactivity; step down any time by
  telling the lead; removal for cause is the lead's call.

## Review board

The code reviews behind the registry's Reviewed tier (tier 3).

- **Scope of a review**: defined by the review checklist the board
  maintains publicly (code standards, security lint findings, plugin
  correctness against its declared metadata). A camp review is not a
  commissioned security audit; the registry's security-review display
  (MDLShield) remains a separate, registry-level surface.
- **Access**: reviews land as pull-request approvals and recorded
  review documents. Board membership does not grant write access to
  the index; merges remain with the lead.
- **Conflicts**: board members do not review plugins they maintain,
  contributed to substantially, or that their employer distributes;
  conflicts are declared on the review.
- **Bar**: demonstrated Moodle code-review competence (registry
  operations work, plugin maintainership, or an established review
  record elsewhere), identifiable identity, organization two-factor.
- **Admissions**: private application, decided by the lead with the
  sitting board against this bar; outcome announced on acceptance.
- **Tenure**: as registry operations.

## The lead

Holds the merge rights on the index, the final call on escalations,
namespace determinations, admissions, and removals for cause. The
runbooks' escalation lists and this document are the public record of
what only the lead decides.
