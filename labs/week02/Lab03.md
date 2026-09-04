# Lab 03: Symmetric Encryption and Keys with OpenSSH / AES

## Objectives

By the end of this lab, you will be able to:

1. List the symmetric ciphers that OpenSSH offers for securing a session, and read an OpenSSH cipher name (e.g. `aes128-ctr`) as algorithm + key size + mode of operation.
2. Explain what a block cipher mode of operation is and describe the difference between ECB, CBC, CFB, OFB, and CTR.
3. Generate a symmetric AES key and, where required, an initialization vector (IV).
4. Encrypt and decrypt a file from the command line using AES in a chosen mode, and verify that decryption recovers the original plaintext.
5. Package a plaintext file, its encrypted counterpart, and a README describing the method and key into a submission folder.

## Prerequisites

- `openssl` must be available on the VM (verify with `openssl version`; version 1.1.1 or later is expected).
- No prior cryptography experience is assumed beyond what was covered in Lesson 5 (Advanced Encryption Standards: symmetric ciphers and block cipher modes).

## Background

### Symmetric encryption

Symmetric encryption uses the **same key** to encrypt and decrypt data. AES (Advanced Encryption Standard) is the standard symmetric block cipher in use today. AES always encrypts data in fixed-size **128-bit (16-byte) blocks**, regardless of the key size (128, 192, or 256 bits).

### Why "mode of operation" matters

A block cipher only defines how to scramble a single 16-byte block. A **mode of operation** defines how to chain many blocks together to encrypt a message of arbitrary length. The mode you choose affects security properties (does an identical plaintext block always produce an identical ciphertext block?), whether an IV is required, and whether encryption can be parallelized.

| Mode | Name | IV required? | Notes |
|---|---|---|---|
| ECB | Electronic Codebook | No | Encrypts each block independently. Identical plaintext blocks produce identical ciphertext blocks — patterns leak through. **Never use for real data.** |
| CBC | Cipher Block Chaining | Yes | Each plaintext block is XORed with the previous ciphertext block before encryption. Sequential — cannot be parallelized. |
| CFB | Cipher Feedback | Yes | Turns the block cipher into a self-synchronizing stream cipher. |
| OFB | Output Feedback | Yes | Turns the block cipher into a synchronous stream cipher; the keystream is independent of the plaintext. |
| CTR | Counter | Yes (used as a nonce/counter) | Turns the block cipher into a stream cipher by encrypting a counter value. Fully parallelizable, a common high-performance choice. |

### Reading an OpenSSH cipher name

OpenSSH names its symmetric ciphers as `<algorithm><key-size>-<mode>`. For example:

- `aes128-ctr` — AES, 128-bit key, CTR mode.
- `aes256-cbc` — AES, 256-bit key, CBC mode.

These are the exact same AES algorithm and modes described above — OpenSSH uses them to encrypt the data channel of every SSH session. In this lab you will first inspect which of these OpenSSH offers, then use `openssl enc`, which accepts the equivalent hyphenated cipher names (e.g. `aes-128-ctr`), to encrypt a file directly so you can inspect and submit the resulting ciphertext.

## Part 1 — Inspect the ciphers OpenSSH offers

Connect to your VM, then run:

```bash
ssh -Q cipher
```

This lists every symmetric cipher your local OpenSSH client can negotiate for the session, including entries such as `aes128-ctr`, `aes192-ctr`, `aes256-ctr`, `aes128-cbc`, `aes256-cbc`, and `aes256-gcm@openssh.com`.

Also check which ciphers your server is currently configured to accept:

```bash
sshd -T 2>/dev/null | grep -i ciphers
```

(If `sshd -T` requires elevated privileges on your VM, you may instead read the `Ciphers` line in `/etc/ssh/sshd_config`, if present.)

You can force a specific cipher when connecting to another host with the `-c` flag, for example:

```bash
ssh -c aes128-ctr user@remotehost
```

Record in your own notes (not required for submission) which mode-based cipher names you saw, and identify at least three that use CTR mode and three that use CBC mode.

## Part 2 — Generate an AES key and IV

You will encrypt a file with `openssl enc`. Two pieces of secret material are needed:

- **Key** — the actual symmetric secret. For AES-128 this is 16 bytes (32 hex characters); for AES-256 this is 32 bytes (64 hex characters).
- **IV (initialization vector)** — a 16-byte (32 hex character) value required by CBC, CFB, OFB, and CTR modes. It does not need to be secret, but it must never be reused with the same key.

Generate both as random hex values:

