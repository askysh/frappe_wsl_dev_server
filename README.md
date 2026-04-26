# Frappe with WSL on Windows

Scripts for setting up a local Frappe / ERPNext development bench on Windows
via WSL2 + Ubuntu 22.04.

This is the Windows / WSL counterpart to
[askysh/frappe_mac_local_install](https://github.com/askysh/frappe_mac_local_install).
The two repos do the same thing on different platforms; conventions are kept
similar where they reasonably can be, but the platform-specific bootstrap
layers differ enough that they live separately.

## What You Get

- A fully provisioned WSL2 Ubuntu 22.04 distribution
- Frappe v15 system dependencies installed inside Ubuntu (build chain, Python
  toolchain, MariaDB 10.6, Redis 6, wkhtmltopdf with patched Qt, X11 stub libs)
- MariaDB hardened (no anonymous users, no test database, no remote root, root
  password set) and configured for `utf8mb4` / `utf8mb4_unicode_ci`
- A `--validate` mode that runs 25 health checks against the installed system
- Idempotent re-runs: every step checks state before acting
- The actual Frappe bench install (script `02-...`) is **not yet shipped** —
  see Roadmap below

## Requirements

- Windows 10 build 19041+ or Windows 11 (any edition; Home is fine)
- Administrator access on the Windows account
- Virtualization enabled in BIOS (the script checks the OS build but not BIOS;
  if `wsl --install` later complains about virtualization, that is the cause)
- Internet access for the first install
- About 5 GB of free disk for WSL + Ubuntu + Frappe deps

Not required: Windows Pro, Hyper-V toggle, Docker, or a separate Linux box.

## Quick Start

Clone this repo onto the Windows side:

```powershell
git clone https://github.com/askysh/Frappe-w-WSL-on-Windows.git
cd "Frappe-w-WSL-on-Windows"
```

Open **Windows PowerShell as Administrator**, navigate to the cloned folder,
and run the WSL setup script:

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\00-windows-setup-wsl.ps1
```

What it does:

1. Enables the Windows features `Microsoft-Windows-Subsystem-Linux` and
   `VirtualMachinePlatform`.
2. **Reboots** if those features were just enabled (you re-run the script
   after the reboot to continue).
3. Installs the WSL runtime and sets WSL2 as the default version.
4. Installs the `Ubuntu-22.04` distribution.
5. Launches a new Ubuntu window for first-time setup. Create your UNIX
   username and password when prompted.

When the script finishes, switch to the Ubuntu shell and run the system
dependency install:

```bash
cd ~
bash /mnt/c/path/to/Frappe-w-WSL-on-Windows/01-wsl-system-deps.sh
```

`mysql_secure_installation` will prompt interactively the first time. The
script prints the recommended answers above the prompt. The root password
you set is the one you will need later for `bench new-site`.

When script `01` ends with `All checks passed`, the system is ready for the
Frappe install (script `02`, future).

## Validation

To re-run only the health checks without installing anything:

```bash
bash 01-wsl-system-deps.sh --validate
```

This runs 25 checks (packages, services, listening ports, `wkhtmltopdf`
patched-Qt build, MariaDB charset, MariaDB hardening). Exits 0 on success,
1 on any failure with a `[FAIL]` line per failed check.

## Re-running Safely

Both scripts are designed to be re-run.

- `00-windows-setup-wsl.ps1` — every step probes Windows / WSL state and
  reports `[OK] already installed/enabled` for steps that are done.
- `01-wsl-system-deps.sh` — `apt install` is naturally idempotent; the
  MariaDB charset config is compared by checksum and only rewritten if it
  differs; `mysql_secure_installation` is launched only if the database
  has not already been hardened; `wkhtmltopdf` install is skipped if a
  patched-Qt build is already on `PATH`.

## WSL-Specific Notes

- **Always work under the Linux home directory** (`~/`, i.e.
  `/home/<username>/`). Files under `/mnt/c/...` traverse a 9P bridge
  into NTFS and are 10–50x slower for filesystem-heavy workloads like
  `git clone`, `bench init`, `npm install`.
- **systemd is on by default** in modern WSL + Ubuntu 22.04 — the distro
  provisioning step writes `/etc/wsl.conf` with `[boot] systemd=true`
  automatically. `mariadb` and `redis-server` start on first boot of
  the distro. No manual `service ... start` needed.
- **Services bind to loopback only** out of the box (MariaDB on
  `127.0.0.1:3306`, Redis on `127.0.0.1:6379`). The Windows host cannot
  reach them by default. If you ever need to expose them to Windows,
  edit the `bind-address` in `/etc/mysql/mariadb.conf.d/50-server.cnf`
  and the `bind` line in `/etc/redis/redis.conf` — and understand the
  security implications first.
- **WSL VM is reaped when idle.** Long shell idleness or all WSL windows
  closed will shut down the VM; the next `wsl` command brings it back
  up and re-runs `systemd` boot. Service start times in `systemctl status`
  resetting unexpectedly is normal.

## Files

```text
00-windows-setup-wsl.ps1     # run on Windows in admin PowerShell
01-wsl-system-deps.sh        # run inside Ubuntu (regular user with sudo)
.gitattributes               # locks bash scripts to LF, ps1 to CRLF
README.md
LICENSE
```

## Roadmap

- `02-wsl-frappe-install.sh` — install Node via nvm, yarn, `frappe-bench`
  via pipx, run `bench init`, create a default site, install ERPNext.
  Equivalent to the Mac repo's `01-install-bench-and-site.sh`.
- App bundles (`minimal` / `common` / `extended`) and profile flags
  (`v15-lts` / `v16-lts`) once `02` is in.
- Test harness under `tests/` with mocked `git` and `bench`.

## Important Notes

- Do not run the bench under `/mnt/c/...`. The 9P bridge will make every
  bench operation painfully slow and some will fail with permission or
  inotify errors.
- Do not install `wkhtmltopdf` from `apt` (`apt install wkhtmltopdf`) —
  that build is missing the patched Qt fork and will crash on real Frappe
  templates. Use the upstream `.deb` that script `01` installs.
- Do not switch MariaDB root to unix_socket-only auth during
  `mysql_secure_installation`. Frappe's `bench` connects with
  username/password and needs password auth available.
- Do not skip the reboot prompted by `00-windows-setup-wsl.ps1` after
  enabling WSL features. The runtime install will fail without it.

## License

MIT — see `LICENSE`.
