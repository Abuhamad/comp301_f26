# Chapter 03: Cryptography and Asymmetric Encryption

## Abstract

This chapter covers asymmetric encryption, where mathematically related public and private key pairs replace shared secret keys, and the RSA, ElGamal, and ECC algorithms that implement it. The chapter explains Diffie-Hellman key establishment and perfect forward secrecy, then turns to the primitives that protect integrity and authenticity: cryptographic hash functions, message authentication codes, and digital signatures. The chapter then shows how digital certificates and Public Key Infrastructure bind public keys to identities, and the obfuscation methods of steganography and tokenization.

## Objectives

- Explain asymmetric encryption with public and private key pairs, one-way and trapdoor one-way functions, the RSA, ElGamal, and ECC algorithms, and Diffie-Hellman key establishment with perfect forward secrecy.
- Describe cryptographic hash functions, their three resistance properties, the avalanche effect, and collision and birthday attacks.
- Explain how a message authentication code verifies data integrity and authenticity and how HMAC combines a hash function with a symmetric key.
- Explain digital signatures, the RSA, DSA, and ECDSA signature algorithms, and the three security services that signatures provide.
- Describe digital certificates, the certificate types, and the X.509 certificate formats.
- Describe the five PKI components, the four-stage certificate life cycle, and the single CA, hierarchical, and bridge trust models.
- Distinguish steganography from tokenization and describe the role of a token vault.

## Content

### 1. Asymmetric Encryption

Asymmetric encryption, also called public-key encryption, uses two different but mathematically related keys to encrypt and decrypt a message. The two keys form a key pair: the public key is openly available, and the private key stays secret and belongs only to the key-pair owner. A message encrypted with a public key can only be decrypted with the related private key, so a sender encrypts a message with the receiver's public key and the receiver decrypts the message with the matching private key.

#### **Asymmetric Algorithms**

Asymmetric encryption rests on one-way functions. A one-way function is easy to compute in one direction and hard to compute in the opposite direction. A trapdoor one-way function is easy to compute in both directions only with specific knowledge of an input or output, and that knowledge is called a trapdoor. Asymmetric algorithms build on two classes of trapdoor one-way functions: integer factorization and discrete logarithms.

Every asymmetric algorithm that supports encryption and signatures implements the same five functions, regardless of the underlying mathematics:

- **KeyGeneration:** Produces the public and private key pair.
- **Encryption:** Transforms a plaintext into a ciphertext with the public key.
- **Decryption:** Transforms a ciphertext back into the plaintext with the private key.
- **SignatureGeneration:** Produces a digital signature with the private key.
- **SignatureVerification:** Checks a digital signature with the public key.

Key-agreement algorithms such as Diffie-Hellman and ECDH are the exception: they establish shared keys only and implement none of the five functions directly. `Encryption` and `SignatureVerification` use the public key, while `Decryption` and `SignatureGeneration` use the private key. Three asymmetric algorithms are in use, and the algorithms differ in the mathematics behind the same five functions.

### **Asymmetric Encryption: RSA**

RSA uses the mathematical properties of prime numbers to generate key pairs. RSA security rests on the difficulty of factoring the product of two large prime numbers, typically a modulus several hundred digits long. RSA encryption and decryption are slow and computationally expensive, so RSA mostly serves to set up a secure communication link for exchanging a session key, a symmetric key used for one communication session only. RSA also serves in digital signatures.

```text
RSA KeyGeneration:
1. Choose two large random primes p and q.
2. Compute the modulus n = p × q.
3. Compute phi = (p - 1) × (q - 1).
4. Choose a public exponent e with 1 < e < phi and gcd(e, phi) = 1.
   (65537 is the common choice.)
5. Compute the private exponent d = e^-1 mod phi.
6. Public key: (n, e). Private key: (n, d).

RSA Encryption:
Input: public key (n, e), message m encoded as a number with 0 <= m < n.
1. Compute c = m^e mod n.
2. Output ciphertext c.

RSA Decryption:
Input: private key (n, d), ciphertext c.
1. Compute m = c^d mod n.
2. Output plaintext m.

RSA SignatureGeneration:
Input: private key (n, d), message m.
1. Compute the message hash h = H(m).
2. Compute s = h^d mod n.
3. Output signature s.

RSA SignatureVerification:
Input: public key (n, e), message m, signature s.
1. Compute h = H(m).
2. Compute h' = s^e mod n.
3. Accept the signature if h' = h, otherwise reject.
```

