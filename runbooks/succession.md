# Succession and root custody

What exists, who holds it, and how signing continues without any one person. Companion to the ceremony record (governance/root-ceremony-2026-09-18.md) and the steward instructions.

## Asset inventory

| asset | where | who can act |
| --- | --- | --- |
| TUF root (v1, threshold 3 of 4) | camp-index `tuf/root.json`, `tuf/1.root.json`; public | any three stewards, together |
| Steward root keys (ECDSA P-256) | inside each steward's hardware token; never exported | the steward alone, PIN plus touch |
| Online role keys (targets, snapshot, timestamp; ed25519) | camp-index Actions secrets `TUF_TARGETS_KEY`, `TUF_SNAPSHOT_KEY`, `TUF_TIMESTAMP_KEY`; public halves in the root | camp-index publish workflow; repository admins can replace them |
| Host tooling | camp-tools `camp tuf root build/check/assemble`, `camp tuf sign` (#45) | anyone with a checkout; needs no secrets for build/check/assemble |
| Ceremony working folder | registry host `_PluginDirectory/ceremony/` (public keys, payloads, signatures; nothing private) | registry lead |

Stewards, as of the v1 ceremony: Mike Churchward (mchurchward), Alexander Bias (abias), Jordan Tomkinson (durzo), David Pesce (davidpesce). Key ids are in the ceremony record and in `tuf/root.json`.

## Roles

- **Host / coordinator:** builds the payload, collects signatures, assembles, publishes the record. Currently David Pesce. If the host is unavailable for a scheduled signing, the signing is rescheduled; there is no standing backup coordinator yet. Any steward could take the role given a checkout of camp-tools and this runbook, and that is the designated fallback if the host is unavailable for longer than the root's remaining validity allows.
- **Stewards:** sign only what they have hashed and had explained. Three of four are required for any root change.

## Routine: annual re-sign

Trigger: eleven months after the last root signing (calendar entry owned by the host). Fifteen minutes.

1. Host: `camp tuf root build OUT --steward … (all current) --online-keys <public halves> --threshold 3 --previous tuf/<current>.root.json`. Rotate the online keys at the same time as a matter of course: generate new ones, load as secrets, list the new public halves here.
2. Distribute `root.json` and `root-payload.bin`; read the payload SHA-256 aloud; everyone confirms and confirms the diff (`camp tuf root check --previous`) shows only the expected changes.
3. Stewards sign (handbook step 4); host runs `camp tuf root assemble OUT --sig … --previous tuf/<current>.root.json`. Both thresholds must be met.
4. Commit `OUT/root.json` as `tuf/root.json` and `OUT/<n>.root.json`; append to the ceremony record or write a new dated record.

The build, read-back, sign and assemble steps are the same sequence as the initial ceremony (see the procedure section of the 2026-09-18 record); the only differences are `--previous <current root>` on both build and assemble, and that a rotation lists a changed steward set.

## Rotation: add, replace or remove a steward

Same steps as the re-sign with the steward list changed. Requirements: at least three signatures from keys the *current* root lists (the new key cannot vouch for itself), threshold stays 3 unless the group decides otherwise and records why. A steward who leaves should give notice; their key simply stops being listed.

## Emergency: online key compromise

Any indication that a publish-side key leaked (unexpected metadata, a CI incident): replace all three online keys at once. Generate new keys, load the secrets, run the rotation above with only the online keys changed, and re-run publish so fresh targets/snapshot/timestamp appear under the new root within the day. No steward key is involved beyond signing the new root.

## Emergency: steward key lost or unavailable

Nothing is at risk from a single lost key. The steward generates a new key when able and a rotation swaps it in. Two keys unavailable at once still leaves a working threshold with four seated; the fifth steward brings that margin to two.

## Recovery: threshold cannot be met

If fewer than three current keys can sign (multiple losses, or stewards unreachable for longer than the root's validity), the root cannot be updated by signature. Recovery is a public re-bootstrap: convene new stewards, run a fresh ceremony producing a new root version 1, publish the record with the reason, and ship a client (tool_camp) release that pins the new root. Sites update through the normal plugin update path. This is deliberately visible and slow; it is the reason the routine paths above exist.

## What a successor needs to run any of this

- A checkout of camp-tools (`camp tuf root --help`) and the public halves of the online keys (in the current root).
- The steward instructions (root-steward-handbook) to send to signers.
- Admin on camp-index to load or replace the online secrets and to commit under `tuf/`.
- This runbook and the ceremony record for the key ids and the last dates.
