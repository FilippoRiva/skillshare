---
name: remote-file-sync
description: "Transfer files over SSH with rsync, troubleshoot authentication, and optionally watch individual files for changes."
version: 3.0.0
license: MIT
---

# Remote File Sync (SSH + rsync)

Merged from the local remote-file-sync and ssh-file-sync skills.

## When to use

- Pull or push specific files between a local machine and an SSH host.
- Fetch a remotely built artifact, such as a PDF, without mounting a whole filesystem.
- Troubleshoot repeated SSH password or key-passphrase prompts.
- Set up an explicitly requested watch function with change-triggered actions.

This is one-way transfer guidance, not automatic two-way synchronization.
Installing this skill does not create services, keys, or background jobs.

## Scope and safety

1. Confirm the host, account, source, destination, and transfer direction. Inspect
   paths before writing; transfers can overwrite existing destination files.
2. Do not install packages, create keys, change SSH configuration, edit shell
   startup files, or start persistent watchers without user authorization.
3. Never copy private keys, passwords, tokens, or personal host configuration into
   a skill, repository, logs, or chat. Do not disable host-key verification.
4. Do not add `--delete` by default. Preview bulk or destructive operations with
   `--dry-run --itemize-changes`, and obtain approval for deletions.
5. Prefer ordinary rsync temporary-file replacement. Avoid `--inplace` by default:
   an interrupted transfer can leave the destination partially updated, and other
   programs may read it while it is changing.

## Choose a transfer method

Use rsync over SSH for selective or repeated transfers; use scp for a simple
one-off copy if appropriate. An SSHFS mount can be useful for interactive access
across many files, but is unnecessary for fetching a single artifact. Performance
is workload- and network-dependent; do not claim SSHFS is always slower.

Both ends need rsync for an rsync-over-SSH transfer. If it is missing, explain the
requirement and ask before installing it. On Arch, packages are `rsync` and
`openssh`. An SSH client does not require enabling a local SSH server.

## One-shot transfer

The following are templates: replace the host and paths with confirmed values.
Quote paths. Modern rsync's `-s` protects remote arguments, including spaces.

Pull a file; create the local destination directory only if requested:

```sh
mkdir -p "$HOME/downloads"
rsync -ai --partial -s --dry-run -- user@host:/absolute/path/report.pdf "$HOME/downloads/report.pdf"
# After reviewing the preview, run the same command without --dry-run.
```

Push a file (the remote parent directory must already exist):

```sh
rsync -ai --partial -s --dry-run -- ./report.pdf user@host:/absolute/path/report.pdf
```

- `-a`: archive mode, including timestamps; consider permissions/metadata needs.
- `-i`: itemize changes, useful for previews and change-triggered actions.
- `--partial`: retain a partial transfer to help subsequent retries.
- `-z`: optional compression; often unnecessary for PDFs, images, and archives.
- For directories, `source/` copies contents; `source` copies the directory itself.
- Default change detection uses size and modification time. Use `--checksum`
  when content comparison is necessary, understanding that both ends must read files.

## SSH authentication without repeated prompts

Distinguish the remote account password from a private-key passphrase. Check
existing keys and the SSH agent before generating anything. Do not delete or
regenerate keys merely to remove a passphrase; other hosts may depend on them.

For a new dedicated key, only after approval, choose an unused path and use an
Ed25519 key with a passphrase. Add it to the existing SSH agent if appropriate.
Install only the public key on the intended host, verifying its fingerprint via
an independent trusted source before accepting a new host key.

```sh
ssh-keygen -t ed25519 -f ~/.ssh/example-host_ed25519
ssh-add ~/.ssh/example-host_ed25519
ssh-copy-id -i ~/.ssh/example-host_ed25519.pub user@host
ssh -i ~/.ssh/example-host_ed25519 -o IdentitiesOnly=yes -o BatchMode=yes user@host true
```

The final command tests noninteractive access; failure is diagnostic, not a
reason to weaken host-key checks. An empty passphrase is a security tradeoff for
explicitly approved unattended automation, not the default recommendation.

For a non-default key path, an approved `~/.ssh/config` entry can select it:

```sshconfig
Host example-host
    HostName actual-hostname
    User account
    IdentityFile ~/.ssh/example-host_ed25519
    IdentitiesOnly yes
```

Use that alias with rsync, or select the key explicitly:

```sh
rsync -ai --partial -s -e 'ssh -i ~/.ssh/example-host_ed25519 -o IdentitiesOnly=yes' -- user@host:/absolute/path/report.pdf ./report.pdf
```

SSH connection multiplexing can reuse an authenticated connection and reduce
latency and repeated prompts while that connection remains alive. It does not
provide credentials for a fresh connection; keys plus an agent are the usual
interactive solution.

## Optional change watcher (Bash/Zsh)

Use only when requested. First verify noninteractive SSH works and the local
parent directory exists. This foreground function polls one file and prints only
when rsync reports a transfer or metadata change. It preserves rsync errors
instead of hiding them or treating a failed transfer as success.

```sh
watch_remote_file() {
    local remote="$1" destination="$2" output status
    while true; do
        if output=$(rsync -ai --partial -s -e 'ssh -o BatchMode=yes' -- "$remote" "$destination"); then
            if [ -n "$output" ]; then
                printf '%s\n' "$output"
                # Put an explicitly requested notification/viewer action here.
            fi
        else
            status=$?
            printf 'rsync failed (exit %s); stopping watcher\n' "$status" >&2
            return "$status"
        fi
        sleep 2
    done
}
```

Call with two arguments, for example a confirmed `user@host:/path/report.pdf`
and local file path. Stop with Ctrl+C. `sleep 2` means two seconds.

Never paste a bare infinite loop into `.zshrc` or `.bashrc`: it blocks shell
startup. Defining a function does not start it. For requested persistent polling,
consider a systemd user service/timer with failure visibility rather than silently
backgrounding a shell loop. Never enable it without approval.