The pseudocode above shows textbook RSA. Real deployments pad messages before applying the exponent: OAEP for encryption and PSS for signatures. Unpadded RSA is deterministic and malleable, so standards such as PKCS #1 require padding.

### **Asymmetric Encryption: ElGamal**

ElGamal uses discrete logarithms to generate key pairs, and ElGamal security rests on the difficulty of solving discrete logarithms. ElGamal encryption is probabilistic, so one plaintext can encrypt to different ciphertexts. The main drawback is that the ciphertext is twice the length of the plaintext, which makes ElGamal inefficient on resource-limited devices and low-bandwidth networks. ElGamal also serves in digital signatures.

```text
ElGamal KeyGeneration:
1. Choose a large prime p.
2. Choose a generator g of the multiplicative group mod p.
3. Choose a random private key x with 1 < x < p - 1.
4. Compute y = g^x mod p.
5. Public key: (p, g, y). Private key: x.

ElGamal Encryption:
Input: public key (p, g, y), message m with 0 <= m < p.
1. Choose a fresh random k with 1 < k < p - 1.
   (A new k per message makes ElGamal probabilistic.)
2. Compute c1 = g^k mod p.
3. Compute c2 = m × y^k mod p.
4. Output ciphertext (c1, c2), twice the length of m.

ElGamal Decryption:
Input: private key x, ciphertext (c1, c2).
1. Compute s = c1^x mod p.  (Note: s = y^k mod p.)
2. Compute m = c2 × s^-1 mod p.
3. Output plaintext m.

ElGamal SignatureGeneration:
Input: private key x, message m.
1. Compute h = H(m).
2. Choose a random k with 1 < k < p - 1 and gcd(k, p - 1) = 1.
3. Compute r = g^k mod p.
4. Compute s = (h - x × r) × k^-1 mod (p - 1).
5. If s = 0, go back to step 2.
6. Output signature (r, s).

ElGamal SignatureVerification:
Input: public key (p, g, y), message m, signature (r, s).
1. Check that 0 < r < p. If not, reject.
2. Compute h = H(m).
3. Compute v1 = g^h mod p.
4. Compute v2 = y^r × r^s mod p.
5. Accept the signature if v1 = v2, otherwise reject.
```

### **Asymmetric Encryption: Elliptic Curve Cryptography (ECC)**

ECC uses elliptic curves, sets of points that satisfy a specific mathematical equation, to generate key pairs. ECC security rests on the difficulty of the Elliptic Curve Discrete Logarithm Problem (ECDLP). ECC delivers the same security level as RSA with smaller keys, and ECC runs more efficiently than RSA, which makes ECC suitable for resource-limited devices such as mobile phones. ECC serves in asymmetric encryption, digital signatures, and key exchange.

