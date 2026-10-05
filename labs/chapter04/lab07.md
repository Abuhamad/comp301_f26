# 07 — Multi-Factor Authentication: TOTP and HOTP — Lab

## Objectives

By the end of this lab, you will be able to:

1. Design an MFA policy that combines at least three distinct factor categories, and state the phishing and replay weaknesses that remain in each chosen factor.
2. Generate and compare TOTP and HOTP codes from a base32 seed, and explain why TOTP depends on clock alignment while HOTP depends on counter synchronization.
3. Build a two-step login (password + TOTP) in Python, demonstrate that an intercepted code fails outside its time window, and explain why FIDO2/WebAuthn resists phishing and replay.

## Prerequisites

- A Unix-like shell (macOS Terminal or Linux) with `python3` (3.6 or newer) available. All required code uses only the Python standard library (`base64`, `hmac`, `hashlib`, `struct`, `time`, `secrets`).
- Optional: `oathtool` and a TOTP authenticator app (e.g., Aegis, 1Password, Authy) for cross-checking codes. The lab works entirely without them.
- SSH access to your assigned course VM.
- Prior lessons: Lab 06 (password hashing and storage); hash functions and HMAC.

## Before you start

### Background

Multi-factor authentication requires evidence from **distinct factor categories**, so that compromising one factor does not compromise the account:

- **Knowledge** — something you *know*: password, PIN, security question.
- **Possession** — something you *have*: authenticator app or hardware token (TOTP/HOTP seed), smart card, FIDO2 security key, a phone receiving SMS.
- **Inherence** — something you *are*: fingerprint, face, voice.
- Additional signals used in practice: **location** (somewhere you are) and **behavior**.

A password plus a password hint is **not** MFA — both are knowledge factors, so one phishing attack steals both.

**HOTP** (RFC 4226) computes `HMAC(secret, counter)` and truncates the digest to a numeric code; the counter increments per use, so codes stay valid until consumed. **TOTP** (RFC 6238) reuses the HOTP construction with a counter derived from the clock: `counter = floor(unix_time / 30)`. Codes are only valid within their 30-second window. The shared secret is encoded in **base32** so it can be typed or scanned (as an `otpauth://` QR code) into authenticator apps.

Both OTP schemes share weaknesses: an attacker who intercepts a code (phishing site, keylogger) can replay it within its window, and a stolen seed compromises the factor permanently. **FIDO2/WebAuthn** avoids this class of attack: the authenticator signs a fresh per-login challenge with an asymmetric private key, and the signature is **bound to the relying party's origin**, so a phishing site on a different domain receives a response that is useless to it.

### Set-up and Implementation

- On the VM, create a folder for this lab on the class folder on home directory.
  - create folder: `mkdir ~/comp301/lab07`
  - Go to folder `cd ~/comp301/lab07`

## Part 1 — Setup

Create a working folder and set a test base32 secret (base32 of the ASCII string `Hello!\0xDEADBEEF`, a standard interoperability example):

```bash
SECRET=JBSWY3DPEHPK3PXP
printf "Using secret: %s\n" "$SECRET"
```

Check for `oathtool`. If `oathtool` exists, the walkthrough shows both `oathtool` and Python ways. If `oathtool` is missing, the Python steps still work.

```bash
command -v oathtool >/dev/null 2>&1 && echo "oathtool available" || echo "oathtool not found, using Python"
```

## Part 2 — Generate TOTP codes (walkthrough)

Using `oathtool` if available:

```bash
# Using oathtool if installed
oathtool --totp -b "$SECRET"
```

Using the Python reference implementation provided here:

```bash
cat > totp_hotp_demo.py <<'PY'
import base64, hmac, hashlib, struct, time
secret_b32 = 'JBSWY3DPEHPK3PXP'
key = base64.b32decode(secret_b32, casefold=True)

def hotp(key, counter, digits=6):
    counter_bytes = struct.pack('>Q', counter)
    h = hmac.new(key, counter_bytes, hashlib.sha1).digest()
    o = h[19] & 15
    code = (struct.unpack('>I', h[o:o+4])[0] & 0x7fffffff) % (10 ** digits)
    return str(code).zfill(digits)

def totp(key, for_time=None, step=30, digits=6):
    if for_time is None:
        for_time = int(time.time())
    counter = int(for_time / step)
    return hotp(key, counter, digits)

print('TOTP now:', totp(key))
print('TOTP after 31s:', totp(key, for_time=int(time.time())+31))
print('HOTP counter=1:', hotp(key, 1))
print('HOTP counter=2:', hotp(key, 2))
PY

python3 totp_hotp_demo.py
```

Observe that the `TOTP now` value differs from `TOTP after 31s` and that HOTP codes change only when the counter increments. (Optional: enter `JBSWY3DPEHPK3PXP` into a TOTP authenticator app as a manual entry and confirm it shows the same `TOTP now` code.)

## Part 3 — HOTP vs TOTP comparison

Generate two HOTP codes with consecutive counters and compare behavior.

```bash
# HOTP counter examples using oathtool if present
oathtool --hotp --counter=1 -b "$SECRET"
oathtool --hotp --counter=2 -b "$SECRET"

# Or using the Python demo
python3 totp_hotp_demo.py
```

Note that TOTP validity depends on clock alignment while HOTP validity depends on counter sync. Record what breaks each scheme: a client whose clock is off by 60 seconds (TOTP), or a token button pressed repeatedly without a login (HOTP counter drift — real servers allow a small "look-ahead" window and then resynchronize).

## Part 4 — Replay-resistance check

Capture a TOTP code and attempt to reuse it after its window. Use the demo output and try submitting the captured code to a verifier only after waiting 31 seconds. Observe that the code no longer matches the current TOTP.

Commands to observe this behavior using the demo script:

```bash
# Run demo to capture the current TOTP
python3 totp_hotp_demo.py > demo_output.txt
grep 'TOTP now' demo_output.txt -n -H || true
# Wait for 31 seconds, then run the demo again and compare
sleep 31
python3 totp_hotp_demo.py > demo_output_after.txt
diff demo_output.txt demo_output_after.txt || true
```

Discuss why an attacker who intercepts a TOTP code can succeed only within the code window, and why FIDO2/WebAuthn avoids this class of attack by binding responses to the origin and using asymmetric signatures.

## Part 5 — A two-step login: password + TOTP

Combine what you learned in Lab 06 with this lab: the server stores a PBKDF2 password record (factor 1 — knowledge) and a base32 TOTP seed (factor 2 — possession), and login requires both.

```bash
cat > mfa_login.py <<'PY'
import base64, hashlib, hmac, secrets, struct, time

# --- account registration ---
def register(user, password):
    salt = secrets.token_bytes(16)
    dk = hashlib.pbkdf2_hmac("sha256", password.encode(), salt, 600_000)
    seed = base64.b32encode(secrets.token_bytes(20)).decode()
    with open("mfa_db.txt", "a") as db:
        db.write(f"{user}:$pbkdf2-sha256$600000${salt.hex()}${dk.hex()}:{seed}\n")
    return seed

# --- factors ---
def check_password(record, candidate):
    _, alg, iters, salt_hex, expected = record.split("$")
    dk = hashlib.pbkdf2_hmac(alg.replace("pbkdf2-", ""), candidate.encode(),
                             bytes.fromhex(salt_hex), int(iters))
    return hmac.compare_digest(dk.hex(), expected)

def totp(seed, for_time=None, step=30, digits=6):
    key = base64.b32decode(seed, casefold=True)
    if for_time is None:
        for_time = int(time.time())
    h = hmac.new(key, struct.pack(">Q", int(for_time / step)), hashlib.sha1).digest()
    o = h[19] & 15
    return str((struct.unpack(">I", h[o:o+4])[0] & 0x7fffffff) % (10 ** digits)).zfill(digits)

def login(user, password, code):
    for line in open("mfa_db.txt"):
        u, record, seed = line.strip().split(":")
        if u == user:
            f1 = check_password(record, password)
            f2 = hmac.compare_digest(totp(seed), code)
            print(f"factor1(knowledge)={f1} factor2(possession)={f2}")
            return f1 and f2
    return False

# --- demo ---
seed = register("alice", "Tr0pical!9")
print(f"alice's TOTP seed: {seed}  (this line is what a QR code would carry)")
good = totp(seed)
print("correct password + current code ->", "ACCEPT" if login("alice", "Tr0pical!9", good) else "REJECT")
print("wrong password + current code   ->", "ACCEPT" if login("alice", "wrong", good) else "REJECT")
print("correct password + stale code   ->", "ACCEPT" if login("alice", "Tr0pical!9", totp(seed, int(time.time()) - 60)) else "REJECT")
print("correct password + random code  ->", "ACCEPT" if login("alice", "Tr0pical!9", "000000") else "REJECT")
PY

python3 mfa_login.py
```

