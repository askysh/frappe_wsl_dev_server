# Frappe with WSL on Windows

Set up a local Frappe / ERPNext development server on Windows. Three scripts
walk you through it; this README walks you through THEM.

This is the Windows / WSL counterpart to
[askysh/frappe_mac_local_install](https://github.com/askysh/frappe_mac_local_install)
(for Mac).

## Who this is for

You want to try Frappe or ERPNext locally on a Windows PC, without renting a
server, paying for cloud hosting, or setting up a Linux VM by hand.

You don't need to be a Linux person — the scripts handle the Linux parts.
You DO need to be comfortable typing commands when this README tells you what
to type. If terms like "open PowerShell as Administrator", "WSL", or "bash"
are new to you, that's fine — they're explained inline.

## What you'll have at the end

- A Frappe v15 + ERPNext v15 dev server running at `http://localhost:8000`,
  open in any Windows browser.
- Logged in as `Administrator` with a password you choose.
- Everything running inside Windows Subsystem for Linux 2 (WSL2) — a real
  Ubuntu Linux environment that lives alongside Windows, started with one
  command whenever you want to use Frappe.
- About 5 GB of disk used.

## Before you start

You need:

- **Windows 10 build 19041+ or Windows 11**, any edition. Home is fine —
  you do NOT need Pro. To check: press `Win+R`, type `winver`, hit Enter,
  read the OS Build line. Must be 19041 or higher.
- **Administrator access** on your Windows account (your normal account, if
  it's the only one, almost certainly has it).
- **Virtualization enabled in BIOS** — on by default on most modern
  computers. You'll only know it's missing if Phase 1 fails with a
  virtualization error. Most users skip this entirely.
- **Internet** for downloads.
- **5 GB free disk** under your Windows user folder.

You do NOT need:

- Windows Pro, Hyper-V toggled on, or any Microsoft Store paid extras
- Docker, a separate VM, or any pre-installed Linux
- Frappe, Python, Node, MariaDB, or anything else installed beforehand —
  the scripts handle all of it

## How long this takes

About **30–60 minutes**, mostly waiting for downloads and installs. You'll
be at your computer the whole time but most of it is reading log output.

Things you can't skip:

- A **Windows reboot** in the middle of Phase 1 (the script tells you when).
- **Three passwords** to set across Phases 1–3 (next section).

## Heads-up: three passwords to write down

You'll set three different passwords at three different points during this
install. **Write each one down as you set it.** They unlock different
things and you'll be asked for them later — sometimes minutes later,
sometimes weeks later when you log back in to use Frappe.

| # | Where you set it | What it's for | When you'll need it again |
|---|---|---|---|
| 1 | Phase 1, Step 1.5 (Ubuntu's first-launch prompt) | Your **Linux user password** — for the WSL Ubuntu account | Every time the scripts run `sudo`. Also when you log back in to Ubuntu after a reboot. |
| 2 | Phase 2, Step 2.2 (MariaDB hardening, "Change the root password?") | The **MariaDB root password** — full admin access to the database | Phase 3, Step 3.2 (the `MySQL root password:` prompt). Also any time you connect to the DB as root later. |
| 3 | Phase 3, Step 3.2 (`Set Administrator password:` prompt) | Your **ERPNext Administrator password** — login to the web UI | Every time you log in at `http://localhost:8000` as `Administrator`. |

These are three SEPARATE accounts. Do not reuse the same password across
all three — when something rejects you later, you want to know exactly
which password is wrong without trying all three.

If you forget any of them later:

- **#1 (Linux user):** reset via `sudo passwd <username>` from a root WSL
  shell (`wsl -u root` in PowerShell)
- **#2 (MariaDB root):** reset via MariaDB's password-reset procedure from
  inside WSL — workable but annoying
- **#3 (ERPNext Administrator):** reset via
  `bench --site wsldev set-admin-password <new-password>` from inside
  `~/frappe-bench`

## The three terminals you'll use

You'll bounce between three different terminal windows during install. The
single most common stumble is typing the right command in the wrong window.

| Terminal | Used in | How to open |
|---|---|---|
| **PowerShell as Administrator** (Windows side) | Phase 1 | Press `Win+X`, click "Terminal (Admin)" or "Windows PowerShell (Admin)". Click Yes on the UAC prompt. The window title contains "Administrator". |
| **Ubuntu shell** (Linux, inside WSL) | Phases 2 and 3 | Start menu → search "Ubuntu" → click result. OR run `wsl -d Ubuntu-22.04` from any PowerShell. Prompt looks like `username@hostname:~$`. |
| **Web browser** (Edge, Chrome, Firefox) | Phase 4 | However you normally open one. |

When this README says "in PowerShell" or "in Ubuntu", that's which window
to use.

---

## Phase 1 — Set up WSL (Windows side)

This phase enables Windows features, installs WSL2, and installs Ubuntu
22.04. There's one Windows reboot in the middle.

### 1.1 — Get the scripts onto your computer

Open **regular PowerShell** (NOT admin yet — admin comes in 1.2). Press
`Win+R`, type `powershell`, hit Enter.

Run:

```powershell
cd $HOME
git clone https://github.com/askysh/frappe_wsl_dev_server.git
cd frappe_wsl_dev_server
```

If `git clone` fails with "git is not recognized", install Git for Windows
from <https://git-scm.com/download/win>, accept all defaults, then retry.

This downloads the repo to `C:\Users\<you>\frappe_wsl_dev_server\`.

### 1.2 — Open PowerShell as Administrator

Press `Win+X`, click "Terminal (Admin)" (Windows 11) or "Windows PowerShell
(Admin)" (Windows 10). Click Yes on the UAC prompt.

Navigate back to the cloned folder:

```powershell
cd $HOME\frappe_wsl_dev_server
```

### 1.3 — Run the WSL setup script

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\00-windows-setup-wsl.ps1
```

(`-ExecutionPolicy Bypass` is needed because Windows blocks unsigned scripts
by default. The flag only affects this one run, not your system policy.)

What you'll see:

- A series of `[OK]` lines for the pre-checks
- The script enables two Windows features
- A prompt: `A Windows reboot is required... Reboot now? [y/N]`

**Type `y` and press Enter.** Windows reboots in 5 seconds.

### 1.4 — After reboot, run the same command again

After Windows comes back up, open **PowerShell as Administrator** again
(Step 1.2's instructions), `cd` back to the repo folder, and re-run:

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\00-windows-setup-wsl.ps1
```

This time the script picks up where it left off — installs the WSL runtime,
sets WSL2 as default, and installs Ubuntu 22.04. Steps already done show
`[OK]   already installed/enabled`.

When Ubuntu finishes installing, **a new Ubuntu window opens automatically**
asking you to create a Linux user.

### 1.5 — Create your Linux user

In the Ubuntu window:

- `Enter new UNIX username:` — pick a lowercase name. e.g. your initials,
  or `frappedev`. This is your Linux login, separate from your Windows
  account.
- `New password:` then `Retype new password:` — pick a password. **Write
  it down.** As you type, no characters appear on screen. That's normal —
  Linux never echoes passwords. Just type and press Enter.

You'll land at `username@hostname:~$`. Phase 1 is done.

You can close the Admin PowerShell window now. The rest of the install
happens in the Ubuntu window.

---

## Phase 2 — Install system dependencies (Ubuntu side)

This installs Linux packages Frappe needs: build tools, Python, MariaDB
(database), Redis (cache), and a special PDF rendering library.

### 2.1 — Get the scripts on the Linux side

The repo you cloned in Phase 1 is on Windows (`C:\Users\<you>\...`). For
Phases 2 and 3, life is easier if you also clone it inside Ubuntu so the
bash commands have short paths.

In **Ubuntu**:

```bash
cd ~
git clone https://github.com/askysh/frappe_wsl_dev_server.git
cd frappe_wsl_dev_server
```

That `~` symbol means your Linux home directory. Always work under `~/` in
WSL — files under `/mnt/c/...` (which is how WSL sees your Windows drive)
are 10–50x slower for filesystem-heavy operations. The Frappe install will
break if you put it under `/mnt/c/`.

### 2.2 — Run script 01

```bash
bash 01-wsl-system-deps.sh
```

You'll be prompted for your **Linux password** (the one from Step 1.5) so
the script can run `sudo`. (`sudo` is Linux's "do this as administrator" —
required for installing system packages.)

The script then:

1. Updates Ubuntu's package list, upgrades to latest patches (~2 min)
2. Installs system packages: build chain, Python 3.10, MariaDB, Redis, etc.
   (~3-5 min)
3. Configures MariaDB for Frappe's character-set requirements
4. Runs MariaDB's hardening wizard — **interactive, pay attention**
5. Downloads and installs `wkhtmltopdf` with patched Qt (PDF generator)
6. Runs 25 health checks and reports

When the hardening wizard runs, the script prints recommended answers
above each prompt. The right answers are:

| Prompt | Answer |
|---|---|
| `Enter current password for root` | Just press Enter (no password yet) |
| `Switch to unix_socket authentication` | `n` |
| `Change the root password?` | `y`, then **pick a strong password — write it down.** You'll need it in Phase 3. This is the **MariaDB root password.** |
| `Remove anonymous users?` | `y` |
| `Disallow root login remotely?` | `y` |
| `Remove test database and access to it?` | `y` |
| `Reload privilege tables now?` | `y` |

### 2.3 — Confirm Phase 2 succeeded

The script ends with either:

- `==> All checks passed. System is ready for 02-wsl-frappe-install.sh.`
  (good — proceed to Phase 3)
- `==> N check(s) failed.` (look at the `[FAIL]` lines; most failures are
  network blips and just re-running the script fixes them)

You can re-run validation anytime without re-installing:

```bash
bash 01-wsl-system-deps.sh --validate
```

---

## Phase 3 — Install Frappe and ERPNext (Ubuntu side)

This installs Node.js, yarn, frappe-bench (Frappe's CLI), the Frappe v15
codebase, the ERPNext v15 codebase, and creates your first site. ~10–15
minutes for first run.

### 3.1 — Run script 02

In Ubuntu, from the same `~/frappe_wsl_dev_server` directory:

```bash
bash 02-wsl-frappe-install.sh
```

The script:

1. Installs nvm (Node Version Manager) and Node 20
2. Pins yarn to version 1.22.22 (Frappe v15 needs yarn Classic, not 4.x)
3. Installs pipx, then uv, then frappe-bench
4. Patches `~/.profile` so future commands find Node correctly (a
   non-obvious requirement — see "Common stumbles" below if you care)
5. Runs `bench init` — clones Frappe v15, builds frontend (~5 min)
6. Runs `bench get-app erpnext` — clones ERPNext, builds frontend (~5 min)
7. Runs `bench new-site wsldev` — **two interactive password prompts**
   (see 3.2 below)
8. Runs `bench install-app erpnext` to install ERPNext into the new site
   (~2 min)
9. Sets `wsldev` as the default site
10. Runs 15 health checks and reports

### 3.2 — The two passwords during `bench new-site`

You'll be prompted twice. **They are different passwords.** Both go on
your written list.

| Prompt | What to type |
|---|---|
| `MySQL root password:` | The MariaDB root password from Phase 2.2. (Yes, "MySQL" — Frappe was MySQL-only at one point and the prompt label never updated. MariaDB is what's actually running.) |
| `Set Administrator password:` then `Re-enter Administrator password:` | A NEW password (different from the one above). This is what you log in to ERPNext with. |

**These are not the same as your Linux password.** Three passwords total
across the install. Keep them straight.

### 3.3 — Skip the prompts (optional)

If you'd rather pass passwords as environment variables to avoid prompting:

```bash
MARIADB_ROOT_PASSWORD='your-mariadb-root-pw' \
ADMIN_PASSWORD='your-erpnext-admin-pw' \
bash 02-wsl-frappe-install.sh
```

### 3.4 — Confirm Phase 3 succeeded

Ends with `==> All checks passed. Frappe v15 + ERPNext are installed.` —
good. Proceed to Phase 4.

You can re-validate anytime:

```bash
bash 02-wsl-frappe-install.sh --validate
```

---

## Phase 4 — Start the server and log in

### 4.1 — Open a fresh Ubuntu shell

If your Ubuntu shell was open before Phase 3 ran, the changes Phase 3 made
to your shell config aren't loaded yet. Either:

- **Close that Ubuntu window and open a new one** (recommended), or
- Run `source ~/.profile && source ~/.bashrc` in the existing shell

If you're not sure, just close and reopen.

### 4.2 — Start the dev server

```bash
cd ~/frappe-bench
bench start
```

This boots all the Frappe processes — web server, two Redis instances,
background worker, scheduler, asset watcher — and streams their output
to your terminal. After ~5 seconds you'll see lines like:

```
17:37:22 web.1 | * Running on http://127.0.0.1:8000
```

That means Frappe is ready.

**Leave this terminal window running.** Do not close it. You can minimize
it. The processes stop the moment you close the window or press Ctrl+C.

### 4.3 — Open ERPNext in your browser

Open `http://localhost:8000` in any Windows browser (Edge, Chrome, Firefox
— all work; WSL2 forwards localhost to Linux automatically, no host file
edits needed).

You'll see a Frappe login page.

**Login:**

- Username: `Administrator`
- Password: the **Administrator password** from Phase 3.2 (NOT MariaDB,
  NOT Linux)

### 4.4 — Walk through the ERPNext setup wizard

First login shows a setup wizard. Answer the prompts:

- Country (sets currency, fiscal year, default chart of accounts)
- Company name
- Currency
- A user record for yourself

When the wizard finishes, you're at the ERPNext desk home — modules list
on the left, onboarding cards in the middle. **You're done.**

---

## Daily use after install

To start Frappe whenever you want to use it:

1. Open the Ubuntu shell (Start menu → "Ubuntu", or run `wsl` in PowerShell)
2. `cd ~/frappe-bench && bench start`
3. Open `http://localhost:8000` in a browser
4. Log in as Administrator

To stop: in the Ubuntu shell, press `Ctrl+C`.

To remove ERPNext entirely: see "Wiping it" below.

---

## Common stumbles

**The reboot prompt asks Y/N but I'm confused which one to pick.**
Phase 1 needs you to reboot Windows. Type `y` and Enter. Windows reboots
after 5 seconds. After it comes back, repeat Step 1.4.

**I rebooted, the script ran the second time, but no Ubuntu window opened.**
Open Ubuntu manually: Start menu → "Ubuntu", or run `wsl -d Ubuntu-22.04`
in any PowerShell. The user-creation prompt shows up the first time you
launch.

**`bench: command not found` after Phase 3.**
Your shell was open before Phase 3 ran. Phase 3 modified `~/.profile` to
add `bench` to your PATH, but already-open shells don't see those changes.
Close the Ubuntu window and open a new one. OR run
`source ~/.profile && source ~/.bashrc` in the existing shell.

**"The page can't be reached" / "ERR_CONNECTION_REFUSED" on `localhost:8000`.**
Check that `bench start` is still running in the Ubuntu shell. If you see
the prompt again (`username@hostname:~$`), the server is stopped — re-run
`bench start`.

**"MySQL root password" prompt rejects what I type.**
Three passwords are in play. The prompt is asking for the **MariaDB root
password from Phase 2.2 step 4** — not your Linux password and not the
Administrator password.

**The Ubuntu password prompt shows nothing as I type.**
Linux does not echo passwords for security. Type your password (you won't
see asterisks or dots) and press Enter. This applies to your initial Linux
user creation, every `sudo` call, and the MariaDB root prompt.

**"WSL 1 is not supported" warning during nvm install.**
False positive. nvm 0.40.4's WSL detection is buggy on WSL2; the install
actually completes fine. The warning is harmless.

**I want to start over with a clean slate.**
See "Wiping it" below. (You can blow away the whole WSL distribution and
re-run Phase 1 from scratch in ~10 minutes.)

**Anything else.**
- `bash 01-wsl-system-deps.sh --validate` to check Phase 2 state
- `bash 02-wsl-frappe-install.sh --validate` to check Phase 3 state
- Both print every check with `[OK]` or `[FAIL]`. Read the failures.

---

## Customizing

Configurable via environment variables when running script 02:

```bash
# Site name (default: wsldev)
SITE_NAME=mysite bash 02-wsl-frappe-install.sh

# Frappe / ERPNext branches (default: version-15)
FRAPPE_BRANCH=develop ERPNEXT_BRANCH=develop bash 02-wsl-frappe-install.sh

# Node major version (default: 20). Frappe v15 needs Node 18 or higher.
NODE_VERSION=22 bash 02-wsl-frappe-install.sh

# Skip both password prompts
MARIADB_ROOT_PASSWORD='...' ADMIN_PASSWORD='...' bash 02-wsl-frappe-install.sh
```

Don't change `YARN_VERSION`. Frappe v15 needs yarn Classic (1.22.x); yarn
4 (Berry) breaks `bench build`.

---

## Wiping it

To remove everything and start fresh:

```powershell
# in PowerShell (any kind)
wsl --unregister Ubuntu-22.04
```

That deletes the entire Ubuntu install and all Frappe data. It does NOT
remove WSL itself or the Windows features. To re-install Frappe, just run
script 00 again — it'll detect that WSL is already there and skip straight
to installing Ubuntu.

To also remove the WSL runtime: `wsl --uninstall` in admin PowerShell.

---

## For AI coding agents

If you're an AI coding agent helping a user install Frappe locally on
Windows: this README is structured to be agent-followable. Each phase has
explicit "Step N.X" subsections that, in sequence, produce a working
install. Defer to the script's own `[OK]` / `[FAIL]` output for diagnostics.

Key things to call out for the user that the README already covers but
that you should reinforce in your own voice:

- They will need admin PowerShell for Phase 1 — explicitly walk them
  through Win+X
- They'll be prompted three times for three different passwords across
  Phases 1-3; tell them to write each one down as they set it
- Windows must reboot mid-Phase-1; explicitly tell them to re-run the same
  script after reboot
- The Ubuntu shell that ran Phase 3 should be closed and reopened before
  Phase 4
- `bench start` (Phase 4) runs in the foreground; the user must leave that
  terminal open while using Frappe

---

## Files

```text
00-windows-setup-wsl.ps1     # Phase 1: Windows side, admin PowerShell
01-wsl-system-deps.sh        # Phase 2: Ubuntu side, regular user
02-wsl-frappe-install.sh     # Phase 3: Ubuntu side, regular user
.gitattributes               # locks line endings (sh=LF, ps1=CRLF)
README.md                     # this file
LICENSE                       # MIT
```

---

## Roadmap

- App bundles (`minimal` / `common` / `extended`) — install HRMS, Payments,
  CRM, Helpdesk, Insights alongside ERPNext, matching the Mac repo's
  pattern
- Profile flags (`v15-lts` / `v16-lts`) for switching Frappe major versions
- Test harness with mocked git + bench so the scripts can be CI-tested
- A `99-wsl-uninstall.sh` rollback for clean experimentation

---

## Important notes

- **Always work under `~/` inside WSL, never under `/mnt/c/...`.** The 9P
  bridge to NTFS is 10–50x slower and breaks some Frappe operations.
- **Don't `apt install wkhtmltopdf`** — the apt version isn't built with
  patched Qt and crashes on real Frappe templates. Script 01 installs the
  right one.
- **Don't switch MariaDB root to unix_socket-only** during
  `mysql_secure_installation` (Phase 2). Frappe needs password auth.
- **Don't skip the reboot** prompted by Phase 1. The WSL runtime install
  will fail without it.

## License

MIT — see [LICENSE](LICENSE).