```text
ECC KeyGeneration:
1. Select the elliptic curve and the base point G of order n.
2. Choose a random private key d with 1 < d < n.
3. Compute the public key Q = d × G (scalar point multiplication).
   (Recovering d from Q requires solving the ECDLP.)
4. Public key: Q. Private key: d.

ECC Encryption (ECIES):
Input: receiver public key Q, message m.
1. Choose an ephemeral random r with 1 < r < n.
2. Compute R = r × G.
3. Compute the shared point S = r × Q.
4. Derive a symmetric key K from S.
5. Encrypt m with K to produce ciphertext c.
6. Output (R, c).

ECC Decryption (ECIES):
Input: private key d, ciphertext (R, c).
1. Compute the shared point S = d × R.
   (Note: d × R = d × r × G = r × Q, the same S the sender computed.)
2. Derive the same symmetric key K from S.
3. Decrypt c with K to recover m.
4. Output plaintext m.

ECC SignatureGeneration (ECDSA):
Input: private key d, message m.
1. Compute h = H(m).
2. Choose a random k with 1 < k < n.
3. Compute the point (x1, y1) = k × G.
4. Compute r = x1 mod n. If r = 0, go back to step 2.
5. Compute s = k^-1 × (h + d × r) mod n. If s = 0, go back to step 2.
6. Output signature (r, s).

ECC SignatureVerification (ECDSA):
Input: public key Q, message m, signature (r, s).
1. Compute h = H(m).
2. Compute w = s^-1 mod n.
3. Compute u1 = h × w mod n and u2 = r × w mod n.
4. Compute the point (x1, y1) = u1 × G + u2 × Q.
5. Accept the signature if x1 mod n = r, otherwise reject.
```

### Asymmetric Encryption: Algorithm Comparison

| Algorithm | Computational efficiency | Power consumption | Typical key size (bits) | Basis of security |
| --------- | ------------------------ | ----------------- | ----------------------- | ----------------- |
| RSA | Low | High | 1024 | Difficulty of factoring the product of two large prime numbers |
| ElGamal | Medium | Medium | 1024 | Difficulty of computing discrete logarithms |
| ECC | High | Low | 160 | Difficulty of computing elliptic curve discrete logarithms |

The key sizes in the table reflect the era of the source material, roughly 80 bits of security. Current NIST guidance recommends at least 2048 bits for RSA and ElGamal keys and at least 224 to 256 bits for ECC keys, which delivers 112 to 128 bits of security.

### Asymmetric Algorithms for Key Establishment

Asymmetric encryption also establishes symmetric keys. The Diffie-Hellman (DH) algorithm is a key exchange algorithm that generates a shared symmetric key between two parties communicating over an insecure network such as the Internet. DH generates both static keys and ephemeral keys. A static key is a long-term key used over an extended period, and the private key of a key pair is a static key. An ephemeral key is used in a single transaction and is generated fresh for each run of a key-establishment process.

Diffie-Hellman key exchange:

```text
            Alice                                   Bob
            -----                                   ---
secret:     a                                       b
compute:    A = g^a mod p                         B = g^b mod p
                |                                   |
                |--------- A (public) ------------->|
                |                                   |
                |<--------- B (public) -------------|
                |                                   |
compute:    s = B^a mod p                         s = A^b mod p
            = g^(a×b) mod p                       = g^(a×b) mod p
                |                                   |
                +------ shared secret key s ------+

Public parameters p (prime) and g (generator) are agreed in the open.
An eavesdropper sees only p, g, A, and B. Recovering a from A or
b from B requires solving the discrete logarithm problem.

With a static DH key, the same a or b serves for many sessions.
With DHE or ECDHE, a and b are freshly generated per session, so
compromising a long-term key cannot reveal past session keys:
that is perfect forward secrecy.
```

Two DH methods use ephemeral keys: Diffie-Hellman Ephemeral (DHE, also called EDH) and Elliptic Curve Diffie-Hellman Ephemeral (ECDHE), which uses ephemeral keys from ECC. DHE and ECDHE provide perfect forward secrecy (PFS), the property of a key exchange algorithm that keeps a session key safe even if the static key that generated the session key is compromised later.

### 2. Cryptographic Hash Functions

A cryptographic hash function maps a variable-length string, the message, to a fixed-size value, the message digest, also called the hash. For example, MD5 maps a message to a 128-bit hash. MD5 itself no longer provides collision resistance: cryptanalysts demonstrated MD5 collisions in 2004, and the Flame malware forged a Microsoft code-signing certificate through an MD5 collision in 2012. Hash functions serve in digital signatures and message authentication codes.

A cryptographic hash function has three properties:

