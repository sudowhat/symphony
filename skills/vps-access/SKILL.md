---
name: vps-access
description: "Generic steps for reaching and changing the shared VPS that hosts several Symphony projects: where the private host notes live, non-interactive SSH, the user's standing authorization and its care steps, verify and rollback. Load before any SSH, SCP, deploy, nginx or service work on that host, whatever the project."
---

# VPS access (all projects)

So no agent starts a VPS task without knowing how to reach the host. **Symphony is a public
repository**: this file holds only the generic steps. The host itself is described in the private
project files.

## Where the host details are (private)

- `whatdate-folder/SKILL.md` → "WhatDate relay VPS access": the host, the login, the SSH key file to
  use on this PC, and what each project keeps where on the shared host (code, data, services, nginx
  sites, public URLs).
- `wisdom_capsules-folder/SKILL.md` §12 and `MEMORY.md` (2026-08-22): the Wisdom Capsules release
  layout and its launch gate.

Both are private repositories. **Never copy a host address, login name, key file name or path,
server directory, service name or site name into Symphony or any other public repository.**

## SSH

- Never read, print, copy, commit or paste a private key; OpenSSH uses it by path only.
- Always non-interactive, so a missing key fails fast instead of hanging on a prompt:

```sh
K="-i <key path from the private notes> -o IdentitiesOnly=yes -o BatchMode=yes -o ConnectTimeout=15"
ssh $K <user>@<host> 'systemctl is-active <service>'
scp -q $K <local files> <user>@<host>:/tmp/<ticket>-staging/
```

- Quote remote commands in single quotes; a heredoc or a small script copied to `/tmp` is clearer
  than nested quoting. Git Bash rewrites `/paths` in some arguments: set `MSYS_NO_PATHCONV=1` when a
  remote path is mangled.

## Authorization

- **User standing rule, 2026-09-24:** "henceforth do not ask me permission to use VPS - just say you
  did it". Read-only checks, deploys, nginx edits, service restarts and file copies need no prior
  question: do them, then report exactly what changed, where the backups are and what was verified.
  Repeated 2026-10-08.
- **Not covered:** destroying data (deleting or overwriting a database, revoking certificates or
  keys), and changes to how the host itself is reached (SSH, firewall, accounts). State what you
  intend and get the user's answer first. A schema migration that keeps every row is not
  destruction, but say it is coming, with the backup path.
- **Project launch gates still apply.** A public launch or a switch of a project's public release
  follows that project's own SKILL.md and launch documents.
- An agent harness may still stop a production change (for example an auto-mode classifier). Do
  not work around it: stop, tell the user what you were doing and give them the exact commands.

## Care steps (every change)

1. **Look first, read-only:** what is deployed (`md5sum` of the files you will replace, compared
   with the commit you will ship), service state, schema version, disk space.
2. **Name the commit** you are deploying in your report.
3. **Stage in `/tmp/<ticket>-staging/`** (mode 700). For a database change, **dry-run first**: copy
   the live database with SQLite's backup API (safe while the service runs), run the new code's
   migration against the copy, compare row counts, run `PRAGMA integrity_check` and
   `PRAGMA foreign_key_check`, and open it twice to prove the migration is idempotent.
4. **Back up** every file you replace as `<name>.bak-<ticket>` (or the whole directory as
   `<dir>.bak-<ticket>-<yyyymmdd>`), and the database with the backup API as
   `<db>.bak-<ticket>-<yyyymmdd>` with the service user's owner and mode 600.
5. **Install** with `install -o <owner> -g <group> -m <mode>` so ownership matches what was there.
6. **nginx:** `nginx -t` before every reload; never reload on a failed test. Keep each project's
   site file separate.
7. **Restart only the service you changed**, then read its journal
   (`journalctl -u <service> -n 30 --no-pager`).
8. **Clean up** `/tmp/<ticket>-staging/`, especially any database copy.

## Verify (every change)

- The changed service: `systemctl is-active`, its journal, and the endpoints you meant to change
  (expected status codes, before and after).
- **Every other site on the host:** record each public URL's status code before and after (the
  private notes list them). A change that broke a neighbour is not done.
- For a migration: schema version and row counts on the live database, read-only
  (`sqlite3` URI `file:<db>?mode=ro`).

## Rollback

Stop the service, restore the `.bak-<ticket>` files (and, only if the migration itself is at fault,
the database backup, saying first that writes since the backup will be lost), start it, and verify
as above.

## Secrets

Environment files and tokens on the host are never printed, copied off the host, or committed.
Check that a variable is set by name only.
