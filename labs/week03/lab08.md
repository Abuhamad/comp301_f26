# 08 — EAP Authentication Methods — Lab

## Objectives

By the end of this lab, you will be able to:

1. Compare EAP-TLS, EAP-FAST, EAP-TTLS, and PEAP on the three axes that distinguish them — who authenticates whom (mutual vs server-only), whether a client certificate is required, and how the user's credential is carried (directly, inside a TLS tunnel, or via a PAC).
2. Demonstrate mutual certificate authentication (EAP-TLS) versus server-only authentication (PEAP/EAP-TTLS outer tunnel) using Python's TLS stack and certificates from a lab CA.
3. Show why a tunneled method protects a legacy password that would otherwise be exposed, and select the correct EAP method for a stated deployment scenario with justification.

## Prerequisites

- A Unix-like shell (macOS Terminal or Linux) with `openssl` and `python3` (3.6 or newer) available. All demo code uses only the Python standard library (`ssl`, `socket`, `hmac`, `hashlib`, `secrets`, `struct`).
- SSH access to your assigned course VM.
- Prior lessons: Lab 05 (X.509 certificates and chains); Lab 06 (salted password hashing). TLS concepts from the lecture.

## Before you start

### Background

**EAP (Extensible Authentication Protocol, RFC 3748)** is a framework, not a single method: it carries an authentication conversation between a *supplicant* (client) and an *authentication server* (e.g., RADIUS), typically through an *authenticator* (a Wi-Fi access point or switch) that just relays EAP messages in 802.1X. The "method" is the specific authentication protocol carried inside EAP. The four methods differ on a few concrete axes you will observe in this lab:

| Method | Server cert required? | Client cert required? | How the client proves identity | Mutual auth? |
| --- | --- | --- | --- | --- |
| **EAP-TLS** | yes | **yes** | client certificate | yes, both via certificates |
| **EAP-FAST** | optional | no | credential inside a tunnel keyed by a **PAC** (Protected Access Credential) provisioned earlier | yes, via PAC/tunnel |
| **EAP-TTLS** | yes | no | *any* inner protocol — even legacy **PAP** (plain password) — inside the server-authenticated TLS tunnel | client authenticated inside tunnel |
| **PEAP** | yes | no | an inner EAP method (e.g., EAP-MSCHAPv2) inside the server-authenticated TLS tunnel | client authenticated inside tunnel |

The security idea is identical in PEAP and EAP-TTLS: the client first **authenticates the server** by its certificate and builds an encrypted TLS tunnel, *then* sends its password-based credential inside that tunnel — so the password is never exposed to an eavesdropper or a rogue access point that cannot present the right certificate. EAP-TLS skips the password entirely: both sides hold certificates, so the TLS handshake itself is mutual and there is no shared secret to phish. EAP-FAST replaces the server certificate with a **PAC** — a credential the server previously issued to that specific client — used to build the tunnel.

Two consequences to observe below: (1) methods that need no client certificate are far easier to deploy at scale (no per-device PKI), which is why PEAP dominates enterprise Wi-Fi despite being "only" password-based inside; (2) the inner credential's strength still matters — a weak password inside a PEAP tunnel is protected from eavesdroppers but not from guessing, which is why EAP-TLS is preferred where a device certificate can be provisioned.

### Set-up and Implementation

- On the VM, create a folder for this lab on the class folder on home directory.
  - create folder: `mkdir ~/comp301/lab08`
  - Go to folder `cd ~/comp301/lab08`

## Part 1 — Setup: a mini certificate authority

Every EAP method here depends on at least a **server** certificate. Create a small CA, then issue a **server** certificate and a **client** certificate — you will use them to show which methods require which.

```bash
cd ~/comp301/lab08

# Certificate Authority
openssl req -x509 -newkey rsa:2048 -keyout ca_key.pem -out ca.pem \
  -days 7 -nodes -subj "/CN=lab08-CA" 2>/dev/null

# Server certificate (the RADIUS / authentication server)
openssl req -newkey rsa:2048 -keyout server_key.pem -out server.csr \
  -nodes -subj "/CN=radius.example.edu" 2>/dev/null
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca_key.pem \
  -CAcreateserial -days 7 -out server.pem 2>/dev/null

# Client certificate (only EAP-TLS needs this)
openssl req -newkey rsa:2048 -keyout client_key.pem -out client.csr \
  -nodes -subj "/CN=alice@example.edu" 2>/dev/null
openssl x509 -req -in client.csr -CA ca.pem -CAkey ca_key.pem \
  -CAcreateserial -days 7 -out client.pem 2>/dev/null
```