- **Collision resistance:** Finding two different inputs with the same digest is computationally infeasible.
- **Preimage resistance:** Finding the original message from a digest is computationally infeasible. This property is also called the one-way property.
- **Second preimage resistance:** Finding a second input with the same digest as a given input is computationally infeasible.

The avalanche effect is a desirable property: a small change to a message produces a digest that differs sharply and shows no correlation with the original digest.

A collision occurs when a hash function produces the same digest for two different messages, and a collision attack tries to find two such messages. For example, a software package ships with its published hash so users can verify authenticity, and an attacker tries to craft malware with the same hash so a target trusts the malware. A birthday attack is a collision attack based on the birthday paradox, the idea that across two groups of people, someone in the first group shares a birthday with someone in the second group. A birthday attack raises the odds of a collision by building two groups of messages and searching for any message in the first group that shares a digest with any message in the second group.

### 3. Message Authentication Code (MAC)

A message authentication code (MAC) is a cryptographic primitive that verifies data integrity and authenticity. A MAC algorithm computes the MAC from a message combined with a symmetric key, and a MAC is also called a tag. The sender computes the MAC with a symmetric key shared with the receiver and appends the MAC to the message. The receiver then performs two steps:

1. Computes the MAC for the received message.
2. Compares the computed MAC with the received MAC.

Matching MACs assure the receiver that the sender sent the message and that nothing modified the message in transit. A MAC provides two security services:

- **Data integrity:** A match between the received MAC and the receiver's computed MAC proves the message arrived unmodified.
- **Data origin authentication:** Only the holder of the shared symmetric key can create a valid MAC, so the receiver knows the message came from the sender.

**A MAC does not provide non-repudiation.** Sender and receiver share the same key, so a sender can deny sending a message and claim the receiver forged the MAC.

A hashed message authentication code (**HMAC**) is a MAC that pairs a symmetric key with a cryptographic hash function. HMAC mixes the key directly into the hash computation rather than encrypting the digest: the sender computes H((K XOR opad) || H((K XOR ipad) || m)), where m is the message, K is the shared key, opad and ipad are fixed padding constants, and || denotes concatenation. An HMAC is named after its hash function: HMAC-MD5 uses MD5 digests, and HMAC-SHA256 produces a 256-bit HMAC. The HMAC size always equals the digest size of the underlying hash function.

### 4. Digital Signatures

A digital signature is a cryptographic primitive that combines a hash function with public-key cryptography to verify the authenticity and integrity of a digital message. The sender hashes the message and applies a signing algorithm with the sender's private key to create the signature, then sends the signature along with the message. The receiver applies a verification algorithm with the sender's public key to confirm that the sender signed the message and that nothing modified the message in transit.

Digital signature algorithms build on the three public-key families:

- **RSA Digital Signature Algorithm:** Uses RSA public-key cryptography, with security based on the difficulty of factoring the product of two large prime numbers.
- **Digital Signature Algorithm (DSA):** Uses a variant of ElGamal public-key cryptography, with security based on the difficulty of the discrete logarithm problem.
- **Elliptic Curve Digital Signature Algorithm (ECDSA):** Uses elliptic curve public-key cryptography, with security based on the difficulty of locating related points on an elliptic curve.

A signature scheme is named by combining the algorithm with the hash function. For example, `ECDSA-SHA512` is an `ECDSA` signature that uses the `SHA-512` hash function to compute the message hash.

A digital signature provides three security services:

- **Data integrity:** Matching digests prove the message arrived unmodified.
- **Data origin authentication:** Only the sender's private key can create a valid signature, so the receiver knows the message came from the sender.
- **Non-repudiation:** The sender cannot deny signing the message, because only the sender holds the private key.

A digital signature does not provide confidentiality. The sender encrypts only the message digest, so an eavesdropper can still read the message.

#### **HMAC or Digital Signature?**

