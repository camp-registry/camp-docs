# Registry roles

Who does what in the registry, how someone takes on a role, and where
each role's authority ends. One principle underlies every rule here:
no one certifies their own work. Admins do not merge changes they
authored, review board members do not review plugins they maintain,
and authors do not verify their own releases.

Two tracks grow from a common base. Contributors who take up queue
work become registry operations, and a sustained record in operations
is the path to admin. The review board is a separate specialist track
for code review: a destination in its own right, not a step toward
admin. Roles are appointed for the work they do, not as steps toward
other roles. Direct entry to any role stays possible for people with
an established record elsewhere in the Moodle community.

Applications for any role are private (volunteer@camp-registry.org,
or a direct message to an admin in the CAMP Matrix room) and are
never published, accepted or not. Appointments are announced publicly
when someone accepts a role. Declining or leaving a role carries no
public record beyond the appointment ending.

## Contributors

Anyone, no privileges, no application. Contribution is showing up on
the work itself: evidence on the review queues, seeding verification,
documentation and display-name fixes, client testing, mirrors. The
contribute page on the site lists the current lanes. This work is also
how the registry gets to know people.

## Registry operations

The day-to-day help: working claim and release pull requests, removal
and seed requests, repoint evidence, author questions, and issue
triage across the registry repositories.

- **Procedure**: the public runbooks (camp-docs runbooks/) are the
  operations manual. Each runbook ends its judgment at an explicit
  escalation list; anything on those lists goes to the admins. An
  operator never needs to guess where their authority ends.
- **Access**: GitHub triage permission on camp-index (label, respond,
  assign, close). No code write. Registry acts that change the index
  (removals, repoints, seeds) are prepared by the operator as pull
  requests and merged by an admin; the operator's work product is a
  prepared decision, the admin's click is the act.
- **Bar**: a track record of correct queue work as a contributor, or
  an established identity in the Moodle community. The GitHub
  organization requires two-factor authentication with secure
  methods (authenticator app, passkey, or hardware key; not SMS).
  GitHub enforces this mechanically: an account without it cannot
  join the organization, so it applies to every role with access.
- **Admissions**: private application, decided by the admins against
  this bar; outcome announced on acceptance.
- **Tenure**: lapses after sustained inactivity; step down any time by
  telling an admin; removal for cause is a reserved decision of the
  admins.

## Review board

The code reviews behind the registry's Reviewed tier (tier 3).

- **Scope of a review**: defined by the review checklist the board
  maintains publicly (code standards, security lint findings, plugin
  correctness against its declared metadata). A camp review is not a
  commissioned security audit; the registry's security-review display
  (MDLShield) remains a separate, registry-level surface.
- **Access**: reviews land as pull-request approvals and recorded
  review documents. Board membership does not grant write access to
  the index; merges remain with the admins.
- **Conflicts**: board members do not review plugins they maintain,
  contributed to substantially, or that their employer distributes;
  conflicts are declared on the review. A board member's own plugin
  can still reach the Reviewed tier: another member reviews it under
  the same checklist.
- **Bar**: demonstrated Moodle code-review competence (registry
  operations work, plugin maintainership, or an established review
  record elsewhere), identifiable identity, organization two-factor.
- **Admissions**: private application, decided by the admins with the
  sitting board against this bar; outcome announced on acceptance.
- **Tenure**: as registry operations.

## Admins

The registry's maintainers: merge rights on the index, and the
authority the runbooks route upward.

- **Merges**: any admin may merge a pull request that passed
  verification. The dividing line is what the checks establish, not
  who authored the change. Pull requests whose assertions the
  verification checks fully establish (release publications from the
  pipeline today; further classes only as the public design record
  admits them, camp-index#210) are reviewed by the checks alone and
  may merge automatically as a registry act; an admin may also merge
  them by hand, even for a plugin they maintain, and any admin may
  hold, close, or revert one. Pull requests that require human
  judgment are merged by an admin who did not author them.
- **Reserved decisions**: namespace determinations and removals for
  cause. Each is recorded in writing with the concurrence of two
  admins before it is enacted, and the enacting change is merged by
  an admin other than its drafter. When the admins disagree, the lead
  decides, and the record says so.
- **Admissions**: private application, decided by the sitting admins.
  Admins are admitted one at a time. Admission to any role is decided
  by the admins and is never put to a vote of the wider roles.
- **Bar**: a sustained record of correct registry operations work, or
  an established identity in the Moodle community; identifiable
  identity; organization two-factor enforcement.
- **Tenure**: lapses after sustained inactivity; step down any time;
  removal of an admin for cause is a reserved decision of the other
  admins.

## The lead

One of the admins serves as lead. The lead breaks ties among the
admins, holds the final call on escalations, represents the registry
externally, and maintains the succession runbook: the inventory of
the registry's assets (organization ownership, domain, hosting and
service accounts, the artifact archive) and the custodian of each.
The GitHub organization keeps at least two owners at all times.

## Selecting the lead

The project began with a founding lead. This section defines how the
role passes on.

- **Triggers**: a selection runs when the lead resigns or retires;
  when the lead has been unreachable for six weeks without notice, in
  which case the admins jointly hold the lead's calls immediately and
  a selection follows; or when two thirds of the electorate call for
  one.
- **Electorate**: admins and review board members who have held their
  role for at least twelve months. A selection passes at two thirds
  of the electorate.
- **Sunrise**: while the electorate is smaller than five, this
  mechanism is dormant. Succession is then by the lead's designation
  with public notice, and the call-for-selection trigger does not
  operate.
- **Nomination**: an outgoing lead may nominate a successor. The
  nomination informs the vote; it does not decide it.
- **Alternative**: by the same two-thirds, the electorate may instead
  vest the lead's calls in the admins jointly and leave the role
  vacant.
- **Custody**: registry assets transfer only on succession under this
  section, per the succession runbook.

A design note on the numbers: the twelve-month tenure requirement,
the two-thirds threshold, and admin-decided admissions are
deliberate. They slow legitimate voices as well as hostile ones. The
registry accepts that trade because a small team is most exposed to
capture through its own growth, and these rules make a takeover
require years of correct public work.
