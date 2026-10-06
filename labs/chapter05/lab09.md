# 09 — DNS Resolution, DNSSEC, and Hosts File Redirection — Lab

## Objectives

By the end of this lab, you will be able to:

1. Trace an iterative DNS resolution from the root zone through the top-level domain to the authoritative name server, and identify the zone boundaries and the authoritative answer flag in the output.
2. Inspect DNSSEC signatures on a signed domain, validate the signatures with `delv`, and observe how a validating resolver rejects broken DNS data while a non-validating query returns it.
3. Demonstrate URL redirection through the local hosts file, show that system tools follow the hosts file while `dig` bypasses it, and connect the behavior to the DNS poisoning attack from the lecture.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required: Part 4 demonstrates the hosts file attack inside a private mount namespace, and Part 1 notes an alternate path if the DNS tools are missing.
- The `dig` and `delv` command-line tools, installed by the `dnsutils` package on Debian/Ubuntu. Part 1 verifies both tools and installs the package if needed.
- Internet access from the VM. The course VM's network blocks outbound UDP and TCP port 53 to external DNS servers (a query such as `dig @ns1.google.com google.com` times out), so Part 2 provides a resolver-mediated fallback that works on the restricted network, and the `dig +trace` walkthrough is marked for use on a local machine or any unrestricted network.
- Chapter 5 lecture, Section 2: DNS, DNS resolution, DNS zones, authoritative name servers, domain reputation, URL redirection, DNS poisoning, and DNSSEC.

## Before you start

### Background

**DNS** is a hierarchical, decentralized naming system. No single server answers every query: a recursive resolver walks the hierarchy from the **root zone** (`.`) to the **top-level domain zone** (`.com`, `.org`) to the domain's **authoritative name server** (for example, *ns1.google.com* for *google.com*), following referrals at each step. A zone owner answers authoritatively for the zone's own records, which you can recognize by the `aa` (authoritative answer) flag.

| Concept from the lecture | Where you observe it in this lab |
| --- | --- |
| Hierarchical resolution and referrals | Part 2: `dig +trace google.com` walks root to .com to *google.com* (Option 1, local machine), or the `dig NS` zone walk lists the same three server sets (Option 2, VM) |
| Authoritative name server | Part 2 Option 1 (local machine): querying *ns1.google.com* directly returns the `aa` flag |
| DNSSEC data origin authentication and integrity | Part 3: `dig +dnssec` shows RRSIG signatures, `delv` validates them |
| DNSSEC discarding invalid data | Part 3: `dnssec-failed.org` returns SERVFAIL, but succeeds with checking disabled |
| URL redirection through the hosts file | Part 4: a fake `/etc/hosts` entry overrides real DNS for system tools |