HMAC and digital signatures both verify integrity and authenticity, and the difference is the key model. HMAC uses one shared symmetric key: computation is fast, but both parties can produce the MAC, so neither can prove to a third party who created the MAC. A digital signature uses the sender's private key: only the signer can produce the signature, anyone with the public key can verify the signature, and the signer cannot deny signing.

Use HMAC when:

- **The parties share a secret key and trust each other.** For example, GitHub signs webhook payloads with `HMAC-SHA256`, and a repository owner verifies that a payload came from GitHub using the shared webhook secret.
- **Speed matters more than third-party proof.** For example, two internal microservices authenticate API calls with HMAC, because HMAC runs far faster than public-key operations.
- **Only the key holders need to verify.** For example, a JWT signed with HS256 works when one issuer and one verifier share the secret.

Use a digital signature when:

- **Many parties must verify.** For example, a software vendor signs an update with the vendor's private key, and thousands of customers verify the update with the vendor's public key.
- **Non-repudiation matters.** For example, a signer cannot deny a DocuSign-signed contract, because only the signer's private key could have produced the signature.
- **Verification must be public.** For example, a JWT signed with RS256 lets any service verify a login token with the public key, while only the identity provider can issue tokens.

### 5. Digital Certificates

A digital certificate, or public key certificate, is an electronic document that proves ownership of a public key. A certificate contains the owner's public key and identity information such as name, address, and organization, and a certificate stays valid only for a fixed validity period. A Certificate Authority (CA) verifies the certificate content and digitally signs the certificate. For example, a certificate issued to Google lists the issuing CA as Issuer, the valid-from and valid-to dates as the validity period, Google's identity information as Subject, and Google's public key. Only the certificate owner holds the private key that matches the certificate's public key, and certificate users use the public key to communicate securely with the owner and to validate documents the owner's private key signed.

#### **Digital Certificate Types**

Web server certificates, also called domain certificates or TLS certificates, establish secure connections between web servers and web clients. Four types exist:

- **Domain validation (DV) certificate:** Verifies only the identity of the domain owner.
- **Extended validation (EV) certificate:** Verifies the domain owner's identity, exclusive control over the domain, and legal and physical existence.
- **Wildcard certificate:** Validates a domain and all of the domain's subdomains.
- **Subject Alternative Name (SAN) certificate:** Covers multiple domains owned by the same owner. A SAN certificate is also called a Unified Communication Certificate (UCC).

Other certificate types include:

- **Root certificate:** Created and self-signed by a CA. A self-signed certificate has the same issuer and subject, and a root certificate depends on no higher authority. A CA uses its root certificate to sign other certificates.
- **Code signing certificate:** Used by a software developer or publisher to sign programs. An installer or operating system uses the certificate to check program integrity before installation.
- **Email certificate:** Used to digitally sign email. The sender signs the message digest with the sender's private key, and the recipient verifies the signature with the public key from the sender's email certificate. Encrypting the email content itself is a separate step that uses the recipient's public key.
- **Machine certificate:** Issued to a hardware device such as a computer, router, or printer, and used to authenticate the device on a network. A machine certificate is also called a computer certificate.

#### **Digital Certificate Formats**

The X.509 standard defines the format of a public key certificate as key-value pairs, where each key names a field and each value holds a number, string, or list. `X.509` certificates are saved in several formats:

- **Privacy Enhanced Mail (PEM):** An ASCII format. PEM encodes binary data with Base64, a binary-to-text scheme that represents binary data as an ASCII string. File extensions: `.pem`, `.crt`, `.cer`.
- **Distinguished Encoding Rules (DER):** A binary format. DER is a subset of Abstract Syntax Notation One (ASN.1), a platform-independent encoding format, and DER ensures certificate content has exactly one encoding. File extensions: `.der`, `.cer`.
- **Personal Information Exchange (PFX):** A password-protected archive format that holds a certificate and the matching private key, used by a server to import both from one file. File extension: `.pfx`.
- **Public Key Cryptography Standards #7 and #12 (PKCS #7, PKCS #12):** Binary formats that define a standard syntax for storing encrypted and signed data, stored as DER binary or PEM ASCII. File extensions: `.p7b` for PKCS #7 and `.p12` for PKCS #12.

