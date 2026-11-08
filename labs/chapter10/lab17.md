# 17 — Software Updates and Patch Auditing — Lab

## Objectives

By the end of this lab, you will be able to:

1. Inventory the installed software on a Linux host and measure the pending security updates, using only read-only commands.
2. Reconstruct the update history of a system from package manager logs and judge how current the system is.
3. Connect the lecture's update types (patch, hotfix, service pack) and the auto-update mechanism to the configuration of a real machine.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required: every command in this lab reads system state without changing anything.
- The `dpkg` and `apt` package tools, present on every Debian or Ubuntu system.
- Internet access from the VM for the changelog lookup in Part 5.
- Chapter 10 lecture, Section 6: software patches, hotfixes, service packs, patch management, and the auto-update option.

## Before you start

### Background

Software updates remove security vulnerabilities, fix errors, and improve performance. The lecture divides updates into three types: a **patch** fixes or improves a program, a **hotfix** addresses one specific recently found problem, and a **service pack** bundles many updates into one installation. **Patch management** is the discipline of distributing and applying updates on time, and an **auto-update** mechanism downloads and installs updates without user intervention.

On a Debian-family Linux system, the package database records every installed program, the `apt` tools compare installed versions against the distribution's current versions, and log files keep the full history. All of this state is readable by any user, so a complete update audit needs no administrator rights. Applying the updates is the step that requires root, which is why the lecture pairs patching with centralized tools such as unified endpoint management.

| Concept from the lecture | Where you observe it in this lab |
| --- | --- |
| Patch, hotfix, service pack | Part 5: classifying a real package changelog entry by update type |
| Patch management | Part 2: the upgradable package list is the pending work of patch management |
| Auto-update | Part 4: the unattended-upgrades configuration controls automatic security updates |
| Outdated software risk | Part 6: a running kernel older than the installed kernel signals a pending reboot |
| Centralized management (UEM) | Part 7: you identify who applies updates on the shared VM and with what tool class |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab17`
  - Go to folder: `cd ~/comp301/lab17`

## Part 1 — Setup and software inventory

```bash
cd ~/comp301/lab17
dpkg -l | grep -c '^ii'
```

The count is the number of installed packages on the VM. Every package is a potential source of vulnerabilities, which is why the lecture lists removal of unnecessary software as an attack-surface reduction. Record the count.

Peek at a few entries to see the format:

```bash
dpkg -l | grep '^ii' | head -5
```

The columns show the package name, the installed version, and the architecture. The version strings are the baseline for the comparison in Part 2.

## Part 2 — Pending updates

List every package with an update waiting in the distribution:

```bash
apt list --upgradable 2>/dev/null | grep -c /
apt list --upgradable 2>/dev/null | head -10
```

Each line names a package, the candidate version, and the installed version. The count is the system's patch backlog. Record both the count and the name of the first listed package.

The list reflects the last time the package index refreshed. Check the refresh stamp:

```bash
ls -l /var/lib/apt/periodic/update-success-stamp 2>/dev/null || echo "no stamp file"
```

A stamp older than a day means the backlog may be larger than reported. A missing stamp means the index has never refreshed on schedule, which the Part 4 configuration explains.

> **\[NOTE\]**
>
> If `apt list --upgradable` prints nothing, the system is fully current as of the stamp date. Record `pending-updates: 0` and pick any installed package for Part 5, for example `coreutils`.

## Part 3 — Update history

The package manager writes a log of every install, upgrade, and removal. Read the recent entries:

```bash
grep "Commandline" /var/log/apt/history.log | tail -5
grep "Start-Date" /var/log/apt/history.log | tail -3
```

The first command shows the last five operations an administrator ran. The second shows the dates of the last three log entries, which tell you how recently the system received attention. Older months live in rotated, compressed logs:

```bash
ls -lh /var/log/apt/
zgrep "Upgrade:" /var/log/apt/history.log.1.gz 2>/dev/null | tail -2
```

Record the date of the most recent log entry.

> **\[NOTE\]**
>
> For example, you can use `echo` with command substitution to write a field directly to `submission.txt` as follows:
>
> ```bash
> echo "pending-updates: $(apt list --upgradable 2>/dev/null | grep -c /)" >> submission.txt
> echo "last-log-entry: $(grep 'Start-Date' /var/log/apt/history.log | tail -1 | awk '{print $2}')" >> submission.txt
> ```
>
> Write the free-text analysis fields with plain `echo`:
>
> ```bash
> echo "analysis-1: A point release bundles many patches, so it acts as the service pack equivalent." >> submission.txt
> ```
>
> The `>>` operator appends, so run each command once and check your work with `cat submission.txt` as you go.

## Part 4 — The auto-update mechanism

The lecture describes auto-update as the option downloading and installing updates without user intervention. Ubuntu implements auto-update with the unattended-upgrades package, configured in plain text files any user can read:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades 2>/dev/null || echo "unattended-upgrades not configured"
```