The hosts file attack matters because of **resolution order**: the operating system's resolver checks `/etc/hosts` *before* sending any DNS query. An attacker who writes one line into that file redirects a domain for every browser and application on the machine, without touching any DNS server. That is the local version of the poisoning attack shown in the lecture's normal-versus-poisoned resolution figure.

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab09`
  - Go to folder: `cd ~/comp301/lab09`

## Part 1 — Setup and a baseline query

Verify the tools, installing them if necessary:

```bash
cd ~/comp301/lab09
dig -v
delv -v
# Both installed but if you run locally and either command is missing run the following:
# sudo apt-get update && sudo apt-get install -y dnsutils
```

Run a baseline query twice and compare the `Query time` line:

```bash
dig google.com
dig google.com
```

Read the sections of the output. The `ANSWER SECTION` holds the A record, and the `flags` line shows `rd` (recursion desired, your request) and `ra` (recursion available, the resolver's offer). **The second run answers from the resolver's cache**, so the `Query time` drops, often from tens of milliseconds to zero or one millisecond.

Get a compact answer for the record:

```bash
dig google.com +short
```

## Part 2 — Walking the hierarchy with dig +trace

### \[NOT for the VM\] Option 1: you can run locally

A normal query hides the hierarchy behind the recursive resolver. The `+trace` option turns `dig` into its own resolver and shows every referral:

```bash
dig +trace google.com
```

Read the output in three handoffs:

1. **Root zone:** the first block lists the root name servers (`a.root-servers.net` through `m.root-servers.net`) for the `.` zone.
2. **Top-level domain:** the next block is the root's referral, listing the `.com` name servers (`a.gtld-servers.net` and its peers).
3. **Domain zone:** the `.com` servers refer you to the *google.com* authoritative servers (*ns1.google.com* and its peers), and the final block carries the A record answer.

Record one server name from each of the three stages.

Now query the authoritative server directly and compare the flags with the baseline query:

```bash
dig @ns1.google.com google.com | grep flags
```

The authoritative server answers with the `aa` flag set and without `ra`, because *ns1.google.com* answers only for zones the server owns and offers no recursion. This is the same server the lecture names as authoritative for *google.com*.

### \[For the VM\] Option 2: When the network blocks outbound port 53 to external DNS servers

Direct queries in **\[Option 1\]** time out with `communications error` or `no servers could be reached`, the network blocks outbound port 53 to external DNS servers. Two workarounds exist.

- First, try forcing TCP, which some networks allow when UDP is filtered: `dig +tcp @ns1.google.com google.com`.
- Second, walk the zone chain through the VM's own resolver, which asks the questions on your behalf:

```bash
dig NS . +short
dig NS com. +short
dig NS google.com. +short
```

The three outputs list the root servers, the `.com` servers, and the *google.com* authoritative servers, the same three stages as `+trace`, though the resolver does the walking and the `aa` flag cannot appear in a recursive answer. **Record these server names for the Assignment**.

> **\[NOTE\]**
>
> For example, you can use `echo` with command substitution to write a field directly to `submission.txt` as follows:
>
> ```bash
> echo "baseline-ip: $(dig google.com +short | head -1)" >> submission.txt
> echo "trace-root-server: $(dig NS . +short | paste -sd, - | sed 's/,/, /g')" >> submission.txt
> echo "trace-tld-server: $(dig NS com. +short | paste -sd, - | sed 's/,/, /g')" >> submission.txt
> echo "trace-authoritative: $(dig NS google.com. +short | paste -sd, - | sed 's/,/, /g')" >> submission.txt
> ```
>
> Write the free-text analysis fields with plain `echo`:
>
> ```bash
> echo "analysis-1: The second query answered from the resolver cache, so no network lookup was needed." >> submission.txt
>   ```
>
> The `>>` operator appends, so run each command once and check your work with `cat submission.txt` as you go.

## Part 3 — DNSSEC: signatures and validation

The VM's default path answers through the systemd-resolved stub at 127.0.0.53, which strips DNSSEC records because Ubuntu runs resolved with `DNSSEC=no` by default. Find the real resolver the stub forwards to, and use that address for every query in this part:

```bash
resolvectl status | grep -B2 -A3 "DNS Servers"
```

The upstream resolver answers on port 53 even on networks that block external DNS servers. In the commands below, replace `<upstream>` with that address.

Start with a discovery: count the signature records for *google.com*:

```bash
dig +dnssec google.com @<upstream> | grep -c RRSIG
```

> **\[NOTE\]**
>
> You can save the `<upstream>` IP in a shell variable to use it later:
>
> ```bash
> UP=10.x.x.x      # use the right IP
> echo $UP         # read it back to confirm
> ```
>
> Or capture the address automatically from `resolvectl`:
>
> ```bash
> UP=$(resolvectl status | grep -m1 -oE '([0-9]{1,3}\.){3}[0-9]{1,3}')
> echo $UP
> ```
>
> The variable lives only in the current shell session. If you disconnect and SSH in again, run the assignment command once more before continuing.
> The previous command becomes:
>
> ```bash
> dig +dnssec google.com @$UP | grep -c RRSIG
> ```

The count is zero. *google.com* is not a signed zone, which shows that DNSSEC deployment remains incomplete even among major domains. Repeat the query against *cloudflare.com*, a signed zone:

```bash
dig +dnssec cloudflare.com @<upstream> | grep RRSIG
dig +dnssec +short DNSKEY cloudflare.com @<upstream>
```

The `RRSIG` record is the zone's digital signature over the answer, produced with the zone's private key, and the `DNSKEY` records carry the public keys a resolver uses to validate the signature.

Validate the chain yourself with `delv`, which fetches the records through the resolver you name and checks the signatures locally against the root trust anchor:

```bash
delv @<upstream> cloudflare.com
```

`delv` builds the chain of trust from the root and reports `; fully validated` above the answer.

Now watch a validator reject broken data. The domain `dnssec-failed.org` is a public test domain with a deliberately broken signature. The campus resolver does not validate DNSSEC, so query a validating resolver through its DNS over HTTPS (DoH) JSON API, which travels over TCP 443 and works on any network that allows web traffic:

```bash
curl -s 'https://dns.google/resolve?name=dnssec-failed.org&type=A&do=1'
curl -s 'https://dns.google/resolve?name=dnssec-failed.org&type=A&cd=1'
```

The first request asks dns.google, a validating resolver, for the answer with DNSSEC checking on (`do=1`), and the JSON reply shows `"Status": 2`, which is SERVFAIL: the resolver discarded the broken data, exactly the failure branch of the DNSSEC validation figure in the lecture. The second request sets `cd=1` (checking disabled), and the broken data comes back with `"Status": 0`, NOERROR. On an unrestricted network, the equivalent `dig` form is `dig dnssec-failed.org @8.8.8.8` for SERVFAIL and `dig +cd dnssec-failed.org @8.8.8.8` for NOERROR. DNSSEC protection works only when validation is enabled.

## Part 4 — Hosts file redirection

First record the real resolution of `example.com` through the system resolver, the path every browser and application uses:

```bash
getent hosts example.com
ping -c 1 example.com
```

Record the public IP address shown.

Editing `/etc/hosts` requires administrator rights, which student accounts on the VM do not have, and an attacker with the same access faces the same barrier. This part demonstrates the redirection in two ways that need no administrator rights.

**Method 1: a per-command mapping.** `curl` accepts a substituted address for one request, which shows the effect on an application directly:

```bash
curl -sS -m 5 -o /dev/null -w '%{http_code}\n' http://example.com
curl -sS -m 5 --resolve example.com:80:127.0.0.1 http://example.com
```

The first request reaches the real site and prints a success code such as 200. The second forces *example.com* to 127.0.0.1, the stand-in for an attacker's server, and fails with a connection error because nothing listens on the loopback address. One substituted address redirected the application with no DNS change anywhere.

**Method 2: the real hosts file mechanism in a private namespace.** Linux user namespaces provide a private mount table without administrator rights, so you can bind a modified copy of the hosts file over `/etc/hosts` for one shell:

```bash
cp /etc/hosts myhosts
echo "127.0.0.1 example.com www.example.com" >> myhosts
unshare -rm bash -c 'mount --bind "$HOME/comp301/lab09/myhosts" /etc/hosts && \
  echo "-- inside the namespace --" && \
  getent hosts example.com && \
  ping -c 1 example.com; \
  curl -sS -m 5 -I http://example.com; \
  dig example.com +short'
