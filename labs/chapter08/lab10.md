# Lab 10: Cross-Site Scripting

## Objectives

By the end of this lab, you will be able to:

1. Trigger reflected, stored, and DOM-based cross-site scripting against a local web application, and identify each type from the response.
2. Show how an injected script reaches session data such as cookies.
3. Fix the application with output encoding and a Content Security Policy header, and verify the same payloads render as inert text.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required.
- Python 3 with the standard library only. No packages to install.
- Chapter 8 notes, Section 1: cross-site scripting, the three XSS types, and the four mitigations.

## Before you start

### Background

Cross-site scripting lets an attacker inject a client-side script into pages a server delivers. The attack fires when a browser downloads the page and executes the script, and the script can then read session tokens and cookies. The three types differ in where the script lives: reflected XSS rides the response to the victim's own request, stored XSS sits in the server's database and reaches every visitor, and DOM-based XSS stays inside the browser and never travels to the server.

| Concept from the lecture | Where you observe the concept in this lab |
| --- | --- |
| Reflected XSS | Part 2: the `/search` route reflects your input unescaped |
| Stored XSS | Part 3: a comment carries a script to every visitor of `/` |
| DOM-based XSS | Part 4: the `/dom` page writes `location.hash` through `innerHTML` |
| Session token theft | Part 3: the payload reads `document.cookie` |
| Output encoding | Part 5: `html.escape` renders the payload as text |
| Content Security Policy | Part 5: the `/fixed` routes send a CSP header |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab10`
  - go to folder: `cd ~/comp301/lab10`

## Part 1: Setup and a baseline page

Write the guestbook application. The application is deliberately vulnerable:

```bash
cat > app.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 10 guestbook. Deliberately vulnerable to XSS on /, /search, and /dom.
The /fixed and /fixed-search routes escape output and send a CSP header."""
import html
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse, parse_qs

COMMENTS = []

PAGE = """<!doctype html>
<html><body>
<h1>Guestbook</h1>
<form method="POST" action="/add">
  <input name="c"><button>Sign</button>
</form>
<h2>Comments</h2>
{comments}
<h2>Search</h2>
<form method="GET" action="/search">
  <input name="q"><button>Search</button>
</form>
<p>DOM demo: <a href="/dom">/dom</a></p>
<p>Fixed version: <a href="/fixed">/fixed</a></p>
</body></html>"""

DOM_PAGE = """<!doctype html>
<html><body>
<h1>DOM demo</h1>
<div id="out"></div>
<script>
  document.getElementById("out").innerHTML = location.hash.slice(1);
