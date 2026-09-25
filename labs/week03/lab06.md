# 06 — Password Storage: Hashing and Salting — Lab

## Objectives

By the end of this lab, you will be able to:

1. Demonstrate why storing passwords in plaintext, reversibly encrypted, or as fast unsalted hashes fails, by cracking an unsalted SHA-256 password database with a small dictionary attack.
2. Construct per-user salted password records and use two users who share one password to show how a unique random salt defeats rainbow tables and cross-user precomputation.
3. Produce a standards-style password record (`$algorithm$cost$salt$hash`) using a slow password-hashing KDF (PBKDF2-HMAC-SHA256 / scrypt), verify login attempts against it with a constant-time comparison, and measure how the work factor reduces an attacker's guess rate.

## Prerequisites

- A Unix-like shell (macOS Terminal or Linux) with `openssl` and `python3` (3.6 or newer) available.
- SSH access to your assigned course VM.
- Prior lessons: hash functions and MACs; completion of Lab 03–05 (comfort with `openssl`, redirection, and small `python3` snippets).

## Before you start

### Background

Assume the password database **will** be stolen (backups leak, SQL injection, insider). The defender's goal is to make the stolen file useless for *offline* guessing:

- **Plaintext storage** fails immediately: every account is compromised the moment the file leaks.
- **Encrypting passwords** is the wrong tool: encryption is reversible, so anyone who steals the file *and* the decryption key recovers every password. Passwords must be **hashed**, not encrypted.
- A **fast hash alone (SHA-256, MD5)** is not enough: an attacker with a GPU can test *billions* of guesses per second, and identical passwords produce identical hashes, revealing which users share a password. Worse, **rainbow tables** let an attacker precompute hashes of common passwords once and reuse them against every database.
- A **salt** (a unique, random, *non-secret* value stored next to each hash) forces the attacker to recompute every guess for every user separately and makes precomputed tables useless. Hash `salt + password`, never the password alone.
- A **slow, work-factor-tunable KDF** (Argon2id, scrypt, bcrypt, or PBKDF2 with a high iteration count) makes *each* guess expensive for the attacker while staying cheap enough (~250 ms–1 s) for a single legitimate login.
- Real systems store everything needed to recompute the hash in one self-describing string, e.g. `/etc/shadow` uses `$6$salt$hash` (sha512crypt) and modern systems use `$argon2id$v=19$m=...,t=...,p=...$salt$hash`. This lab uses the same pattern: `$pbkdf2-sha256$iterations$salt$hash`.
- Verification must compare hashes with a **constant-time comparison** (`hmac.compare_digest`) so the time taken does not leak how many characters were correct.

### Set-up and Implementation

- On the VM, create a folder for this lab on the class folder on home directory.
  - create folder: `mkdir ~/comp301/lab06`
  - Go to folder `cd ~/comp301/lab06`

## Part 1 — Setup: users and a dictionary

Create a small set of accounts. Note that `alice` and `bob` deliberately use the **same** password.

```sh
cat > users.txt <<'EOF'
alice:Sunshine23
bob:Sunshine23
carol:qwerty7
dave:Tr0pical!9
EOF
```

Create a small "attacker dictionary" of common passwords. (Real dictionaries such as `rockyou.txt` hold millions of entries; weak passwords appear in all of them.)

```sh
cat > dictionary.txt <<'EOF'
password
123456
letmein
dragon
iloveyou
Sunshine23
qwerty7
Tr0pical!9
EOF
```

## Part 2 — The naive approach: unsalted fast hash

Build a password database the wrong way: `username:sha256(password)`.

```sh
while IFS=: read -r user pw; do
  h=$(printf '%s' "$pw" | openssl dgst -sha256 | awk '{print $NF}')
  echo "$user:$h" >> naive_db.txt
done < users.txt
cat naive_db.txt
```

Record what you observe: which two users have **identical hashes**, and what does that reveal?

Now play the attacker who stole `naive_db.txt`. Hash each dictionary word once and match it against every account:

```sh
python3 - <<'PY'
import hashlib, time

db = {}
for line in open("naive_db.txt"):
    user, h = line.strip().split(":", 1)
    db.setdefault(h, []).append(user)

words = [w.strip() for w in open("dictionary.txt")]
t0 = time.time()
for w in words:
    h = hashlib.sha256(w.encode()).hexdigest()
    for u in db.get(h, []):
        print(f"CRACKED {u}: password is '{w}'")
dt = time.time() - t0
print(f"tested {len(words)} guesses in {dt:.6f}s ({len(words)/dt:,.0f} hashes/sec)")
PY
```

Record the guess rate. A single table of hashes cracked every account whose password appears in the dictionary — this is the failure mode a salt must prevent.

## Part 3 — Add a random salt

Rebuild the database as `username:salt:hash`, with a fresh 128-bit random salt per user:

```sh
python3 - <<'PY'
import hashlib, secrets

with open("salted_db.txt", "w") as db:
    for line in open("users.txt"):
        user, pw = line.strip().split(":", 1)
        salt = secrets.token_bytes(16)   # unique random salt per user
        h = hashlib.sha256(salt + pw.encode()).hexdigest()
        db.write(f"{user}:{salt.hex()}:{h}\n")

print(open("salted_db.txt").read())
PY
```

Observe: `alice` and `bob` now have completely different records even though their passwords are identical. The salt is stored **in the clear** next to the hash — it is not a secret; it only needs to be unique so that no two records share a precomputation.

Rerun the dictionary attack. It must now recompute every guess for every `(salt, user)` pair, and no precomputed table helps:

```sh
python3 - <<'PY'
import hashlib

rows = [l.strip().split(":") for l in open("salted_db.txt")]
words = [w.strip() for w in open("dictionary.txt")]
guesses = 0
for user, salt_hex, h in rows:
    salt = bytes.fromhex(salt_hex)
    for w in words:
        guesses += 1
        if hashlib.sha256(salt + w.encode()).hexdigest() == h:
            print(f"CRACKED {user}: password is '{w}'")
print(f"required {guesses} salted hashes for {len(rows)} users "
      f"(vs {len(words)} unsalted hashes total in Part 2)")
PY
```

The salt removed sharing and precomputation, but each guess is still one fast SHA-256 — brute force remains cheap. The missing piece is *slowness*.

## Part 4 — Use a slow password-hashing KDF

Measure how the choice of function changes the cost of one guess:

```sh
python3 - <<'PY'
import hashlib, secrets, time

password = b"Sunshine23"
salt = secrets.token_bytes(16)

def bench(label, fn):
    t0 = time.time(); r = fn(); dt = time.time() - t0
    print(f"{label:<30} {dt*1000:9.2f} ms -> {r.hex()[:32]}...")

bench("SHA-256 (1 round)", lambda: hashlib.sha256(salt + password).digest())
bench("PBKDF2-HMAC-SHA256 600k iters", lambda: hashlib.pbkdf2_hmac("sha256", password, salt, 600_000))
bench("scrypt N=2^14 r=8 p=1", lambda: hashlib.scrypt(password, salt=salt, n=2**14, r=8, p=1))
PY
```

Record the three timings. The KDFs are deliberately hundreds of thousands of times slower; scrypt additionally consumes ~16 MiB of memory per guess, which defeats massively parallel GPU cracking. (In production, prefer **Argon2id**, then scrypt or bcrypt; PBKDF2 is the accepted minimum and is built into Python, which is why this lab uses it.)

Build the real database in a self-describing modular format, `$algorithm$cost$salt$hash`:

```sh
python3 - <<'PY'
import hashlib, secrets

ITER = 600_000
with open("shadow_db.txt", "w") as db:
    for line in open("users.txt"):
        user, pw = line.strip().split(":", 1)
        salt = secrets.token_bytes(16)
        dk = hashlib.pbkdf2_hmac("sha256", pw.encode(), salt, ITER)
        db.write(f"{user}:$pbkdf2-sha256${ITER}${salt.hex()}${dk.hex()}\n")

print(open("shadow_db.txt").read())
PY
```

This mirrors real formats: `$6$salt$hash` (sha512crypt) in `/etc/shadow`, or `$argon2id$v=19$m=65536,t=3,p=4$salt$hash`. Everything needed to verify a login — algorithm, work factor, salt, and hash — travels in one string.

## Part 5 — Verify logins and measure the attacker's cost