```

Inside the namespace, `getent` and `ping` resolve *example.com* to 127.0.0.1 and `curl` fails, while `dig` still returns the real public IP, because the system resolver reads the hosts file before consulting DNS and `dig` talks to DNS servers directly. A browser running inside this namespace would follow the hosts file exactly as `ping` and `curl` did: one edited line redirected the domain for every application that uses the system resolver, which is precisely the hosts file variant of the URL redirection attack from the lecture.

Exiting the command closes the namespace and discards the bind mount, so nothing needs cleanup. Confirm on the normal shell:

```bash
getent hosts example.com && ping -c 1 example.com
```

The output must show the real public IP again, not 127.0.0.1. If `unshare` reports `operation not permitted`, the kernel disallows unprivileged user namespaces on this system; rely on Method 1 as the demonstration and record that in your submission.

## Part 5 — Analysis

Answer in one sentence each, from what you observed:

1. Why did the second `dig google.com` in Part 1 answer faster than the first?
2. In Part 4, which stages of the resolution chain from Part 2 did the attacker's hosts file line bypass?
3. Why did `dig` still show the real IP address for *example.com* while `ping` showed 127.0.0.1?

## Assignment

Create a `submission.txt` file with the following entries:

- `baseline-ip: <the address from dig google.com +short>`
- `baseline-query-time-2nd: <Query time of the second baseline query, in ms>`
- `trace-root-server: <one root server name from the Part 2 zone walk>`
- `trace-tld-server: <one .com server name from the Part 2 zone walk>`
- `trace-authoritative: <one google.com authoritative server from the Part 2 zone walk>`
- `dnssec-google-signed: <YES or NO — an RRSIG record appeared for google.com>`
- `dnssec-rrsig-cloudflare: <YES or NO — an RRSIG record appeared for cloudflare.com>`
- `delv-cloudflare: <the validation result line from delv>`
- `dnssec-failed-doh: <the Status value in the JSON reply with do=1>`
- `dnssec-failed-doh-cd: <the Status value in the JSON reply with cd=1>`
- `hosts-before: <real public IP of example.com>`
- `hosts-after-ping: <IP ping used inside the namespace, or "namespace unavailable, Method 1 used">`
- `hosts-after-dig: <IP dig returned inside the namespace>`
- `hosts-restored: <YES or NO — getent shows the real IP after exiting the namespace>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab09`) and name the file exactly `submission.txt`.

## Useful References and Resources

- `man dig` and `man delv` — the full option lists, including `+trace`, `+dnssec`, `+cd`, and `+short`.
- RFC 8499 — DNS Terminology, the definitions of zone, referral, and authoritative server used in Parts 2 and 4.
- RFC 4033 — DNS Security Introduction and Requirements, the threat model behind DNSSEC.
- dnssec-failed.org — the public broken-signature test domain used in Part 3, operated by the Internet Society.
- Google Public DNS JSON API documentation (developers.google.com/speed/public-dns/docs/doh) — the DoH endpoint used in Part 3.
- IANA Root Zone Database (iana.org/domains/root) — the authoritative list of root zone contents and root server operators.

## Optional Challenges

- Run `delv @<upstream> dnssec-failed.org` and read the validation failure chain line by line, identifying which signature check fails first.
- Query a DoH resolver from the lecture's table with `curl -sH 'accept: application/dns-json' 'https://cloudflare-dns.com/dns-query?name=google.com&type=A'` and compare the JSON answer with the `dig +short` answer from Part 1.
