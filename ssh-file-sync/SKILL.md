---
name: ssh-file-sync
category: software-development
description: Use when syncing files with a remote over SSH.
---

# Fast file sync over SSH

Purpose: pull/push a specific file (e.g. a build artifact like a thesis PDF) from a remote host to local without mounting the whole remote filesystem. sshfs on a large tree is sluggish (FUSE round-trips + directory traversal) — for one file, skip it entirely with rsync over ssh.

## When to choose rsync-over-ssh over sshfs
- Only one (or few specific) files need to be local; everything else stays remote.
- Remote tree is large; the FUSE local-daemon overhead and per-stat traffic make interactive/remote work laggy.
- Latency matters (e.g. Tailscale links). sshfs pays traversal cost every mount; rsync touches only the target path.

## Procedure
1. Confirm the exact remote path exists first (`search_files` / `ls`) so the command doesn't fail on a typo.
2. One-shot sync of a single file:
   ```
   rsync -avz --partial --inplace user@host:absolute/path/file.ext ./file.ext
   ```
   - `--partial` + `--inplace`: resume an interrupted transfer and write in place instead of a temp copy.
   - `-z` is usually a wash on already-compressed files (PDFs, images) — drop it if CPU is the bottleneck.
3. Watch-and-report loop for a file that updates periodically (e.g. a document you recompile):
   ```
   while true; do
     if rsync -aiz --partial --inplace user@host:path/file ./file 2>/dev/null | grep -q '.'
     then echo "[$(date +%H:%M:%S)] updated"; fi
     sleep 2
   done
   ```
   `-i` (itemize-changes) prints a line only for files that actually differ, so `grep -q '.'` stays silent between updates.

## Pitfalls
- Pasting a bare `while true` loop into .zshrc/.bashrc executes it at every shell startup and hangs the session. Wrap it in a function the user invokes (`mysync() { while true; do ...; done }`), or a background job.
- `sleep` takes SECONDS by default (60 = one minute; `sleep 2` for near real-time).
- A loop that calls ssh/rsync repeatedly will prompt for the password every call (every 2s = unusable). Fix permanently with key auth:
  1. `ssh-keygen -t ed25519` (local; `-t ed25519` = modern Ed25519 keytype — preferred over rsa/ecdsa).
  2. `ssh-copy-id user@host` (prompts for password once).
  3. Verify: `ssh user@host 'echo ok'` prints ok without prompting.
  - Optional: SSH control socket in ~/.ssh/config (`ControlMaster auto; ControlPath ~/.ssh/cm-%r@%h:%p; ControlPersist 10m`) so repeats reuse one connection instead of re-handshaking — helps latency, does NOT remove the password prompt. Only keys fix the password prompt.
- Make sure the local destination directory exists before rsync, or it errors out (`mkdir -p ~/dst`).
- If timestamps trail actual changes under a slow loop, prefix the loop command with `stdbuf -oL` to defeat output buffering.