Verify that both certificates chain back to the lab CA:

```bash
openssl verify -CAfile ca.pem server.pem
openssl verify -CAfile ca.pem client.pem
openssl x509 -in server.pem -noout -subject
openssl x509 -in client.pem -noout -subject
```

Record the two `subject=` lines. Both should report `OK` from `openssl verify`.

## Part 2 — EAP-TLS: mutual certificate authentication

EAP-TLS is the one method where the **client also presents a certificate**. You will model the EAP conversation directly as Python function calls and use the real `openssl verify` to do the certificate checking — so the logic you see (who presents a cert, who verifies it, what the server learns) is exactly what EAP-TLS does, without any network sockets.

First confirm the certificates actually verify against the CA (this is the real authentication check every method relies on):

```bash
openssl verify -CAfile ca.pem server.pem
openssl verify -CAfile ca.pem client.pem
```

Now model the EAP-TLS exchange. Create `eap_tls_demo.py`:

```bash
cat > eap_tls_demo.py <<'PY'
import subprocess

def verify(cert_file):
    """The real certificate check: does this chain to our CA?"""
    r = subprocess.run(["openssl", "verify", "-CAfile", "ca.pem", cert_file],
                       capture_output=True, text=True)
    return r.stdout.strip().endswith("OK")

def subject(cert_file):
    r = subprocess.run(["openssl", "x509", "-in", cert_file, "-noout", "-subject"],
                       capture_output=True, text=True)
    return r.stdout.strip()

def eap_tls(server_cert, client_cert):
    """EAP-TLS: BOTH sides present and verify a certificate. No password exists."""
    print("  1. server sends its certificate ->", subject(server_cert))
    if not verify(server_cert):
        return "FAIL: client could not verify the server certificate"
    print("  2. client VERIFIED the server certificate")
    if client_cert is None:
        return "FAIL: EAP-TLS requires a client certificate, client has none"
    print("  3. client sends its certificate ->", subject(client_cert))
    if not verify(client_cert):
        return "FAIL: server could not verify the client certificate"
    print("  4. server VERIFIED the client certificate;",
          "server learns identity:", subject(client_cert))
    return "EAP-Success (mutual certificate authentication, no password used)"

print("[EAP-TLS, client has certificate]")
print(" ", eap_tls("server.pem", "client.pem"))

print("\n[EAP-TLS, client has NO certificate]")
print(" ", eap_tls("server.pem", None))
PY

python3 eap_tls_demo.py
```

Record the outcome of each scenario. The first reaches `EAP-Success` and the server learns the client's identity (`alice@example.edu`) from its certificate — this is mutual authentication with no password anywhere. The second fails: EAP-TLS *requires* the client certificate, so a device without one is rejected.

## Part 3 — PEAP / EAP-TTLS: server-only tunnel, password inside

PEAP and EAP-TTLS drop the client certificate requirement. The client authenticates **only the server** by certificate and builds an encrypted tunnel, then sends a password credential inside it. First, show what the tunnel is protecting against — a legacy cleartext password (PAP) sent with **no** tunnel:

```bash
cat > pap_cleartext.py <<'PY'
# PAP: the password travels as-is. Anything that can read the traffic reads the password.
wire = "USER alice PASS Tr0pical!9"   # this is what crosses the network unencrypted
print("  on the wire:", wire)
print("  -> an eavesdropper read the password 'Tr0pical!9' directly, no cracking needed.")
PY

python3 pap_cleartext.py
```

The password crosses readable by anyone watching. Now run the same credential **inside a server-authenticated TLS tunnel** — the PEAP/EAP-TTLS model. Note the client presents **no certificate**, yet the connection is still authenticated on the server side because the client verified the server's certificate:

```bash
cat > peap_ttls_demo.py <<'PY'
import subprocess, hashlib, hmac, secrets

def verify(cert_file):
    r = subprocess.run(["openssl", "verify", "-CAfile", "ca.pem", cert_file],
                       capture_output=True, text=True)
    return r.stdout.strip().endswith("OK")

def peap_ttls(server_cert, client_cert, password):
    """PEAP / EAP-TTLS: only the SERVER presents a certificate. The client builds an
    encrypted tunnel from it, then proves its identity INSIDE with a password."""
    print("  1. server sends its certificate; client presents:",
          client_cert or "no certificate")
    if not verify(server_cert):
        return "FAIL: client could not verify the server certificate"
    print("  2. client VERIFIED the server certificate -> TLS tunnel is up")
    print("     (client needed no certificate -> no per-device PKI required)")
    # Inner credential travels inside the tunnel. Use challenge-response so the
    # raw password is not sent even inside the tunnel (EAP-TTLS allows MSCHAPv2).
    challenge = secrets.token_bytes(16)
    proof = hmac.new(password.encode(), challenge, hashlib.sha256).hexdigest()
    expected = hmac.new(b"Tr0pical!9", challenge, hashlib.sha256).hexdigest()
    ok = hmac.compare_digest(proof, expected)
    print("  3. inner challenge-response inside tunnel:",
          "ACCEPT" if ok else "REJECT")
    return "EAP-Success (password protected by tunnel)" if ok else "FAIL: bad password"

print("[PEAP / EAP-TTLS, client has NO certificate, uses a password]")
print(" ", peap_ttls("server.pem", None, "Tr0pical!9"))
PY

python3 peap_ttls_demo.py
```

The client presented **no certificate**, yet authentication succeeds because the client verified the server's certificate and proved its password inside the tunnel — that is the whole deployment appeal of PEAP/EAP-TTLS (no per-device PKI). Compare with Part 2: EAP-TLS got the client's identity from a certificate, PEAP/EAP-TTLS get it from a password checked inside the tunnel.

To see why the inner challenge–response matters (it keeps the raw password out of even the tunnel and blocks replay), create `inner_auth.py`:

```bash
cat > inner_auth.py <<'PY'
import hashlib, hmac, secrets

# Server stores a SALTED HASH (Lab 06), never the password.
salt = secrets.token_bytes(16)
stored = hashlib.pbkdf2_hmac("sha256", b"Tr0pical!9", salt, 600_000)

# Server issues a fresh challenge for this session.
challenge = secrets.token_bytes(16)

# Client answers HMAC(password, challenge) — the password itself is never sent,
# and a replayed response fails because the next challenge differs.
def respond(password, chal):
    return hmac.new(password.encode(), chal, hashlib.sha256).hexdigest()

client_proof = respond("Tr0pical!9", challenge)
# Server recomputes the expected answer from the same password and challenge.
server_expected = respond("Tr0pical!9", challenge)
print("challenge-response inner auth:",
      "ACCEPT" if hmac.compare_digest(client_proof, server_expected) else "REJECT")

# Replay: an attacker reuses the captured proof against a NEW challenge.
new_challenge = secrets.token_bytes(16)
replayed = client_proof
expected_new = respond("Tr0pical!9", new_challenge)
print("replayed response vs new challenge:",
      "ACCEPT" if hmac.compare_digest(replayed, expected_new) else "REJECT")
PY

python3 inner_auth.py
```

Observe that the proof is accepted once but a replayed proof fails against a new challenge — challenge–response inner methods add replay resistance on top of the tunnel.

## Part 4 — EAP-FAST: the PAC

EAP-FAST supports certificates but does not require them. Instead the server provisions each client a **PAC** — a client-specific credential — and the PAC (not a server certificate) is what keys the tunnel. Model provisioning and PAC-based tunnel setup with symmetric HMAC:

