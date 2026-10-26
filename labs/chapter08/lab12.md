# Lab 12: SQL Injection and Directory Traversal

## Objectives

By the end of this lab, you will be able to:

1. Bypass a login form with a crafted SQL fragment and explain the SQL statement the application built.
2. Extract a password table through a UNION injection against a search route.
3. Stop both attacks with bound parameters, and verify the payloads fail.
4. Read a file outside the web root with a dot-dot-slash payload, and block the traversal with path validation.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required.
- Python 3 with the standard library only. SQLite ships with Python.
- Chapter 8 notes, Section 3: injection attacks and SQL injection. Chapter 8 notes, Section 4: directory traversal.

## Before you start

### Background

An injection attack inserts malicious input into a program or database query, exploiting weak input validation. SQL injection turns a user input field into part of the SQL grammar. Directory traversal walks out of the web root with `../` sequences until the path reaches a file the server should never serve.

| Concept from the lecture | Where you observe the concept in this lab |
| --- | --- |
| Crafted input forms a valid SQL statement | Part 2: `alice'--` comments out the password check |
| Injection reads other tables | Part 3: a UNION query dumps the users table |
| Input validation and parameterized queries | Part 4: `/login2` and `/search2` treat input as data |
| Dot-dot-slash traversal | Part 5: `/dl?f=../../../../etc/passwd` reads the password file |
| Restricting access to the web root | Part 5: `/dl2` rejects resolved paths outside the root |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab12`
  - go to folder: `cd ~/comp301/lab12`

## Part 1: The vulnerable shop

Write the shop application:

```bash
cat > sqli.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 12 shop. /login and /search build SQL with f-strings (vulnerable).
/login2 and /search2 use bound parameters (fixed)."""
import sqlite3
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

db = sqlite3.connect(":memory:", check_same_thread=False)
db.execute("CREATE TABLE users (name TEXT, pw TEXT)")
db.execute("INSERT INTO users VALUES ('alice', 's3cret-alice'), ('bob', 'b0b-pass')")
db.execute("CREATE TABLE products (name TEXT, price INTEGER)")
db.execute("INSERT INTO products VALUES ('keyboard', 25), ('mouse', 15)")


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
        if u.path == "/login":
            name = q.get("u", [""])[0]
            pw = q.get("p", [""])[0]
            sql = f"SELECT * FROM users WHERE name='{name}' AND pw='{pw}'"
            rows = db.execute(sql).fetchall()
            self._send("welcome\n" if rows else "denied\n")
        elif u.path == "/login2":
            name = q.get("u", [""])[0]
            pw = q.get("p", [""])[0]
            rows = db.execute("SELECT * FROM users WHERE name=? AND pw=?",
                              (name, pw)).fetchall()
            self._send("welcome\n" if rows else "denied\n")
        elif u.path == "/search":
            term = q.get("q", [""])[0]
            sql = f"SELECT name, price FROM products WHERE name LIKE '%{term}%'"
            try:
                rows = db.execute(sql).fetchall()
                self._send("".join(f"{r}\n" for r in rows))
            except Exception as e:
                self._send(f"error: {e}\n", 400)
        elif u.path == "/search2":
            term = q.get("q", [""])[0]
            rows = db.execute("SELECT name, price FROM products WHERE name LIKE ?",
                              ("%" + term + "%",)).fetchall()
            self._send("".join(f"{r}\n" for r in rows))
        else:
            self.send_error(404)

    def log_message(self, *a):
        pass


print("Listening on http://127.0.0.1:8300")
HTTPServer(("127.0.0.1", 8300), H).serve_forever()
PYEOF
```

Start the shop and leave the server running, then work from a second SSH session:

```bash
python3 sqli.py
```

Check the baseline behavior:

```bash
curl -s 'http://127.0.0.1:8300/login?u=alice&p=s3cret-alice'
curl -s 'http://127.0.0.1:8300/login?u=alice&p=wrong'
curl -s 'http://127.0.0.1:8300/search?q=key'
```

The correct password answers `welcome`, the wrong password answers `denied`, and the search returns the keyboard row.

## Part 2: Login bypass

The `/login` route builds the statement with an f-string:

```sql
SELECT * FROM users WHERE name='{name}' AND pw='{pw}'
```

Send a name that closes the quote and comments out the rest of the statement:

```bash
curl -s "http://127.0.0.1:8300/login?u=alice%27--&p=x"
```

The `%27` is a URL-encoded single quote. The database receives:

```sql
SELECT * FROM users WHERE name='alice'--' AND pw='x'
```

Everything after `--` is a comment, so the password check disappears and alice's row matches. The answer is `welcome` even though the password `x` is wrong. Try the classic disjunction as well:

```bash
curl -s "http://127.0.0.1:8300/login?u=%27%20OR%20%271%27=%271&p=x"
```

Record both responses.

## Part 3: UNION extraction

The `/search` route runs:

```sql
SELECT name, price FROM products WHERE name LIKE '%{term}%'
```

A UNION query appends a second result set when the column counts match. The products query returns two columns, so the injected query must return two columns:

```bash
curl -s "http://127.0.0.1:8300/search?q=%27%20UNION%20SELECT%20name,%20pw%20FROM%20users--"
```

The response lists the products and then both users with their passwords. A search box became a database dump. Record the leaked password for alice.

## Part 4: Bound parameters

Send the same payloads to the fixed routes:

```bash
curl -s "http://127.0.0.1:8300/login2?u=alice%27--&p=x"
curl -s "http://127.0.0.1:8300/search2?q=%27%20UNION%20SELECT%20name,%20pw%20FROM%20users--"
```

Both fail: the login is denied, and the search returns an empty list. Bound parameters keep user input out of the SQL grammar entirely, because the database receives the statement structure and the data separately. Record both responses.

## Part 5: Directory traversal

Write the file server in the first terminal after stopping the shop with `Ctrl+C`, or use a third terminal:

```bash
cat > files.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 12 file server. /dl joins user input to the root directory (vulnerable
to directory traversal). /dl2 rejects anything that resolves outside ROOT."""
import os
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