```bash
openssl rand -hex 16   # 128-bit AES key
openssl rand -hex 16   # 16-byte IV (all AES modes use a 16-byte block/IV size)
```

Save the output of each command — you will need to write both values into your README in Part 4.

## Part 3 — Encrypt and decrypt a file

The general form of the command is:

```bash
openssl enc -<cipher> -in <plaintext-file> -out <encrypted-file> -K <key-in-hex> -iv <iv-in-hex>
```

Below is one command per mode of operation, all using AES-128, so you can see how only the mode changes:

```bash
# ECB — no IV needed; shown for comparison only, do not use for the assignment
openssl enc -aes-128-ecb -in plaintext.txt -out ciphertext_ecb.enc -K <KEY>

# CBC
openssl enc -aes-128-cbc -in plaintext.txt -out ciphertext_cbc.enc -K <KEY> -iv <IV>

# CFB
openssl enc -aes-128-cfb -in plaintext.txt -out ciphertext_cfb.enc -K <KEY> -iv <IV>

# OFB
openssl enc -aes-128-ofb -in plaintext.txt -out ciphertext_ofb.enc -K <KEY> -iv <IV>

# CTR
openssl enc -aes-128-ctr -in plaintext.txt -out ciphertext_ctr.enc -K <KEY> -iv <IV>
```

Replace `<KEY>` and `<IV>` with the hex values you generated in Part 2. You are free to use AES-192 or AES-256 instead of AES-128 by changing the cipher name (e.g. `-aes-256-ctr`) and generating a longer key with `openssl rand -hex 32`.

To decrypt and confirm you recover the original message, add the `-d` flag and swap `-in`/`-out`:

```bash
openssl enc -aes-128-ctr -d -in ciphertext_ctr.enc -out recovered.txt -K <KEY> -iv <IV>
diff plaintext.txt recovered.txt
```

If `diff` produces no output, the decrypted file is identical to the original — decryption succeeded. Confirm this before moving on to the assignment.

## Assignment

1. Choose **one** AES mode of operation from Part 3 (CBC, CFB, OFB, or CTR — do not use ECB for this assignment).
2. Write a short original message of your choice into a plaintext file named `plaintext.txt`, for example:

   ```bash
   echo "This is my COMP301 Lab 03 message." > plaintext.txt
   ```

3. Generate a key (and IV, if your chosen mode requires one) as shown in Part 2.
4. Encrypt `plaintext.txt` with your chosen cipher, producing `ciphertext.enc`.
5. Decrypt `ciphertext.enc` back to a separate file and run `diff` against `plaintext.txt` to prove the round trip works. This verification step is for you to confirm correctness — do not submit the recovered file.
6. Create a `README.md` that states:
   - The exact cipher name you used (e.g. `aes-128-ctr`).
   - The key, in hex, that you used.
   - The IV, in hex, if your mode required one (write "not applicable" if not).
   - The exact `openssl enc` command you ran to produce `ciphertext.enc`.

## Submission

On the VM, create the submission folder in your home directory and place your three files in it:

```bash
mkdir -p ~/comp301/lab03
cp plaintext.txt ciphertext.enc README.md ~/comp301/Lab03/
```

Your submission folder `~/comp301/Lab03/` must contain exactly:

- `plaintext.txt` — your original message.
- `ciphertext.enc` — the AES-encrypted version of `plaintext.txt`.
- `README.md` — the cipher name, key, IV (if applicable), and the command used.

## Verification / Completion Criteria

- [ ] `ssh -Q cipher` was run and at least three CTR-mode and three CBC-mode OpenSSH cipher names were identified.
- [ ] A key (and IV, if required by the chosen mode) was generated with `openssl rand -hex`.
- [ ] `plaintext.txt` was encrypted into `ciphertext.enc` using one non-ECB AES mode via `openssl enc`.
- [ ] Decryption of `ciphertext.enc` was verified locally with `diff` to match `plaintext.txt` exactly.
- [ ] `~/comp301/Lab03/` on the VM contains `plaintext.txt`, `ciphertext.enc`, and `README.md`, and the README correctly states the cipher, key, and IV (if applicable) used.

## Useful References and Resources

- `man ssh_config` — see the `Ciphers` keyword for the full list of supported names and syntax.
- `man openssl-enc` (or `openssl enc -h` on older systems) — full list of supported ciphers and flags.
- NIST SP 800-38A — *Recommendation for Block Cipher Modes of Operation* (defines ECB, CBC, CFB, OFB, CTR).
