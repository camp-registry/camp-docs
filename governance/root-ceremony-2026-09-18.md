# Root signing ceremony, 18 September 2026

This record documents the creation of camp's TUF root: the set of hardware keys whose signatures, at a threshold, define which keys may sign the registry's published metadata (RFC 4.3, DESIGN.md D9). Design: camp-tools#47. Tooling: camp-tools#45.

## Outcome

- Root version 1, threshold 3 of 4, expires 2027-09-18 (one year).
- Canonical root payload SHA-256: `e9c53ce416c4ef703dc8c5a76c90e1230aa73f186d73ece905647242e498cf1c`
- Signed root: camp-index `tuf/root.json` (commit `7228e421294a951846f47f279b86bca0438ef9ad`), also `tuf/1.root.json`.
- Signatures attached: 4 of 4 (three are required; all four were collected when possible).
- The unsigned root and canonical payload exactly as distributed on the call are kept beside this record in `root-ceremony-2026-09-18/`.

## Stewards

| steward | GitHub | key id (TUF keyid, ECDSA P-256) | signed |
| --- | --- | --- | --- |
| Mike Churchward | mchurchward | `52115d7b55867b05521eeb2a9f9df31c18f59766dfe9e1580db76262ef8b214d` | yes |
| Alexander Bias | abias | `b562ebaa9f458716f2178094c4b5f37e16e42702a91501398a959ad288ffac5b` | yes |
| Jordan Tomkinson | durzo | `ba474a69e6fea713abdb3741e1b0a2a590cec24f644d2f1d59d5686795cd4a1b` | yes |
| David Pesce | davidpesce | `f3e0ad3d6b3b8ac8e4294b6f281c4bc8008357dcee3c83f24d24d413caebd1f7` | yes |

Each key was generated inside the steward's own hardware token during the call (PIV slot 9c, PIN required for every signature, touch required) and has never existed outside it. A fifth steward will be added by a root rotation signed by at least three of the keys above; the threshold stays three.

## Online role keys listed in this root

| role | key id (ed25519) |
| --- | --- |
| targets | `96f2d28a2116c03923dbc6c0fd0b53f910f5488166ef072acdd26cbec3492f89` |
| snapshot | `b1962ea6259e7c0f6f3b8198bcc94aafaf30fd0eb0f27dfe4599e9e7bae9758a` |
| timestamp | `bac44af50f6a6848cbd19d021a1768e35ccab67929aa0df7be53258e1c6b9184` |

Generated 8 September 2026 in memory on the registry host; private halves exist only as camp-index Actions secrets (`TUF_TARGETS_KEY`, `TUF_SNAPSHOT_KEY`, `TUF_TIMESTAMP_KEY`); rotated at each annual re-sign as routine. These roles are re-signed on every publish; their validity periods are set by the publish pipeline, not by this root, and are recorded in the design issue (camp-tools#47) when the pipeline starts signing (phase 2).

## Procedure followed

1. Each steward generated a key and published its public half in the call.
2. The host built the unsigned root (`camp tuf root build`) from the four public keys, the online public keys, threshold 3 and a one-year expiry, and distributed `root.json` and the canonical `root-payload.bin`.
3. Every participant computed the payload's SHA-256 independently and read it aloud; all matched before anyone signed.
4. Each steward signed the payload on their token (`pkcs11-tool`, ECDSA-SHA256, DER) and returned the signature.
5. The host attached the signatures (`camp tuf root assemble`); each verified against its key and the threshold was met. The result was shown to all participants.
6. The signed root was committed to camp-index and this record published.

Call: Google Meet, 13:00–13:50 UTC, not recorded. Participants: the four stewards. Host: David Pesce.

## Next dates

- Annual re-sign: by 2027-08-18; 15-minute call, same procedure from step 2 with a new expiry.
- Fifth-steward rotation: when seated; same procedure with `--previous tuf/1.root.json`, signed by at least three current keys.
