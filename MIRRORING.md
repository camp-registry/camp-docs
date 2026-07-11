# Running a camp mirror

The published repository is a plain file tree (RFC §4.5): plugin ZIPs,
Composer metadata, TUF signatures, advisories, and the website. There is no
database or application server in the download path, so a full mirror is
one sync job on a cron schedule.

## Why mirrors don't need to be trusted

Clients verify content, not servers:

- Every artifact download is checked against the SHA-256 recorded in the
  signed repository metadata before installation (both the Composer channel
  and the tool_camp client plugin do this).
- Repository metadata itself is TUF-signed (RFC §4.3); once client-side TUF
  verification lands, a mirror cannot alter metadata either.
- The worst a malicious or compromised mirror can do is serve *stale* or
  *missing* files — never tampered ones. TUF timestamp metadata expires
  daily, which bounds how stale "stale" can be before clients notice.

## What you need

- A web server that can serve static files over HTTPS (any of nginx,
  Apache, Caddy, an object-storage bucket behind a CDN).
- Disk: the full archive is dominated by plugin ZIPs. Budget tens of GB;
  every version of every listed plugin is preserved permanently, so plan
  for monotonic growth.
- A cron job. That's all.

## Sync

```
# hourly is plenty; timestamp metadata is valid for a day
17 * * * *  rsync -az --delete rsync://<primary>/camp/ /srv/camp/
```

`--delete` is safe *for distribution*: removed files are ones the registry
deliberately withdrew from installation channels. Mirrors that also want to
be archival keep a second, append-only copy without `--delete`.

Serving is then just pointing your web server's document root at the
synced tree. No rewrites, no dynamic handlers. Set long cache lifetimes on
`artifacts/` (content-addressed by version, effectively immutable) and
short ones on metadata (`packages.json`, `security-advisories.json`, TUF
metadata, the website).

## Privacy expectations (RFC §4.6)

Mirror operators see only anonymous file downloads. Please match the
project's data-minimization posture: keep access logs short-lived (or
disabled), don't attempt to correlate downloads into per-site plugin
inventories, and publish your log-retention policy if you can. An update
check reveals which plugins a site runs; collectively that is a map of the
security posture of thousands of institutions, which is why the project
treats download logs as sensitive.

## Becoming a listed mirror

At launch the project will list 3–5 institutional mirrors in different
countries (RFC §4.5). Listed mirrors agree to: HTTPS, a sync interval of at
most 24 hours, and the privacy expectations above. Contact details will be
published with the launch announcement; until then, this document is
preparation.
