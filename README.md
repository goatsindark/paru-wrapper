# Paru Safety Wrapper

A Bash wrapper that makes AUR access a deliberate action on Arch Linux and Arch-based systems.

It blocks direct execution of `/usr/bin/paru`, installs a user-owned wrapper at `~/.local/bin/paru`, and creates a root-facing symlink at `/usr/local/sbin/paru`.

AUR commands require `sudo` and the wrapper-only `--override` flag:

```text
sudo /usr/local/sbin/paru <arguments> --override
```

For each approved command, the wrapper temporarily unlocks the real `paru` binary, runs it as the invoking user, then locks it again.

## Why this exists

AUR packages are community-maintained build recipes. Installing one deserves more attention than installing a package from the official repositories.

The wrapper adds a few deliberate steps:

- ordinary `paru` commands are blocked
- broad update commands are rejected
- overrides require `sudo`
- `/usr/bin/paru` is unlocked for one command at a time
- `paru` runs as the normal invoking user
- `/usr/bin/paru` is relocked after completion, failure, or interruption
- package targets are checked against the Arch unsafe-package list

The wrapper does not inspect PKGBUILDs or determine whether a package is safe. Review AUR packages before installing them.

## Requirements

- Arch Linux, CachyOS, or another Arch-based distribution
- Bash
- `paru` installed at `/usr/bin/paru`
- `sudo`
- `curl`
- `runuser` from `util-linux`
- `pstree` from `procps-ng`
- standard GNU tools including `install`, `chmod`, `stat`, `readlink`, `cmp`, and `getent`

## Installation

Download the script, make it executable, and run it from your normal account:

```bash
chmod +x paru
./paru
```

The installer checks the complete installation and offers install or repair mode when it finds a problem:

- `~/.local/bin/paru` is missing, outdated, or not executable
- `/usr/local/sbin/paru` is missing or points to the wrong location
- `/usr/bin/paru` is missing or does not have mode `000`

Before changing anything, the script lists what it found and asks for confirmation.

The wrapper copy in `~/.local/bin` is installed as the current user. `sudo` is requested for the system symlink and the permission change on `/usr/bin/paru`.

A successful installation looks like this:

```text
~/.local/bin/paru              user-owned wrapper, mode 755
/usr/local/sbin/paru           symlink to the user-owned wrapper
/usr/bin/paru                  real paru binary, mode 000
```

Running a newer copy of the script offers to update the installed wrapper.

## Usage

### Check for AUR updates

```bash
sudo /usr/local/sbin/paru -Qua --override
```

For `paru -Qua`, paru returns:

- exit `0` when one or more updates are found
- exit `1` when no updates are found

The wrapper treats a clean no-update result as success and exits with `0`.

### Run a command for a specific package

Pass normal paru arguments followed by `--override`:

```bash
sudo /usr/local/sbin/paru -S package-name --override
```

The wrapper removes `--override` before passing the remaining arguments to `/usr/bin/paru`.

### Commands blocked by policy

Broad update commands are rejected:

```bash
paru
paru -Syu
paru -Sua
paru --sysupgrade
```

Use `pacman` for official repository updates. Use the wrapper for specific AUR package operations.

## Unsafe-package check

When a command contains a likely package target, the wrapper checks the current Arch Linux unsafe-package list before running paru.

If the list cannot be fetched, the wrapper prints a warning and proceeds because the command already has an explicit override. You can change `UNSAFE_URL` in the script to use another source or a local list.

## Recovery

If a previous operation leaves `/usr/bin/paru` unlocked, the next wrapper run detects the incorrect mode and restores `000` before continuing.

Signal traps also relock the binary after normal completion, an error, or a common termination signal.

## Exit codes

These exit codes belong to the wrapper. When paru fails, its original status is printed and the wrapper exits with `70`.

| Code | Meaning |
|---:|---|
| `0` | Success, including `paru -Qua` with no available updates |
| `64` | Invalid usage, missing `--override`, or unavailable interactive confirmation |
| `65` | Command blocked by wrapper policy |
| `66` | Requested package appears on the unsafe-package list |
| `67` | Root privileges are required for the override |
| `68` | The invoking user could not be determined |
| `69` | `/usr/bin/paru` could not be unlocked or relocked |
| `70` | Paru returned a failure status |
| `71` | Internal wrapper or environment failure |
| `72` | Installation or repair failed |

## Updating

Run a newer copy of the script:

```bash
chmod +x paru
./paru
```

The installer compares it with `~/.local/bin/paru`. When the files differ, it offers to replace the installed copy and verifies the symlink and lock state again.

## Uninstalling

Restore direct execution of paru and remove both wrapper entry points:

```bash
sudo chmod 755 /usr/bin/paru
sudo rm -f /usr/local/sbin/paru
rm -f ~/.local/bin/paru
```

A paru package upgrade may restore the permissions on `/usr/bin/paru`. The next wrapper run will detect this and offer repair mode.

## Security notes

- The wrapper uses filesystem permissions as a behavioural lock. A privileged user can bypass or remove it.
- Keep the script writable only by the intended user. The root-facing symlink runs that user-owned file through `sudo`.
- Review changes before replacing the installed copy.
- Review PKGBUILDs and related files before installing AUR packages.
- A local allowlist or denylist can provide stronger policy control.

## License

Released under the [Zero-Clause BSD License](LICENSE). You may use, copy, modify, and distribute it without attribution requirements.
