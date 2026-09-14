# Lab 04: Asymmetric Encryption and Key Exchange with OpenSSH / RSA / ECC

## Objectives

By the end of this lab, you will be able to:

1. List the public-key algorithms and key-exchange methods that OpenSSH offers, and read a key-exchange name (e.g. `ecdh-sha2-nistp256`) as algorithm + curve/group + hash.
2. Explain the difference between asymmetric *encryption* and asymmetric *key agreement*, and why raw asymmetric algorithms are not used to encrypt bulk data directly.
3. Generate key pair.
4. Use asymmetric cryptography to establish a shared AES key and then use that AES key to encrypt a message with a block cipher mode of operation (reusing the AES modes from Lab 03).


## Prerequisites

- Completion of Lab 03 (Symmetric Encryption with OpenSSH / AES), this lab builds directly on the `openssl enc` and AES-mode commands used there.
- SSH access to your assigned course VM.


## Before you start:

### Know there are two different problems

- **Symmetric encryption (Lab 03)** needs both parties to already share one secret key. The open problem is: how do two parties who have never met agree on that secret over an insecure channel?
- **Asymmetric (public-key) cryptography** solves that problem two different ways:
  - **Encryption (e.g., RSA and ECC):** anyone can encrypt with your *public* key, but only your *private* key can decrypt. You can use this to send someone a symmetric key securely.
  - **Key agreement (e.g., ECDH — Elliptic Curve Diffie-Hellman):** two parties each generate a key pair, exchange only their *public* keys, and each independently computes the same shared secret. Neither party ever transmits the secret itself.

In practice nobody encrypts bulk data (files, SSH sessions, TLS traffic) directly with RSA or ECC (**asymmetric operations are slow** and, for RSA, limited to messages smaller than the key size). Instead, real systems use a **hybrid scheme**: asymmetric crypto only to establish a symmetric AES key, then AES (with a mode of operation, as in Lab 03) to encrypt the actual data. This is exactly what a TLS or SSH handshake does, and it is what you will build in this lab.

### How OpenSSH uses this

OpenSSH does not use RSA encryption to exchange the session key. It uses a **key-exchange (KEX)** algorithm, almost always a *Diffie-Hellman* variant, to agree on a shared secret, then derives symmetric keys (the AES ciphers from Lab 03) from it. RSA/ECDSA/Ed25519 keys are used separately, for **authentication** (proving identity), not for encrypting the session.

### Set-up and Implementation

- On the VM, create a folder for this lab on the class folder on home directory.
    - create folder: `mkdir ~/comp301/lab04`
    - Go to folder `cd ~/comp301/lab04`

## Part 1: Key Generation

### Inspect the asymmetric algorithms OpenSSH offers

Connect to your VM and list the public-key types OpenSSH supports for authentication:

```bash
ssh -Q key
```

You should see entries such as `ssh-rsa`, `ecdsa-sha2-nistp256`, `ecdsa-sha2-nistp384`, `ecdsa-sha2-nistp521`, and `ssh-ed25519` (Ed25519 is a modern elliptic-curve signature scheme).

Now list the key-exchange algorithms OpenSSH offers to establish the session's shared secret:

```bash
ssh -Q kex
```

You should see entries such as: `curve25519-sha256`, `ecdh-sha2-nistp256`, `ecdh-sha2-nistp384`, `ecdh-sha2-nistp521`, and `diffie-hellman-group14-sha256`. 

> Read `ecdh-sha2-nistp256` as: ECDH key agreement, on the NIST P-256 curve, with SHA-256 used to hash the result into a session key.

Generate *one keypair* of each family with `ssh-keygen` and inspect the fingerprint and bit length:

```bash
ssh-keygen -t rsa -b 2048 -f rsa_id -N ""
ssh-keygen -t ecdsa -b 256 -f ecdsa_id -N ""
ssh-keygen -t ed25519 -f ed25519_id -N ""
```
Then
```bash
ssh-keygen -l -f rsa_id.pub
ssh-keygen -l -f ecdsa_id.pub
ssh-keygen -l -f ed25519_id.pub
```

These SSH keypairs are for your own inspection in this part only — they are used for SSH authentication, not for the encryption exercise below, so they are not part of your submission.

## Part 2: Asymmetric Encryption Using RSA Hybrid Encryption

1. Generate an RSA key pair with OpenSSL or use/rename the one from **Part 1**:

    ```bash
    openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out rsa_private.pem
    openssl pkey -in rsa_private.pem -pubout -out rsa_public.pem
    ```

2. Generate a random 256-bit AES key as raw bytes (this simulates the symmetric key you want to send to your counterpart):

    ```bash
    openssl rand 32 -out aes_key.bin
    ```

3. Encrypt `aes_key.bin` with the RSA **public** key, using OAEP padding (the modern, secure RSA padding scheme):

    ```bash
    openssl pkeyutl -encrypt \
    -pubin -inkey rsa_public.pem \
    -in aes_key.bin -out aes_key.enc \
    -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256
    ```

