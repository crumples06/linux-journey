# Shell Basics

Everyday navigation, file handling and shell features. Covers the week-1 material that wasn't in the other reference files. (Permissions/`chmod`, `ls -l`, `/etc/passwd`, links: `filesystem-and-permissions.md`. Redirection details and exit status: `shell-scripting.md`.)

## Command anatomy

`command [options] [arguments]`
- **command**: what to run.
- **options**: modify its behaviour (`-l`, `--all`).
- **arguments**: what it operates on.
Options can be combined: `ls -l --all --human-readable`.

## Navigating with `cd`

| Command | Goes to |
|---|---|
| `cd -` | The previous working directory (toggles back and forth) |
| `cd /` | The root of the filesystem, above `/home` |
| `cd ~` | My home directory |

`cd /` and `cd ~` are **not** the same: `/` is the top of the whole tree, `~` is `/home/<me>`.

## Reading files

### `cat`
- `cat file` prints a file. Awkward for long files: no mouse, keyboard only.
- `cat --number file` prints with line numbers.
- Create a file by typing into it: `cat > notes.txt` (Ctrl+D to finish).
- Merge: `cat file1 file2 > file3` writes both into `file3`.
- **`>` overwrites** the target's previous contents; **`>>` appends**: `cat file1 >> file2`.

### `less`
The right tool for just *reading*. `less file` is interactive: scroll, and search for words (`/word`). Quit with `q`. Far better than `cat` for big files.

## Wildcards (globbing)

The **shell expands wildcards before the command runs**, so the command just sees a list of file names.

| Pattern | Matches |
|---|---|
| `*` | Any characters (every file name) |
| `g*` | Names starting with `g` |
| `?` | Exactly one character |
| `d??` | `d` followed by exactly two characters |
| `[abc]` | One character that is a, b or c |
| `[abc]*` | Names starting with a, b or c |
| `[[:upper:]]*` | Names starting with an uppercase letter |
| `[![:lower:]]*` | Names *not* starting with a lowercase letter (`!` negates) |

Character classes: `[:alnum:]`, `[:alpha:]`, `[:digit:]`, `[:lower:]`, `[:upper:]`. Similar in spirit to regex, but a different, simpler language.

**Surprise worth remembering:** `ls l*` listed the *contents* of directories starting with `l`, as well as matching files. The shell expands `l*` into a list of names; when `ls` is given a directory name, it lists what is inside. Use `ls -d l*` to see only the entries themselves.

## Copy, move, remove

### `cp`
- `cp file1 file2` behaves like `cat file1 > file2`.
- `cp file1 dir1` copies into the directory.
- `cp -R dir1 dir2` copies recursively. If `dir2` exists, it creates `dir2/dir1`; it does **not** just dump the contents into `dir2`.

### `mv`
- Rename: `mv file1 file2`
- Move into a directory: `mv file1 file2 dir1` (last argument is the destination)
- `mv dir1 dir2`: moves if `dir2` exists, otherwise renames.

### `rm`
- `rm file1 file2`
- `rm -r dir1 dir2` (recursive)
- No trash can. Gone is gone. Double-check wildcards before pressing Enter.

## I/O redirection and pipes

- `>` sends a command's **stdout** to a file (overwrite); `>>` appends.
- `ls -l > list.txt` makes a huge listing (like `/usr/bin`) easy to navigate.
- `|` (pipe) connects the **stdout of one command to the stdin of the next**: `ls -l | less`, a favourite for big directories without making a file.
- Pipes are what make long one-liners from the internet possible.

## Shell arithmetic

```bash
echo $((2+3+7))        # 12
echo $(((5**2)+3))     # 28
```
`$(( ))` evaluates integer arithmetic.

## Brace expansion

Generates multiple strings from a pattern:
```bash
echo F-{A,B,C}-B       # F-A-B F-B-B F-C-B
echo {1..5}            # 1 2 3 4 5
```

## Aliases

```bash
alias l='ls -l'
```
- **No spaces around `=`.** `alias l = 'ls -l'` fails, for the same reason as variable assignment.
- Typed in a terminal, it lasts for that session only. Put it in `~/.bashrc` to make it permanent.

## Shell functions

For anything bigger than an alias:
```bash
today() {
    echo -n "Today's date is "
    date +"%A, %B %-d, %Y"
}
```

## Running my own scripts as commands

- A script must be executable (`chmod`) and on the `PATH`. `echo $PATH` lists the directories the shell searches, in order.
- I keep scripts in `~/.local/bin`.
- The file name must **be** the command name: `helloworld`, not `helloworld.txt`. No extension needed.
