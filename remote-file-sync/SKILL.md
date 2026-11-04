---
name: remote-file-sync
description: "Sync files over SSH fast and fix SSH password prompts."
version: 2.1.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [ssh, rsync, sync, remote, file-transfer, devops]
    related_skills: [hermes-agent]
---

# Remote File Sync (SSH + rsync)

## When to Use

- Pull or push a single file (a PDF, a build artifact) between a local machine and a
  remote server, fast, without the lag of a full remote mount.
- Watch a remote file and trigger an action every time it changes.
- rsync/ssh keeps prompting for a password on every invocation.

Fast, low-latency syncing of individual files between a local machine and a remote
host over SSH, with change-triggered actions and passwordless key auth.

## Core procedure

### 1. Pick rsync over sshfs for one file
Mounting an entire folder with sshfs is laggy (FUSE + per-op round-trips + full
directory walks). For a single file use one-shot rsync — only that file is touched.

    rsync -avz --partial --inplace \
      user@host:path/to/file.txt ./local-copy.txt

- `--partial --inplace`: resume a dropped/interrupted transfer from where it stopped
  instead of restarting; write in place rather than via a temp file.
- `-z` (compression) is usually a wash on already-compressed data (PDFs, zips) —
  if transfer speed is bottlenecked by CPU, drop `-z` and keep the delta.

### 2. Act ONLY when the file actually changes
`--itemize-changes` (`-i`) prints a line per changed file and prints NOTHING when
the remote copy is byte-identical. Pipe to `grep -q '.'` so an action only fires on
a real update, not on every poll.

    rsync -aiz --partial --inplace user@host:path ./local 2>/dev/null \
      | grep -q '.' && echo "updated"

- Poll loop with `sleep N` — N is always SECONDS (e.g. `sleep 2` = 2 seconds).
- First run always reports if the local copy doesn't exist yet; subsequent runs stay
  silent until a change ships.
- Chain the side effect in the same `&&` branch: `&& { echo ...; open-cmd file; }`
  so the viewer/notification only fires on an actual update.
  Open a PDF in a new tab: `zen-browser --new-tab /path/file.pdf` (drop `--new-tab`
  to just focus the file).

### 3. Passwordless key auth (stop prompt-per-call)
rsync shells out to ssh, so a password prompt fires on EVERY invocation in a loop.
Fix once with key auth:

    ssh-keygen -t ed25519 -f ~/ssh-keys/host                     # -t type, -f path
    ssh-copy-id -i ~/ssh-keys/host user@host                     # prompts password THIS once

- `-t ed25519` = modern default key type (fast, short, secure) vs rsa/ecdsa.
- Generate passwordless/non-interactive with `-N ""` (empty passphrase); to re-do:
  `rm key key.pub && ssh-keygen -t ed25519 -f <path> -N ""`. A key created WITH a
  passphrase is a different key than one made with `-N""` — regenerate the whole
  pair, then reinstall, don't just strip the passphrase in place.
- `ssh-copy-id -i` takes the PRIVATE key path; it auto-appends `.pub`.

A key stored outside the default location is NOT offered automatically. Add it to
`~/.ssh/config` or ssh won't present it and the loop STILL prompts:

    Host host
        IdentityFile ~/ssh-keys/host

or pass it per-call to rsync with `-e 'ssh -i ~/ssh-keys/host'`.

## Pitfalls

- **A bare `while true` pasted into `~/.zshrc` executes at shell startup and hangs
the session.** Wrap it in a function (`name() { while true; do ...; done }`) so it
only runs when invoked; or call it with `&` to background it.
- Verify a remote key removal by grepping the count in `authorized_keys`
(`grep -c ssh-ed25519 authorized_keys` -> `0`), not by assuming the delete worked.