</script>
</body></html>"""


class H(BaseHTTPRequestHandler):
    def _send(self, body, csp=False):
        data = body.encode()
        self.send_response(200)
        if csp:
            self.send_header("Content-Security-Policy", "default-src 'self'")
        self.send_header("Content-Type", "text/html")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

    def do_GET(self):
        u = urlparse(self.path)
        q = parse_qs(u.query)
        if u.path == "/":
            items = "".join(f"<li>{c}</li>" for c in COMMENTS)
            self._send(PAGE.format(comments="<ul>" + items + "</ul>"))
        elif u.path == "/search":
            term = q.get("q", [""])[0]
            self._send(f"<p>You searched for: {term}</p>")
        elif u.path == "/dom":
            self._send(DOM_PAGE)
        elif u.path == "/fixed":
            items = "".join(f"<li>{html.escape(c)}</li>" for c in COMMENTS)
            self._send(PAGE.format(comments="<ul>" + items + "</ul>"), csp=True)
        elif u.path == "/fixed-search":
            term = html.escape(q.get("q", [""])[0])
            self._send(f"<p>You searched for: {term}</p>", csp=True)
        else:
            self.send_error(404)

    def do_POST(self):
        u = urlparse(self.path)
        if u.path == "/add":
            n = int(self.headers.get("Content-Length", 0))
            data = parse_qs(self.rfile.read(n).decode())
            COMMENTS.append(data.get("c", [""])[0])
            self.send_response(303)
            self.send_header("Location", "/")
            self.end_headers()
        else:
            self.send_error(404)

    def log_message(self, *a):
        pass


print("Listening on http://127.0.0.1:8100")
HTTPServer(("127.0.0.1", 8100), H).serve_forever()
PYEOF
```

Start the server and leave the server running:

```bash
python3 app.py
```

Open a second SSH session to the VM for the remaining commands, and work from `~/comp301/lab10`. Send a baseline request:

```bash
curl -s http://127.0.0.1:8100/ | head -5
```

The response is the guestbook HTML page.

## Part 2: Reflected XSS

Send a search request with a script as the search term:

```bash
curl -s 'http://127.0.0.1:8100/search?q=<script>alert(1)</script>'
```

The response contains your script verbatim:

```html
<p>You searched for: <script>alert(1)</script></p>
```

A browser rendering this response executes the script. The script traveled with the victim's own request and came back in the response, which is the reflected XSS pattern from the notes. In a real attack, the victim clicks a crafted link, often a shortened URL or a link in a phishing email.

Record the response line for the Assignment.

## Part 3: Stored XSS and session data

Sign the guestbook with a script that reads the cookie:

```bash
curl -s -X POST -d 'c=<script>alert(document.cookie)</script>' http://127.0.0.1:8100/add -o /dev/null -w "%{http_code}\n"
curl -s http://127.0.0.1:8100/ | grep script
```

The script now lives in the server's comment list. Every visitor who loads `/` receives the payload, with no crafted link required. In a browser, `alert(document.cookie)` would display the visitor's session cookie, which is the session token theft described in the notes.

Count how many script tags the stored page serves:

```bash
curl -s http://127.0.0.1:8100/ | grep -c '<script>alert'
```

## Part 4: DOM-based XSS

Fetch the DOM demo page and read the source:

```bash
curl -s http://127.0.0.1:8100/dom
```

The page reads `location.hash` and writes the value through `innerHTML`. A URL such as `http://127.0.0.1:8100/dom#<img src=x onerror=alert(1)>` injects markup entirely inside the browser. The server never receives the fragment, because browsers never send the part after `#`. Record the line of source that performs the dangerous write.

## Part 5: Output encoding and CSP

Send the same reflected payload to the fixed route:

```bash
curl -s 'http://127.0.0.1:8100/fixed-search?q=<script>alert(1)</script>'
```

The angle brackets arrive encoded as `&lt;` and `&gt;`, so a browser renders the payload as text instead of executing the payload. This is the output encoding mitigation, performed here by `html.escape`, the same job `htmlspecialchars()` performs in PHP.

Check the response headers on the fixed page:

```bash
curl -sI http://127.0.0.1:8100/fixed | grep -i content-security
```

The `Content-Security-Policy: default-src 'self'` header tells the browser to run only scripts from the application's own origin, so injected inline scripts cannot execute even if a payload slips past encoding.

Confirm the stored payload is also neutralized on the fixed page:

```bash
curl -s http://127.0.0.1:8100/fixed | grep -c '&lt;script&gt;'
```

Stop the server with `Ctrl+C` in the first terminal when you finish.

## Part 6: Analysis

Answer in one sentence each, from what you observed:

1. Why does stored XSS reach more victims than reflected XSS?
2. Which defense in Part 5 changes the markup, and which defense changes browser behavior?
3. Why does the DOM-based payload never appear in the server log?

## Assignment

Create a `submission.txt` file with the following entries:

- `reflected-response: <the full response line from the vulnerable /search>`
- `stored-script-count: <output of the grep -c command in Part 3>`
- `dom-sink-line: <the source line from /dom that writes to the page>`
- `fixed-search-response: <the full response line from /fixed-search>`
- `csp-header: <the Content-Security-Policy header value>`
- `fixed-script-count: <output of the grep -c command in Part 5>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

> **\[NOTE\]**
>
> You can write a field directly with `echo`:
>
> ```bash
> echo "stored-script-count: $(curl -s http://127.0.0.1:8100/ | grep -c '<script>alert')" >> submission.txt
> ```
>
> The `>>` operator appends, so run each command once and check your work with `cat submission.txt` as you go.

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab10`) and name the file exactly `submission.txt`.

## Useful References and Resources

- OWASP, "Cross Site Scripting Prevention Cheat Sheet": output encoding and CSP defenses.
- OWASP, "DOM based XSS Prevention Cheat Sheet": safe alternatives to `innerHTML`.
- CWE-79, "Improper Neutralization of Input During Web Page Generation (Cross-site Scripting)."
- Python documentation for `html.escape` and `http.server`.

## Optional Challenges

- Add a route that stores the payload and exfiltrates the cookie to a listener you run with `nc -l 8102`, and describe what a real attacker receives.
- Modify the DOM page to use `textContent` instead of `innerHTML` and confirm the payload no longer executes.
