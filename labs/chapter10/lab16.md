# 16 — Endpoint Attack Surface and Port Enumeration with Nmap — Lab

## Objectives

By the end of this lab, you will be able to:

1. List the listening TCP and UDP ports on a Linux host without administrator rights, and explain why the process names stay hidden from a non-privileged account.
2. Scan a host with `nmap`, compare the scanner's view with the host's own socket list, and match each open port to the protocol table from the lecture.
3. Show how the attack surface differs between the loopback interface and the external interface, and apply the lecture's close-or-filter rule to a real port inventory.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required: every command in this lab runs as a regular user.
- The `nmap` command-line tool. Part 1 verifies the tool, and every nmap part includes an `ss`-based fallback if the package is missing.
- Internet access from the VM for the optional external scan in Part 5.
- Chapter 10 lecture, Section 5: operating system ports, well-known and registered port ranges, open and closed ports, port enumeration, and the close-or-filter rule for shrinking the attack surface.

## Before you start

### Background

Every network service on an endpoint sends and receives data through a **port**, and each **open port** exposes the vulnerabilities of the service behind the port. Attackers run **port enumeration** to find those open doors before choosing an exploit, and defenders run the same scans to find ports to close. The lecture's defense has two moves: close or filter unused ports with a firewall, and disable unnecessary services.

Two views of the same host exist. The host itself knows every listening socket, which `ss` reveals. A remote scanner sees only the ports the network path allows through, which `nmap` reveals. Comparing the two views shows exactly how much a firewall or a loopback-only binding hides.