4. Only the holder of `rsa_private.pem` can recover the key. Prove it by decrypting with the **private** key:

    ```bash
    openssl pkeyutl -decrypt \
    -inkey rsa_private.pem \
    -in aes_key.enc -out aes_key_recovered.bin \
    -pkeyopt rsa_padding_mode:oaep -pkeyopt rsa_oaep_md:sha256
    ```

    Then, see the difference:
        ```bash
        diff aes_key.bin aes_key_recovered.bin
        ```

    > No output from `diff` confirms the recovered AES key matches the original exactly.

5. Now use the AES key you just exchanged to encrypt a message, choosing one block cipher mode of operation (as in Lab 03):

    * Generate the AES Key and IV
        ```bash
        KEY_HEX=$(xxd -p aes_key.bin | tr -d '\n')
        IV_HEX=$(openssl rand -hex 16)
        ```
    * Write a message to `plaintext.txt`

        ```bash
        echo "This is my COMP301 Lab 04 message (RSA track)." > plaintext.txt
        ```

    * Encrypt the message:

        ```bash
        openssl enc -aes-256-ctr -in plaintext.txt -out ciphertext.enc -K "$KEY_HEX" -iv "$IV_HEX"
        ```

6. Verify round trip
    ```bash
    openssl enc -aes-256-ctr -d -in ciphertext.enc -out recovered.txt -K "$KEY_HEX" -iv "$IV_HEX"
    diff plaintext.txt recovered.txt
    ```

7. Record `$KEY_HEX` and `$IV_HEX`, you will need them for your README.
    ```bash
    echo $KEY_HEX > aes_key.bin

    echo $IV_HEX > aes_iv.bin
    ```

## Part 3: ECC key exchange (ECDH)

This track simulates two parties (**Alice** and **Bob**) independently deriving the same shared secret, the way SSH's `ecdh-sha2-nistp256` key exchange works.

1. Generate an EC key pair for each party, on the NIST P-256 curve (`prime256v1`):

    ```bash
    openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:prime256v1 -out alice_private.pem

    openssl pkey -in alice_private.pem -pubout -out alice_public.pem

    openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:prime256v1 -out bob_private.pem

    openssl pkey -in bob_private.pem -pubout -out bob_public.pem
    ```

2. In a real exchange, only the `*_public.pem` files would cross the network. Each side computes the shared secret using their own private key and the other party's public key:

    ```bash
    openssl pkeyutl -derive -inkey alice_private.pem -peerkey bob_public.pem -out alice_shared.bin

    openssl pkeyutl -derive -inkey bob_private.pem -peerkey alice_public.pem -out bob_shared.bin
    ```
    You can check the difference of the shared key
    ```bash
    diff alice_shared.bin bob_shared.bin
    ```

    > No output from `diff` confirms both parties arrived at the identical shared secret without either one ever transmitting it.

3. A raw ECDH shared secret should not be used directly as an AES key. You need to hash it first (this is what the `-sha256` in `ecdh-sha2-nistp256` refers to) to produce a well-distributed key of the right length:

    ```bash
    # AES Key
    KEY_HEX=$(openssl dgst -sha256 -binary alice_shared.bin | xxd -p | tr -d '\n')
    
    # AES IV
    IV_HEX=$(openssl rand -hex 16)
    
    # Message to send
    echo "This is my COMP301 Lab 04 message (ECC track)." > plaintext_ecc.txt
    
    # Encrypt the message
    openssl enc -aes-256-ctr -in plaintext_ecc.txt -out ciphertext_ecc.enc -K "$KEY_HEX" -iv "$IV_HEX"

    # verify round trip
    openssl enc -aes-256-ctr -d -in ciphertext_ecc.enc -out recovered_ecc.txt -K "$KEY_HEX" -iv "$IV_HEX"
    
    # Check the difference
    diff plaintext_ecc.txt recovered_ecc.txt
    ```

6. Record `$KEY_HEX` and `$IV_HEX` — you will need them for your README.
    ```bash
    echo $KEY_HEX > aes_key_ecc.bin
    echo $IV_HEX > aes_iv_ecc.bin
    ```

## Assignment

1. Complete Parts 1, 2, and 3.
6. Write a `README.md` stating:
   - What have you completed in parts 1, 2, and 3.
   - Write the files names associated with each part.

## Submission

Create the submission folder on the VM and make sure that your deliverables are there. It should be in `~/comp301/lab04/`.


## Useful References and Resources

- `man ssh_config` — see the `KexAlgorithms` and `PubkeyAcceptedAlgorithms` keywords.
- `man openssl-pkeyutl` — RSA encrypt/decrypt and ECDH derive operations.
- `man openssl-genpkey` — key generation for RSA and EC.
- NIST SP 800-56A — *Recommendation for Pair-Wise Key-Establishment Schemes Using Discrete Logarithm Cryptography* (Diffie-Hellman/ECDH).
- RFC 5656 — *Elliptic Curve Algorithm Integration in the Secure Shell Transport Layer* (defines `ecdh-sha2-nistp*` for SSH).