### 6. Public Key Infrastructure (PKI)

A Public Key Infrastructure (PKI) is a framework for managing digital certificates and public keys. A PKI lets users of an insecure network such as the Internet exchange data securely with public and private keys, and a PKI spans the hardware, software, people, policies, and procedures that create, renew, revoke, and distribute certificates. Five components make up a PKI:

- **Certificate Authority (CA):** Issues, renews, revokes, and distributes certificates. The CA is a third party trusted by the certificate owner (the subject) and the certificate user (the relying party), and the relying party depends on the accuracy of the binding between the certificate's public key and the owner's identity.
- **Registration Authority (RA):** Verifies the identity of a certificate applicant. The applicant sends the RA a Certificate Signing Request (CSR) containing the applicant's public key, name, organization, department, and physical and email addresses. The RA validates the identity and approves or rejects the CSR, and an approved CSR goes to the CA for certificate issuance.
- **Certificate Repository (CR):** A central directory that stores the certificates a CA has issued. The CR also holds the Certificate Revocation List (CRL), the list of certificates a CA revoked before the end of their validity periods.
- **Certificate Policy (CP):** Defines the structure of the PKI, the PKI's entities and roles, and the PKI's procedures and operational requirements.
- **Certificate Practice Statement (CPS):** Describes how the CA issues, renews, revokes, and distributes certificates, and helps a certificate user decide whether to trust the PKI's certificates.

#### **Certificate Life Cycle**

A certificate passes through four stages during its lifetime:

1. **Issuance:** The CA issues the certificate after the RA validates the applicant's identity, and the issued certificate is stored in the CR.
2. **Revocation:** The CA revokes the certificate before the validity period ends, and the revoked certificate is added to the CRL.
3. **Suspension:** The CA temporarily suspends the certificate's validity. A suspended certificate can be reinstated or revoked.
4. **Expiration:** The certificate expires at the end of its validity period, and the PKI's CP defines the process for applying for a new certificate.

#### **PKI Trust Models**

A PKI trust model, or PKI architecture, describes the trust relationship between a PKI and the PKI's certificate users, and the model lets a certificate user judge the legitimacy of issued certificates. Three main trust models exist.

##### Single CA Trust Model

One CA issues all certificates. A compromise of the CA's private key revokes every certificate in the PKI, and the model does not scale to networks with many entities.

```text
Single CA trust model:

              +--------+
              |   CA   |
              +---+----+
                  |
          +-------+-------+
          |       |       |
          v       v       v
       +----+  +----+  +----+
       | E1 |  | E2 |  | E3 |
       +----+  +----+  +----+

E = entity holding a certificate issued by the CA.
One CA compromise revokes every certificate.
```

##### Hierarchical Trust Model

A root CA issues certificates to intermediate CAs, and intermediate CAs issue certificates to entities. The root CA never issues certificates to entities. A certificate user trusts an intermediate CA's certificates because the root CA trusts the intermediate CA, and the chain of trust describes the trust relationship between the user and the issuing intermediate CA. The root CA is called the trust anchor, and a compromise of one intermediate CA affects only the certificates that intermediate CA issued. The hierarchical model is the most common trust model on the Internet.

```text
Hierarchical trust model:

                +-----------+
                |  Root CA  |  <- trust anchor
                +-----+-----+
                      |
              +-------+-------+
              |               |
              v               v
       +--------------+ +--------------+
       | Intermediate | | Intermediate |
       |     CA 1     | |     CA 2     |
       +------+-------+ +------+-------+
              |               |
           +--+---+        +--+---+
           |      |        |      |
           v      v        v      v
          E1     E2       E3     E4

Chain of trust: E1 -> intermediate CA 1 -> root CA.
Compromise of intermediate CA 1 affects only E1 and E2.
```

##### Bridge Trust Model

