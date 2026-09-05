# Filesystem & Permissions

Topic-wise reference on the Linux file hierarchy, permissions, links, and related concepts.

## ls -l and permission strings

`ls -l` output breakdown (this confused me at first — too much info in one line):

- **1st character** — file type: `-` regular file, `d` directory, `l` symbolic link (symlink), `c` character device, etc.
- **Next 9 characters** — three groups of `rwx`, in order: owner, group, everyone else.
- **Owner / group names** — who owns the file and which group it belongs to.
- **Size** — in bytes by default; add `--human-readable` for proper units (KB/MB/etc).
- **Modification time**
- **Filename**

Other useful `ls` flags:
- `--all` — show hidden files (dotfiles like `.git`, `.gitignore`)
- `--classify` — appends a marker to entries: `/` for directories, `*` for executables
- Flags can be chained: `ls -l --all --human-readable`
- `ls -ld <target>` — shows the entry's own metadata rather than listing its contents (used this to spot symlinks and character devices — see below)

## chmod (permissions)

- Permissions are `rwx`, representable in binary as `111`.
- There are three permission sets (owner / group / other), same order as in `ls -l`.
- Represent all three as three digits, e.g. `755` = `111 101 101` = `rwxr-xr-x`.
```bash
chmod 755 script.sh
```

## umask

Controls the **default** permissions new files get on creation. Considered using this to avoid manually `chmod`-ing every new script, but decided against it — it's simpler and safer to just `chmod` scripts individually as needed rather than changing system-wide defaults.

## /etc/passwd — user account database

One line per user account, fields separated by `:`:
```
username:password:UID:GID:comment:home_directory:shell
```
- **password** field just holds `x` — the real password hash lives in `/etc/shadow`.
- **shell** — the user's default login shell, e.g. `/bin/bash`.

Ran `cat /etc/passwd` expecting a handful of accounts and found **49**. Turns out there are two categories:
- **Human accounts** — actual login users (on my laptop: `user` and `tanish`).
- **System accounts** — run background processes with limited privileges, not meant for human login. Kept separate from human accounts for security (limits blast radius if one is compromised).

## Filesystem Hierarchy Structure (FHS)

- **`/`** — root directory. Nothing exists above it; only root can modify all of it.
- **`/bin`** — system-wide commands needed by all users. Contains symlinks (see below); actual binaries are compiled/not human-readable as text.
- **`/boot`** — files required for booting: GRUB config, kernel files. (Found a leftover PopOS boot folder in `/boot/efi/EFI` from before switching to Ubuntu — harmless, left it alone.)
- **`/dev`** — device files, the interface between hardware and software.
  - `ls -ld hpet` → starts with `c` → **character device** — transfers data byte-by-byte as a stream (keyboard, mouse, etc).
- **`/etc`** — configuration files for system apps, users, services, tools.
- **`/home`** — one directory per non-root user (e.g. `/home/tanish`); `cd ~` goes here. Each home dir can have its own `bin`, `dev`, `etc`, etc.
- **`/media`** — mount point for removable devices (CD, USB drives).
- **`/mnt`** — mount point for external drives.
- **`/proc`** — virtual filesystem representing live kernel state.
  - Has a directory per running process, named after its PID.
  - A process can read its own info from `/proc/PID/*` with no extra permissions.

## Symbolic links (symlinks)

Discovered by noticing blue-colored entries in `/bin` under `ls`. Checking one with `ls -ld` showed a leading `l` — symlink.
- Example: `GET` is a symlink to `lwp-request`.
- A symlink is a pointer to another file/directory — it stores a **path**, not content.
- Different inode number from the target.
- Can span across different filesystems.
- Can point to a directory.
- **Breaks** if the target is deleted (becomes a "hanging link") or if the target is renamed.
- Create with: `ln -s <original> <link_name>`

## Hard links

- A hard link shares the **same inode number** as the original — both names point to the same physical data on disk.
- Example: `ln original.txt hardlinked.txt` — either filename accesses the same content.
- **Cannot** span multiple filesystems.
- **Cannot** be created for directories (avoids recursive loop issues).
- If the original file is deleted, the hard link **still works** — the physical data isn't tied to either name specifically, it's tied to the inode, which persists until *all* links to it are gone.
- Create with: `ln <original> <link_name>`

## Inodes

- An inode (index node) is a metadata structure created when a filesystem is initialized.
- Every file is identified by an inode.
- The **total number of inodes is fixed at filesystem creation** — this caps the maximum number of files/directories the filesystem can ever hold, independent of raw disk space.

**Stored attributes:**
- File size
- Permissions
- File type
- Timestamps
- Owner and group

(Notably: **not** the filename or the actual file content — that's why hard links work the way they do.)