Two lines matter. `Update-Package-Lists "1"` refreshes the index on a schedule, and `Unattended-Upgrade "1"` installs security updates automatically. A value of `"0"` disables the feature. Check the timers driving the schedule:

```bash
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer --no-pager 2>/dev/null | head -4
```

The `NEXT` column shows when the next automatic run happens. Record whether unattended-upgrades is enabled and when the next timer fires.

## Part 5 — Classifying a real update

Pick one package from the Part 2 backlog and read the changelog of the pending version:

```bash
PKG=$(apt list --upgradable 2>/dev/null | sed -n 2p | cut -d/ -f1)
echo $PKG
apt changelog $PKG 2>/dev/null | head -30
```

If the backlog was empty, use `PKG=coreutils`. The changelog command downloads the entry over HTTPS. If the campus network blocks the download and the command prints nothing, fall back to the local package description and the version strings from Part 2:

```bash
apt show $PKG | head -15
```

The changelog entries describe each fix. Classify the entry:

- A security fix for a specific CVE behaves like a **hotfix**: one specific problem, released quickly.
- A routine version update with several fixes behaves like a **patch**.
- A distribution point release bundling many updates behaves like a **service pack**.

Record the package name and your classification with a one-line justification.

## Part 6 — Kernel posture and the pending reboot

Some updates take effect only after a restart. Compare the running kernel with the newest installed kernel:

```bash
uname -r
dpkg -l | grep linux-image | awk '{print $3}' | sort -V | tail -1
```

If the two version strings differ, the system installed a kernel update but still runs the old kernel, so the fix is dormant. Check the explicit reboot flag:

```bash
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required || echo "no reboot pending"
```

A shared server cannot reboot on demand, which matches the lecture's second condition blocking patching: some systems cannot shut down for updates. Record whether a reboot is pending.

## Part 7 — Analysis

Answer in one sentence each, from what you observed:

1. Which lecture update type does a distribution point release resemble, and why?
2. The lecture claims auto-update cuts the odds of running outdated software. Based on the Part 4 configuration, is the VM relying on auto-update or on manual administrator action?
3. You can audit every package but cannot apply a single update. Which lecture technology (Section 1) gives an administrator the ability to push updates to every endpoint from one console, and why does a shared server need such a tool?

## Assignment

Create a `submission.txt` file with the following entries:

- `installed-packages: <the count from Part 1>`
- `pending-updates: <the upgradable package count from Part 2>`
- `first-upgradable: <the first package name from Part 2, or "none">`
- `index-stamp: <the date of update-success-stamp, or "no stamp file">`
- `last-log-entry: <the date of the most recent history.log entry>`
- `auto-update-enabled: <YES or NO — Unattended-Upgrade set to "1">`
- `next-timer: <the NEXT time of apt-daily-upgrade.timer, or "not listed">`
- `changelog-package: <the package you classified in Part 5>`
- `update-classification: <hotfix, patch, or service pack, with a one-line justification>`
- `running-kernel: <the output of uname -r>`
- `newest-installed-kernel: <the newest linux-image version>`
- `reboot-pending: <YES or NO>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab17`) and name the file exactly `submission.txt`.

## Useful References and Resources

- `man apt` and `man apt-get` — the package management commands, including `apt list --upgradable` and `apt changelog`.
- `man dpkg` — the low-level package tool behind the inventory in Part 1.
- Ubuntu Security Notices (ubuntu.com/security/notices) — the public list of security updates, the Linux equivalent of a vendor hotfix advisory.
- Debian Administrator's Handbook, Chapter 6 (debian-handbook.info) — package lifecycle, changelogs, and the unattended-upgrades configuration used in Part 4.
- `man 5 apt.conf` — the configuration file format read in Part 4.

## Optional Challenges

- Run `apt-get --simulate upgrade 2>/dev/null | grep -c '^Inst'` to count how many packages a full upgrade would touch, and compare the number with the Part 2 backlog.
- Run `zgrep -h " linux-image" /var/log/apt/history.log* | tail -5` to trace the kernel update history across rotated logs, and count how many kernel updates the VM received this term.