os.makedirs("files", exist_ok=True)
with open("files/readme.txt", "w") as fh:
    fh.write("public readme\n")
ROOT = os.path.realpath("files")


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
        name = q.get("f", [""])[0]
        if u.path == "/dl":
            try:
                with open(os.path.join(ROOT, name)) as fh:
                    self._send(fh.read(2000))
            except Exception as e:
                self._send(f"error: {e}\n", 400)
        elif u.path == "/dl2":
            target = os.path.realpath(os.path.join(ROOT, name))
            if not target.startswith(ROOT + os.sep):
                self._send("outside the root directory\n", 403)
                return
            try:
                with open(target) as fh:
                    self._send(fh.read(2000))
            except Exception as e:
                self._send(f"error: {e}\n", 400)
        else:
            self.send_error(404)

    def log_message(self, *a):
        pass


print("Listening on http://127.0.0.1:8301, serving", ROOT)
HTTPServer(("127.0.0.1", 8301), H).serve_forever()
PYEOF

python3 files.py
```

Serve the intended file, then walk out of the root:

```bash
curl -s 'http://127.0.0.1:8301/dl?f=readme.txt'
curl -s 'http://127.0.0.1:8301/dl?f=../../../../etc/passwd' | head -3
```

The second request returns the start of `/etc/passwd`, beginning with the root account line on Linux. Each `../` climbs one directory, and the server joins the result to the web root without checking where the path lands. The IIS root directory `C:\Inetpub\wwwroot` from the notes faces the same risk on Windows servers.

Apply the fixed route:

```bash
curl -s 'http://127.0.0.1:8301/dl2?f=../../../../etc/passwd'
curl -s 'http://127.0.0.1:8301/dl2?f=readme.txt'
```

The fixed route resolves the real path and refuses anything outside the root, while legitimate files still download.

Stop every server with `Ctrl+C` when you finish.

## Part 6: Analysis

Answer in one sentence each, from what you observed:

1. In Part 2, why does the password value `x` never matter?
2. In Part 3, why did the injected query need exactly two columns?
3. In Part 5, what does `realpath` compute that `os.path.join` alone does not?

## Assignment

Create a `submission.txt` file with the following entries:

- `bypass-comment: <response for the comment payload against /login>`
- `bypass-or: <response for the OR 1=1 payload>`
- `union-alice: <the leaked row for alice from Part 3>`
- `fixed-login2: <response for the bypass payload against /login2>`
- `fixed-search2-rows: <number of rows returned by the UNION payload against /search2>`
- `traversal-first-line: <the first line returned by the /etc/passwd payload>`
- `fixed-dl2: <response body for the payload against /dl2>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab12`) and name the file exactly `submission.txt`.

## Useful References and Resources

- OWASP, "SQL Injection Prevention Cheat Sheet": parameterized queries and input validation.
- OWASP, "Path Traversal" attack page and CWE-22, "Improper Limitation of a Pathname to a Restricted Directory."
- CWE-89, "Improper Neutralization of Special Elements used in an SQL Command (SQL Injection)."
- SQLite documentation for parameter substitution in `execute`.

## Optional Challenges

- Write a payload that reads the SQLite schema through the UNION route by selecting from `sqlite_master`.
- Add a third defense layer to `/dl2` that also rejects absolute paths and symlink escapes, and test all three bypass styles against your code.
