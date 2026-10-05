<!-- lab-latest.md mirrors lab-v1.md -->

# 05 — Trust PK Infrastructure — Lab


## Objectives
By the end of this lab, you will be able to:

1. Diagram the components of a PKI and state the purpose of each role by inspecting a live certificate's fields.
2. Trace a certificate chain from a leaf certificate to its root by recording each issuer→subject link.
3. Locate the CRL Distribution Points and OCSP responder fields in a certificate and choose CRL or OCSP for a stated revocation-checking requirement with justification.

## Prerequisites
- A Unix-like shell (macOS Terminal or Linux) with `openssl` available (OpenSSL 1.1 or newer recommended).
- Network access to fetch an HTTPS site's certificate chain.
- Optional: a modern browser with a certificate viewer for a visual inspection.
- Prior lessons:  (asymmetric key primitives and public-key primitives).

## Before you start

### Background

This lab turns the lecture concepts into observable artifacts by fetching a live X.509 certificate chain and reading the certificate fields. The walkthrough uses `openssl s_client` to retrieve the chain and `openssl x509` to display certificate fields such as subject, issuer, serial, validity dates, CRL Distribution Points, and Authority Information Access (OCSP).

### Set-up and Implementation

- On the VM, create a folder for this lab on the class folder on home directory.
    - create folder: `mkdir ~/comp301/lab05`
    - Go to folder `cd ~/comp301/lab05`

## Part 1 — Setup 
Open a terminal. Use a temporary directory for artifacts. Set a target host to inspect. Replace `TARGET` with your chosen HTTPS host if desired.

```sh
TARGET=www.wikipedia.org
TMPDIR=$(mktemp -d)
echo "using tmpdir: $TMPDIR"
```

## Part 2 — Fetch and save the chain
Retrieve the certificate chain from the target and save it to a file.

```sh
echo | openssl s_client -connect $TARGET:443 -servername $TARGET -showcerts 2>/dev/null > $TMPDIR/chain.txt
```

Split the chain into individual PEM files.

```sh
python3 - "$TMPDIR" <<'PY'
import sys
t = sys.argv[1]
c = 0
fname = None
for line in open(t+"/chain.txt"):
    if "-----BEGIN CERTIFICATE-----" in line:
        c += 1
        fname = f"{t}/cert-{c:02d}.pem"
    if fname:
        open(fname,"a").write(line)
PY

ls -l $TMPDIR
```

## Part 3 — Read the leaf certificate 

Display the leaf certificate's key fields and record them.

```sh
openssl x509 -in $TMPDIR/cert-01.pem -noout -subject -issuer -serial -startdate -enddate

openssl x509 -in $TMPDIR/cert-01.pem -noout -text | sed -n '1,200p'
```

Record the `subject`, `issuer`, `serial`, `notBefore`, `notAfter`, and the public-key algorithm shown in the output.

## Part 4 — Trace the chain

List each certificate in the chain and record the subject→issuer link for each.

```sh
for f in $TMPDIR/cert-*.pem; do
  echo "---- $f ----"
  openssl x509 -in $f -noout -subject -issuer
done
```

Confirm that the final certificate in the chain is self-signed (subject equals issuer) for a root CA.

## Part 5 — Revocation check 

Locate CRL Distribution Points and the OCSP responder in the leaf certificate.

```sh
openssl x509 -in $TMPDIR/cert-01.pem -noout -text | egrep -n "CRL Distribution Points|Authority Information Access" -n || true
```

Pick which mechanism a client should use given one of these scenarios: offline client with intermittent network, or always-online client requiring fresh status. State your choice and one reason.

## Assignment
- Create a `submission.txt` file with the following entries: `leaf-subject: <value>`, `leaf-issuer: <value>`, `public-key-alg: <value>`, `serial: <value>`, `valid-from: <value>`, `valid-until: <value>`, `chain.txt: <list of issuer->subject lines>`, `crl: <CRL URL or none>`, `ocsp: <OCSP URL or none>`, `revocation-choice: <CRL or OCSP>`, `revocation-justify: <one-sentence justification>`.

## Submission
- Place `submission.txt` in this lab folder (`~/comp301/lab05`) and name the file exactly `submission.txt`.

## Verification / Completion Criteria

- `openssl x509 -in $TMPDIR/cert-01.pem -noout -subject` returns the recorded subject.

- `openssl x509 -in $TMPDIR/cert-01.pem -noout -issuer` returns the recorded issuer.

- The `for f in $TMPDIR/cert-*.pem; do openssl x509 -in $f -noout -subject -issuer; done` command shows a chain of issuer→subject links that ends with a self-signed root.
- `openssl x509 -in $TMPDIR/cert-01.pem -noout -text` includes CRL Distribution Points or Authority Information Access entries, and these values appear in `submission.txt`.
- `submission.txt` contains the `revocation-choice` and a one-sentence `revocation-justify` that references the tradeoff (freshness vs availability) for the chosen mechanism.

## Useful References and Resources
- RFC 5280 — Internet X.509 Public Key Infrastructure Certificate and CRL Profile.
- OpenSSL `s_client` and `x509` man pages for certificate retrieval and display.
- Let's Encrypt documentation on certificate chains and chain building.
- NIST SP 800-57 — Recommendation for Key Management.