Write a verifier that parses the stored record, recomputes the KDF on the candidate password, and compares in **constant time**:

```sh
cat > verify.py <<'PY'
import hashlib, hmac, sys

def verify(user, candidate):
    for line in open("shadow_db.txt"):
        u, record = line.strip().split(":", 1)
        if u != user:
            continue
        _, alg, iters, salt_hex, expected = record.split("$")
        dk = hashlib.pbkdf2_hmac(alg.replace("pbkdf2-", ""), candidate.encode(),
                                 bytes.fromhex(salt_hex), int(iters))
        return hmac.compare_digest(dk.hex(), expected)
    return False

print("ACCEPT" if verify(sys.argv[1], sys.argv[2]) else "REJECT")
PY
```

Test a correct and an incorrect login:

```sh
python3 verify.py alice Sunshine23
python3 verify.py alice WrongPassword
```

Finally, rerun the dictionary attack against `shadow_db.txt` and record the attacker's new guess rate:

```sh
python3 - <<'PY'
import hashlib, hmac, time

rows = [l.strip().split(":", 1) for l in open("shadow_db.txt")]
words = [w.strip() for w in open("dictionary.txt")]
total = len(rows) * len(words)
t0 = time.time()
for user, record in rows:
    _, alg, iters, salt_hex, expected = record.split("$")
    for w in words:
        dk = hashlib.pbkdf2_hmac("sha256", w.encode(), bytes.fromhex(salt_hex), int(iters))
        if hmac.compare_digest(dk.hex(), expected):
            print(f"CRACKED {user}: password is '{w}'")
dt = time.time() - t0
print(f"{total} KDF guesses in {dt:.2f}s -> {total/dt:,.1f} guesses/sec on this machine")
PY
```

Compare the guesses/sec here with the hashes/sec from Part 2 and compute the slowdown factor. Note also what hashing **cannot** fix: the weak passwords were still cracked, just slower — which is why password policy, rate limiting, and MFA must accompany good storage.

## Assignment

Create a `submission.txt` file with the following entries:

- `naive-hash-alice: <value>`,
- `naive-hash-bob: <value>`,
- `identical-hash-users: <the two users sharing a hash, and one sentence on what that leaks>`,
- `salted-record-alice: <user:salt:hash>`,
- `kdf-record-alice: <full $pbkdf2-sha256$... record>`,
- `iterations: <value>`, `time-sha256-ms: <value>`,
- `time-pbkdf2-ms: <value>`, `time-scrypt-ms: <value>`,
- `verify-correct: <ACCEPT or REJECT>`,
- `verify-wrong: <ACCEPT or REJECT>`,
- `crack-rate-unsalted: <hashes/sec from Part 2>`,
- `crack-rate-kdf: <guesses/sec from Part 5>`,
- `slowdown-factor: <ratio of the two rates>`,
- `remaining-risk: <one sentence on why weak passwords still need policy/MFA>`.

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab06`) and name the file exactly `submission.txt`.

## Verification / Completion Criteria

- `cat naive_db.txt` shows two identical hashes, and the Part 2 script prints `CRACKED` lines for every account whose password is in `dictionary.txt`.
- `cat salted_db.txt` shows no two identical salts or hashes, and the Part 3 attack reports more salted hashes than the number of unsalted hashes needed in Part 2.
- `cat shadow_db.txt` contains records of the form `user:$pbkdf2-sha256$600000$<salt>$<hash>`, matching the values recorded in `submission.txt`.
- `python3 verify.py alice Sunshine23` prints `ACCEPT` and `python3 verify.py alice WrongPassword` prints `REJECT`.
- `submission.txt` contains both crack rates and a `slowdown-factor` of at least several orders of magnitude, plus a `remaining-risk` sentence that references password policy or MFA.

## Useful References and Resources

- OWASP Password Storage Cheat Sheet — recommended algorithms and work factors.
- NIST SP 800-63B — Digital Identity Guidelines: Authentication and Authenticator Management.
- RFC 8018 — PKCS #5: Password-Based Cryptography Specification (PBKDF2).
- RFC 9106 — Argon2 Memory-Hard Function for Password Hashing.
- `crypt(3)` and `shadow(5)` man pages — the `$id$salt$hash` storage format used by Unix systems.