A bridge trust model links PKIs that use different trust models through a bridge CA (BCA). The bridge CA only establishes trust paths between the linked PKIs and issues no certificates to entities.

```text
Bridge trust model:

                        +----------------+
                        |   Bridge CA    |
                        +-------+--------+
                                |
          +---------------------+---------------------+
          |                                           |
          v                                           v
    +-----------+                               +-----------+
    |  Root A   |                               |  Root B   |
    +-----+-----+                               +-----+-----+
          |                                           |
    +-----------+                               +-----------+
    |  Int. A1  |                               |  Int. B1  |
    +-----+-----+                               +-----+-----+
          |                                           |
        +--+--+                                     +--+--+
        |     |                                     |     |
        v     v                                     v     v
       E1    E2                                    E3    E4

   PKI A (hierarchical)                   PKI B (hierarchical)

The bridge CA cross-certifies with the root CA of each linked
PKI and issues no certificates to entities.

Trust path from E1 to E3:
E1 -> Int. A1 -> Root A -> Bridge CA -> Root B -> Int. B1 -> E3
```

### 7. Obfuscation Methods

#### **Steganography**

Steganography hides data by embedding the data in a host medium without noticeably changing the medium's appearance or function. For example, a text message can be embedded in a PNG image without affecting the image's visible quality or rendering. Steganography provides a covert way to protect and transmit sensitive data: concealing copyright information inside digital content protects intellectual property, embedding data in ordinary images secures confidential data, and hiding data in network packets enables secret communication. Steganographic techniques exploit the properties of each medium. For example, text steganography hides data in invisible characters such as tabs and spaces without changing the text's readability, structure, formatting, or semantics.

#### **Tokenization**

Tokenization replaces a sensitive data element with a non-sensitive element called a token. For example, a random string of numbers can stand in for a social security number in a database. A token vault is a secure storage system that maintains the mapping between sensitive data and the matching tokens. A token carries no external meaning or exploitable value and serves only as an identifier for mapping back to the sensitive data. Tokenization and encryption both protect sensitive data, but tokenized data is not reversible and shares no mathematical relationship with the original data, unlike encrypted data. For example, medical records are tokenized to preserve patient confidentiality while still allowing data analysis without exposing sensitive information.

## Summary

This chapter explained asymmetric encryption through the RSA, ElGamal, and ECC algorithms and Diffie-Hellman key establishment with perfect forward secrecy. The chapter covered hash functions and their three resistance properties, message authentication codes and HMAC, and digital signatures with RSA, DSA, and ECDSA. The chapter then described digital certificates, X.509 formats, the five PKI components, and the three trust models, and the obfuscation methods of steganography and tokenization.

## Useful References and Resources

- Whitfield Diffie and Martin Hellman, "New Directions in Cryptography" (1976), the paper that introduced public-key cryptography and the Diffie-Hellman key exchange.
- Ronald Rivest, Adi Shamir, and Leonard Adleman, "A Method for Obtaining Digital Signatures and Public-Key Cryptosystems" (1978), the original RSA paper.
- NIST FIPS 180-4, "Secure Hash Standard," the specification of the SHA-1 and SHA-2 hash functions.
- NIST FIPS 198-1, "The Keyed-Hash Message Authentication Code (HMAC)," the specification of HMAC.
- NIST FIPS 186-5, "Digital Signature Standard (DSS)," the specification of the RSA, DSA, and ECDSA signature algorithms. FIPS 186-5 permits DSA only for verifying legacy signatures, not for generating new ones.
- NIST Special Publication 800-56A Rev 3, "Recommendation for Pair-Wise Key-Establishment Schemes Using Discrete Logarithm Cryptography," covering Diffie-Hellman and elliptic curve key establishment.
- RFC 5280, "Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile," the X.509 certificate and CRL format used on the Internet.
- PCI Security Standards Council, "Information Supplement: PCI DSS Tokenization Guidelines" (2011), guidance on tokenization and token vaults for payment data.