| Concept from the lecture | Where you observe it in this lab |
| --- | --- |
| Open and closed ports | Part 3: `nmap` reports open ports on 127.0.0.1 and marks every other port as closed |
| Port enumeration with Nmap | Part 3: `nmap -sV` identifies the service and version behind each open port |
| Well-known port assignments | Part 3: matching scanned ports against the lecture's secure protocol table |
| Attack surface per interface | Part 4: the loopback scan and the external-interface scan return different port lists |
| Close-or-filter rule | Part 6: you choose which observed port to close and name the mechanism |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab16`
  - Go to folder: `cd ~/comp301/lab16`

## Part 1 — Setup and tool check

```bash
cd ~/comp301/lab16
nmap --version
```

If the command prints a version, continue. If the shell reports `command not found`, the package is missing and you cannot install it without administrator rights. In that case, use the fallback command given in each part and record `nmap-missing: YES` in your submission.

> **\[NOTE\]**
>
> `nmap` runs several scan types. **TCP SYN scan** (`-sS`), the default for privileged users, needs raw socket access and fails without root. As a regular user, `nmap` automatically falls back to the **TCP connect scan** (`-sT`), which completes a full TCP handshake through the operating system. The results are the same for this lab; the scan is only slower and more visible in logs. UDP scanning (`-sU`) and OS detection (`-O`) need root and will refuse to run.

## Part 2 — The host's own view with ss

List every listening socket on the VM:

```bash
ss -tuln
```

The flags select TCP (`-t`), UDP (`-u`), listening sockets (`-l`), and numeric addresses (`-n`). Read two columns: `Local Address:Port` shows the bind address and port, and the missing process column is the point. Add the process flag and compare:

```bash
ss -tulnp
```

Process names appear only for sockets your own user owns. Every system service stays anonymous, because reading another user's process information needs administrator rights. An unprivileged attacker on the same machine faces exactly this blind spot.

Count the listening sockets and save the number:

```bash
ss -tuln | tail -n +2 | wc -l
```

Identify any entry bound to `127.0.0.53:53`. That is the systemd-resolved DNS stub, the same component the DNS lab worked around, and the entry matters again in Part 4.

## Part 3 — Scanning the loopback with nmap

Scan the loopback address, the view of the machine from itself:

```bash
nmap 127.0.0.1
```

Fallback if nmap is missing: reuse the `ss -tuln` output and treat each `127.0.0.1` or `0.0.0.0` entry as open.

The report lists each open port with a service name. `nmap` resolves the names from its own services database, which follows the IANA assignments from the lecture. Now add version detection:

```bash
nmap -sV 127.0.0.1
```

The `VERSION` column shows the actual software answering on each port, such as `OpenSSH 9.6p1`. Version detection sends probes and reads the banners, the same reconnaissance step an attacker runs before selecting an exploit for a known vulnerability.

Match every open port against the secure protocol table in the lecture. An SSH listener on TCP 22 confirms the remote access service you used to connect. Record the list of open ports and the SSH version string.

> **\[NOTE\]**
>
> For example, you can use `echo` with command substitution to write a field directly to `submission.txt` as follows:
>
> ```bash
> echo "loopback-open-ports: $(nmap 127.0.0.1 | grep open | awk '{print $1}' | paste -sd, -)" >> submission.txt
> echo "service-ssh-version: $(nmap -sV 127.0.0.1 | grep '22/tcp' | awk '{print $4, $5}')" >> submission.txt
> ```
>
> Write the free-text analysis fields with plain `echo`:
>
> ```bash
> echo "analysis-1: Process names stay hidden because reading another user's processes needs root." >> submission.txt
> ```
>
> The `>>` operator appends, so run each command once and check your work with `cat submission.txt` as you go.

## Part 4 — The external interface view

A service bound to `127.0.0.1` answers only local connections. A service bound to `0.0.0.0` answers on every interface, including the network-facing one. Find the VM's external address and scan it:

```bash
hostname -I
nmap <vm-ip>
```

Replace `<vm-ip>` with the first address from `hostname -I`. Fallback if nmap is missing: compare the `ss` entries bound to `0.0.0.0` against the entries bound to `127.0.0.1`.

Compare this port list with the Part 3 list. The DNS stub on `127.0.0.53:53` disappears from the external view, because the stub binds to the loopback range only. SSH on `0.0.0.0:22` appears in both views, which is exactly why you can reach the VM from your own machine. The difference between the two lists is the per-interface attack surface from the lecture.

Record which service from Part 2 is visible only from the loopback view.

## Part 5 — An authorized external target

Scanning a machine without permission is hostile and, in many jurisdictions, illegal. The Nmap project operates **scanme.nmap.org** as a public test target with explicit permission for a few light scans per day. Scan only this host, and only with the default scan:

```bash
nmap scanme.nmap.org
```

The report shows the open ports of a deliberately exposed server, typically SSH on 22 and HTTP on 80, along with the host's reverse DNS record. If the campus network blocks the probes and every port shows as filtered, record `scanme-open-ports: network blocked` and move on.

## Part 6 — Analysis

Answer in one sentence each, from what you observed:

1. Why did `ss -tulnp` hide the process names of system services from your account?
2. What did the difference between the loopback scan and the external-interface scan tell you about the VM's attack surface?
3. Pick one open port you observed. Following the lecture's rule, state whether you would close the port or filter the port with a firewall, and explain the choice.

## Assignment

Create a `submission.txt` file with the following entries:

- `nmap-missing: <YES or NO — nmap was unavailable and you used the ss fallback>`
- `listen-count: <the number of listening sockets from Part 2>`
- `loopback-open-ports: <comma-separated open ports on 127.0.0.1>`
- `service-ssh-version: <the VERSION column for port 22 from nmap -sV>`
- `external-open-ports: <comma-separated open ports on the external interface>`
- `localhost-only-service: <the service visible only from the loopback view>`
- `scanme-open-ports: <comma-separated open ports on scanme.nmap.org, or "network blocked">`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab16`) and name the file exactly `submission.txt`.

## Useful References and Resources

- `man nmap` — the full option list, including `-sV` for version detection and the privilege requirements of each scan type.
- `man ss` — the socket statistics tool used in Part 2, with the meaning of each flag.
- Nmap Network Scanning, Chapter 15 (nmap.org/book) — the official description of TCP connect versus SYN scanning.
- scanme.nmap.org — the authorized test target used in Part 5, with the project's scanning policy.
- IANA Service Name and Transport Protocol Port Number Registry (iana.org/assignments/service-names-port-numbers) — the authoritative port assignments behind nmap's service names.

## Optional Challenges

- Run `nmap -O 127.0.0.1` and `nmap -sU 127.0.0.1`, read the privilege error each one prints, and explain which operating system feature each scan type needs.
- Run `nc -vz 127.0.0.1 22` and `nc -vz 127.0.0.1 23`, and explain how the two outputs map to the open and closed port definitions from the lecture.
