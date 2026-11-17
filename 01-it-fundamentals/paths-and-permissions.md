# Paths and permissions — a filesystem and key audit

**Phase 1 · 17 November 2026**
Measured on my own workstation. Hardware profile in [workstation-inventory.md](workstation-inventory.md).

> **The principle under all of it:** where a file lives, and who is allowed to read it, are **two separate controls.** You need both. Two findings below are a failure of one or the other.

---

## 1 · Paths, and the bug they cause

### The tree

A filesystem is a tree. Windows has several roots — this machine has `C:` (237.7 GB) and `D:` (0.7 GB), two separate trees. **Linux has exactly one root, `/`**, and additional disks are attached into it at a chosen point rather than becoming a new letter. That attachment is **mounting**, and under WSL2 the Windows `C:` drive appears at `/mnt/c` — a branch of the one tree, not a second one.

### Absolute and relative

| Kind | Example | Resolved against |
|---|---|---|
| Absolute | `C:\Users\<user>\Documents\...` | the root |
| Relative | `01-it-fundamentals\process-utilisation-audit.md` | the **working directory** |

The working directory is a single piece of per-process state. `Get-Location` on Windows, `pwd` on Linux. Every relative path is resolved against it.

### Reproducing the failure deliberately

Identical path, run from two different working directories:

```powershell
cd ~\Documents\cloud-devops-engineering-journey
Get-Content 01-it-fundamentals\process-utilisation-audit.md -TotalCount 2   # works

cd ~
Get-Content 01-it-fundamentals\process-utilisation-audit.md -TotalCount 2   # fails
```

```
Get-Content : Cannot find path 'C:\Users\<user>\01-it-fundamentals\process-utilisation-audit.md'
because it does not exist.
    + CategoryInfo : ObjectNotFound: (...) [Get-Content], ItemNotFoundException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.GetContentCommand
```

**The useful information is the path in the error, not the clause at the end.** I typed a relative path; the shell glued it onto the working directory and reported the absolute path it actually tried. The file exists — it was resolved from the wrong starting point.

> **Habit:** on any path error, compare the path in the error against the path you typed. The difference is the bug. Most path failures are not missing files.

Distinguishing the two failure modes by their message:

| Message | Cause |
|---|---|
| `Cannot find path` · `No such file or directory` | wrong **location** |
| `Access is denied` · `Permission denied` | wrong **permissions** |

### Why this matters in automation

A script using relative paths works when run by hand and fails in a CI pipeline, because the runner starts in a different working directory. The script did not change; the assumption about where it would be invoked from was wrong.

The standard fix is to make a script resolve paths relative to **itself**:

```bash
cd "$(dirname "$0")"
```

---

## 2 · Permissions on an SSH private key

### What is in `~/.ssh`

| File | Size | Role |
|---|---|---|
| `id_ed25519` | **419 bytes** | private key — never leaves this machine |
| `id_ed25519.pub` | 107 bytes | public half — this is what GitHub holds |
| `known_hosts` | 8,122 bytes | recorded fingerprint of every host connected to |
| `known_hosts.old` | 7,947 bytes | previous version, retained automatically |

419 bytes is the entire credential controlling push access to this repository. `ed25519` is an elliptic-curve algorithm; an RSA key of comparable strength would be 2–3 KB.

### Who can read it

```powershell
(Get-Acl "$env:USERPROFILE\.ssh\id_ed25519").Access |
  Select-Object IdentityReference, FileSystemRights, AccessControlType
```

| Identity | Rights | What it is |
|---|---|---|
| `NT AUTHORITY\SYSTEM` | FullControl | the operating system. Unavoidable on Windows |
| `BUILTIN\Administrators` | FullControl | **any administrator account on this machine** |
| `DESKTOP-XXXXXXX\<user>` | FullControl | my own account. Correct and necessary |

*(Hostname redacted — it appears in the raw output.)*

**Three identities.** The second is the consequential one: any local administrator can read this key. On a personal machine that is me; on a managed corporate machine it would be the IT department. This is why *"an attacker with admin on your machine has your SSH keys"* is simply true — it follows from what administrator means.

### The same file on Linux would be refused

```
Windows:  3 identities, FullControl   →  ssh works
Linux:    anything other than 600     →  ssh refuses the key
```

```
Permissions 0644 for 'id_rsa' are too open.
This private key will be ignored.
```

`chmod 600` means exactly one identity — the owner — and OpenSSH enforces it rather than warning. Linux is strictly stricter here than Windows.

### `known_hosts` is a security control, not a cache

SSH records each server's host-key fingerprint on first connection and verifies it on every subsequent one. A mismatch produces:

```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Common advice is to delete `known_hosts` and reconnect. That disables the only defence against the interception the warning describes. Host keys do legitimately change — a rebuilt server, usually — but removing the entry should be a decision, not a reflex.

---

## 3 · Findings and decisions

### Finding 1 — plaintext 2FA recovery codes in a shared folder · **RESOLVED**

GitHub two-factor recovery codes were sitting in plain text in a downloads subfolder alongside course material, unchanged since 21 July 2026. Those codes bypass 2FA by design, and the account now carries a public repository.

**Both controls failed at once.** Wrong location — a folder liable to be zipped, shared or synced without anyone auditing its contents. Wrong protection — plain text, readable by any process running as me.

One control did hold: the repository `.gitignore` contains `*recovery-codes*`, so the file could never have been committed. **But `.gitignore` protects the repository, not the disk** — which is precisely why location and protection are counted separately.

**Resolved 17 November 2026:** codes moved to a password manager, file deleted, Recycle Bin emptied. Verified absent, along with a scan confirming no other loose `*.pem`, `id_rsa*` or backup-code files in the same tree.

### Finding 2 — SSH private key has no passphrase · **ACCEPTED RISK**

```powershell
ssh-keygen -y -f "$env:USERPROFILE\.ssh\id_ed25519"
```

Returned the public key without prompting, confirming the private key is not encrypted at rest.

**Implication:** the 419-byte file is sufficient on its own to authenticate as me to GitHub. The realistic threat is not device theft but any code running under my own account — a malicious package, a browser or editor extension, an installer — reading one small file from a predictable path.

Location control exists (the ACL above). **Protection control does not.** Same shape as Finding 1, in a different form.

**Decision, 17 November 2026: accepted, not remediated.** A passphrase can be added to the existing key with `ssh-keygen -p` without regenerating it or updating GitHub, and that remains available at any time.

**Compensating control:** with no passphrase the key file *is* the credential, so the posture shifts from prevention to rapid revocation. Deleting the public key under **GitHub → Settings → SSH and GPG keys** invalidates it immediately. That path is noted deliberately in advance of needing it.

*Recorded rather than left implicit. An undocumented gap is one nobody owns; a documented one is a decision with a date against it, and can be revisited.*

---

## Summary

1. Windows has multiple filesystem roots; Linux has one, and extra disks are **mounted** into it.
2. Relative paths resolve against the **working directory** — a single piece of per-process state.
3. On a path error, **read the resolved path in the message**, not the one you typed.
4. `Cannot find path` means location. `Access is denied` means permissions. Different halves.
5. Scripts should anchor to themselves (`cd "$(dirname "$0")"`) rather than assume an invocation directory.
6. A 419-byte file controls repository access; an SSH key's size is no guide to its importance.
7. Any local administrator can read a user's private keys. This is definitional, not a misconfiguration.
8. Linux enforces `600` on private keys; Windows ACLs are more permissive.
9. `known_hosts` is interception defence. Deleting it to silence a warning removes the protection.
10. **Location and protection are independent controls, and two findings on this machine failed one each.**