Record the four results. Observe that the stolen-code replay (the stale code from 60 seconds ago) is rejected, and that either factor failing is enough to reject the login.

## Part 6 — Design a three-factor policy (discussion)

Write a short MFA policy for **one** scenario: a hospital's patient-records system, an online bank, or this university's student portal. Your policy must:

1. Combine factors from at least **three distinct categories** (e.g., password + TOTP app + fingerprint + location check).
2. State for **each** chosen factor how it could be attacked (phished, SIM-swapped, spoofed, stolen seed, replayed) and one mitigation.
3. State which single factor you would upgrade to FIDO2/WebAuthn first, and why.

## Assignment

Create a `submission.txt` file with the following entries:

- `totp-now: <value>`,
- `totp-after-31s: <value>`,
- `hotp-counter-1: <value>`,
- `hotp-counter-2: <value>`,
- `totp-source: <python3 or oathtool or authenticator-app>`,
- `hotp-vs-totp: <one sentence stating what each one depends on>`,
- `replay-observation: <one sentence on what diff showed after 31 seconds>`,
- `mfa-accept-correct: <ACCEPT or REJECT>`,
- `mfa-reject-wrong-password: <ACCEPT or REJECT>`,
- `mfa-reject-stale-code: <ACCEPT or REJECT>`,
- `factor-categories: <your three chosen categories>`,
- `factor-attacks: <one attack per chosen factor>`,
- `fido2-upgrade: <the factor you would replace first and one reason>`,
- `phishing-justify: <one sentence on why WebAuthn's origin binding defeats the replay attack from Part 4>`.

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab07`) and name the file exactly `submission.txt`.

## Verification / Completion Criteria

- `python3 totp_hotp_demo.py` prints a `TOTP now` that differs from `TOTP after 31s`, and `HOTP counter=1` that differs from `HOTP counter=2`; all four values appear in `submission.txt`.
- The Part 4 `diff` shows the captured `TOTP now` no longer matches after the 31-second wait, and `submission.txt` records this as `replay-observation`.
- `python3 mfa_login.py` prints `factor1(knowledge)=True factor2(possession)=True` followed by `ACCEPT` for the correct combination, and rejects the wrong-password, stale-code, and random-code attempts.
- `mfa_db.txt` contains a record of the form `user:$pbkdf2-sha256$600000$<salt>$<hash>:<base32 seed>`, showing both factors stored server-side.
- `submission.txt` lists three **distinct** factor categories, one attack per factor, and a `phishing-justify` sentence that references origin binding or per-login asymmetric signatures.

## Useful References and Resources

- RFC 4226 — HOTP: HMAC-Based One-Time Password Algorithm.
- RFC 6238 — TOTP: Time-Based One-Time Password Algorithm.
- FIDO2 / WebAuthn specification (FIDO Alliance and W3C).
- `oathtool` project documentation and man page.
- NIST SP 800-63B — Digital Identity Guidelines: Authentication and Authenticator Management.