```bash
cat > eap_fast_demo.py <<'PY'
import hashlib, hmac, secrets

# --- provisioning (done once, in-band, by the server) ---
pac_key = secrets.token_bytes(32)          # shared secret, unique per client
pac_id  = "alice-pac-001"                   # PAC-ID the client will present
print(f"provisioned PAC id={pac_id}")

# --- tunnel setup: client presents the PAC-ID, both sides derive a tunnel key ---
client_hello = secrets.token_bytes(16)      # client random
server_hello = secrets.token_bytes(16)      # server random

def derive_tunnel_key(pac, label, c, s):
    return hmac.new(pac, label + c + s, hashlib.sha256).digest()

client_tk = derive_tunnel_key(pac_key, b"EAP-FAST tunnel", client_hello, server_hello)
server_tk = derive_tunnel_key(pac_key, b"EAP-FAST tunnel", client_hello, server_hello)
print("tunnel key established from PAC:",
      "MATCH" if hmac.compare_digest(client_tk, server_tk) else "MISMATCH")

# --- a client with the WRONG PAC cannot build the tunnel ---
wrong_pac = secrets.token_bytes(32)
attacker_tk = derive_tunnel_key(wrong_pac, b"EAP-FAST tunnel", client_hello, server_hello)
print("attacker with wrong PAC derives same tunnel key:",
      "yes" if hmac.compare_digest(attacker_tk, server_tk) else "no (rejected)")
PY

python3 eap_fast_demo.py
```

Both sides derive the same tunnel key from the PAC, while a client holding the wrong PAC derives a different key and is rejected — the PAC plays the role the server certificate plays in PEAP/EAP-TTLS, binding the tunnel to a known client.

## Part 5 — Method selection (analysis)

Fill in the decision matrix from what you observed, then answer the scenarios.

| Requirement | EAP-TLS | EAP-FAST | EAP-TTLS | PEAP |
| --- | --- | --- | --- | --- |
| Server certificate required | | | | |
| Client certificate required | | | | |
| Works with legacy password (PAP) inside | | | | |
| No per-device PKI needed | | | | |
| Resists phishing of a shared secret | | | | |

For each scenario, pick **one** method and justify it in a sentence:

1. A corporate fleet of managed laptops where IT can push a certificate to every device.
2. University BYOD Wi-Fi where students connect personal phones and cannot be given client certificates.
3. A legacy system whose accounts database can only check plain passwords (PAP).
4. A deployment that wants tunnel security without running a server-certificate PKI.

## Assignment

Create a `submission.txt` file with the following entries:

- `server-subject: <value>`, `client-subject: <value>`,
- `eaptls-mutual: <the client identity the server saw>`,
- `eaptls-no-client-cert: <handshake outcome>`,
- `pap-cleartext: <what the server received>`,
- `peap-ttls-tunnel: <handshake outcome with no client cert>`,
- `inner-challenge-response: <ACCEPT or REJECT>`,
- `inner-replay: <ACCEPT or REJECT>`,
- `eapfast-tunnel-key: <MATCH or MISMATCH>`,
- `eapfast-wrong-pac: <accepted or rejected>`,
- `scenario-managed: <method + one-sentence justification>`,
- `scenario-byod: <method + justification>`,
- `scenario-legacy: <method + justification>`,
- `scenario-no-pki: <method + justification>`.

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab08`) and name the file exactly `submission.txt`.

## Verification / Completion Criteria

- `openssl verify -CAfile ca.pem server.pem` and `openssl verify -CAfile ca.pem client.pem` both print `OK`, and the two `subject=` lines are recorded in `submission.txt`.
- `python3 eap_tls_demo.py` shows the **EAP-TLS (mutual)** scenario succeed with the server seeing `alice@example.edu`, the **no client cert** scenario fail at the handshake, and the **PEAP/EAP-TTLS tunnel** scenario succeed with no client certificate.
- `python3 pap_cleartext.py` shows the password readable on the wire.
- `python3 inner_auth.py` prints `ACCEPT` for the fresh challenge–response and `REJECT` for the replayed proof.
- `python3 eap_fast_demo.py` prints `MATCH` for the PAC-derived tunnel key and `no (rejected)` for the wrong PAC.
- `submission.txt` selects a method for each of the four scenarios with a justification consistent with the certificate/tunnel/PAC trade-offs shown in the lab.

## Useful References and Resources

- RFC 3748 — Extensible Authentication Protocol (EAP).
- RFC 5216 — EAP-TLS Authentication Protocol.
- RFC 4851 — EAP-FAST Authentication Protocol.
- RFC 5281 — EAP-TTLS Authentication Protocol (EAP-TTLSv0).
- PEAP — Protected EAP Protocol (Microsoft/Cisco/RSA joint IETF drafts).
- NIST SP 800-63B — Digital Identity Guidelines: Authentication and Authenticator Management.
