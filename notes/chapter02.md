# Chapter 02: Cryptography and Symmetric Encryption

## Abstract

This chapter introduces cryptography as the science of hiding the meaning of messages and presents the four security services and four cryptographic primitives. The chapter moves from historical substitution and transposition ciphers to the foundations of modern encryption: keys, key spaces, key stretching, confusion, diffusion, and Kerckhoffs's principle. The chapter then covers symmetric encryption in depth, from stream ciphers such as RC4, A5/1, and ChaCha20 to block ciphers with padding and the five modes of operation. The chapter closes with the DES, 3DES, and AES standards, including a detailed walkthrough of the AES state matrix, round operations, and key expansion.

## Objectives

- Define cryptography, cryptanalysis, and cryptology, and identify the four security services and four cryptographic primitives.
- Describe substitution and transposition ciphers and explain the shift cipher, the rail fence cipher, and the columnar cipher.
- Explain encryption and decryption, keys and key spaces, key length and key stretching, confusion and diffusion, and Kerckhoffs's principle.
- Describe symmetric encryption and explain how stream ciphers encrypt data one bit at a time using a keystream.
- Describe block ciphers, padding, and the five modes of operation with their strengths and weaknesses, and compare DES, 3DES, and AES key sizes and round counts.

## Content

### 1. Cryptographic Principles

Cryptography is the science of secret writing with the goal of hiding the meaning of a message. Cryptanalysis is the science of breaking cryptography. Cryptology covers both fields together.

Cryptography delivers four security services:

- **Confidentiality:** Only authorized entities can disclose data. Confidentiality also goes by the name secrecy.
- **Integrity:** No unauthorized modification of data since an authorized entity created, stored, or transmitted the data. The integrity service does not block modification but supplies a way to detect modification.
- **Message authentication:** A receiver can verify the source of a message. Message authentication also goes by the name data origin authentication, and the service implies message integrity.
- **Non-repudiation:** A sender cannot deny sending a message. The service assures the receiver that the message came from the sender.

Cryptographic primitives are the basic building blocks of cryptography, and each primitive provides a security service. Four primitives exist:

- **Encryption:** An algorithm and a key hide the meaning of a message. Asymmetric encryption uses two different but mathematically related keys, while symmetric encryption uses one key for both encryption and decryption. Symmetric encryption splits into stream ciphers, which encrypt one bit at a time, and block ciphers, which encrypt one block of bits at a time. Encryption provides confidentiality.
- **Cryptographic hash function:** Outputs a fixed-length string for a variable-length input string. A hash function provides integrity.
- **Message authentication code (MAC):** Combines a hash function with symmetric encryption to provide integrity and authenticity.
- **Digital signature:** Combines a hash function with asymmetric encryption to provide integrity, authenticity, and non-repudiation.

### 2. Historical Cryptosystems

Historical cryptosystems predate the computer age. Modern cryptosystems rely on the difficulty of solving mathematical problems, while historical cryptosystems used simple operations to scramble letters of the alphabet. Historical ciphers no longer provide meaningful security because cryptanalysts solve them easily, yet two historical techniques, substitution and transposition, still form the basis of modern cryptography.

#### **Substitution Ciphers**

A substitution cipher replaces one element of plaintext with a different element to produce ciphertext. Two variants exist:

- **Monoalphabetic substitution cipher:** Each plaintext letter maps to one fixed ciphertext letter. For example, the letter C always becomes the letter X.
- **Polyalphabetic substitution cipher:** One plaintext letter maps to different ciphertext letters. For example, the letter C can become P, T, X, or any other letter.

For example, a monoalphabetic cipher with the fixed mapping H to M, E to T, L to G, and O to Q encrypts the plaintext HELLO to the ciphertext MTGGQ. The same fixed mapping decrypts MTGGQ back to HELLO every time, and every occurrence of L in the plaintext becomes G in the ciphertext.

A polyalphabetic cipher changes the mapping with each letter position. For example, the plaintext HELLO can encrypt to KMPXR: H becomes K, E becomes M, the first L becomes P, the second L becomes X, and O becomes R. The two identical plaintext letters L produce two different ciphertext letters, so the ciphertext hides the letter repetition that HELLO shows.

A shift cipher replaces each plaintext letter with the letter a fixed number of positions to the right in the alphabet, and that fixed number serves as the encryption key. The Caesar cipher is a shift cipher with a key of 3: each letter moves 3 positions right, and Z wraps back to A. For example, the Caesar cipher turns A into D, because D sits 3 positions to the right of A.

Caesar cipher encryption with a key of 3:

```text
Plaintext:   A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
             | | | | | | | | | | | | | | | | | | | | | | | | | |
             v v v v v v v v v v v v v v v v v v v v v v v v v v
Ciphertext:  D E F G H I J K L M N O P Q R S T U V W X Y Z A B C

             H E L L O
             | | | | |
             v v v v v
             K H O O R
```

#### **Transposition Ciphers**

A transposition cipher, also called a permutation cipher, reorders the elements of a plaintext without adding or removing elements, so the ciphertext contains the same elements in a different order. For example, a cipher that reverses letter order turns the word "elephant" into "tnahpele."

**A rail fence cipher** writes plaintext letters diagonally downward on successive rails of an imaginary fence, then diagonally upward after reaching the bottom rail, repeating the pattern until every letter sits on a rail. The ciphertext is read rail by rail from top to bottom, and the number of rails serves as the key. For example, the plaintext HELLOWORLD with a key of 3 rails encrypts to the ciphertext HOLELWRDLO.

Rail fence cipher with a key of 3:

```text
H . . . O . . . L .
. E . L . W . R . D
. . L . . . O . . .

Rail 1: H O L
Rail 2: E L W R D
Rail 3: L O

Ciphertext: H O L E L W R D L O
```

**A columnar cipher** places plaintext letters in rows with a length equal to the keyword length, then reads the letters out in columns. The alphabetical order of the keyword letters sets the column read order. For example, with the keyword RUN, the plaintext fills rows of length 3. The alphabetical order of R, U, N is "2 3 1" because N comes first in the alphabet, then R, then U, so the cipher reads the third column first, then the first column, then the second column. For example, the plaintext HELLOWORLD with the keyword RUN encrypts to the ciphertext LWLHLODEOR.

Columnar cipher with the keyword RUN:

```text
Keyword:    R   U   N
Order:      2   3   1
          +---+---+---+
          | H | E | L |
          | L | O | W |
          | O | R | L |
          | D |   |   |
          +---+---+---+

Column read order: 3rd, 1st, 2nd

Column 3 (N): L W L
Column 1 (R): H L O D
Column 2 (U): E O R

Ciphertext: L W L H L O D E O R
```

### 3. Encryption

Encryption is the process of encoding or scrambling a message, and decryption is the process of reversing that scrambling. Both processes follow a fixed sequence of instructions called an algorithm, and an encryption or decryption algorithm is called a cipher.

An encryption algorithm transforms plaintext into ciphertext using a key:

- **Key:** A string of bits.
- **Key space:** The set of all possible keys.
- **Plaintext:** The unencrypted, unscrambled message.
- **Ciphertext:** The encrypted, scrambled message.

Key length is the size of the key measured in bits. Longer keys raise security by increasing the number of possible combinations, which makes the encryption more resistant to brute-force attacks. Key stretching artificially increases a key's length and complexity, which makes short or simple keys harder to brute-force.

**A secure encryption algorithm has two properties:**

- **Confusion:** Changing one bit of the encryption key changes most of the ciphertext bits. Confusion hides the relationship between the ciphertext and the key.
- **Diffusion:** Changing one plaintext bit changes about half of the ciphertext bits, and changing one ciphertext bit changes about half of the plaintext bits. Diffusion hides the relationship between the plaintext and the ciphertext.

Kerckhoffs's principle states that the security of a cryptographic algorithm depends on the secrecy of the key, not on the secrecy of the algorithm. The method of encryption can be public knowledge, while the key stays secret. Every contemporary encryption algorithm follows this principle, and the details of the most widely used algorithms are public and have survived extensive cryptanalysis.

### 4. Symmetric Encryption: Stream Ciphers

Symmetric encryption uses the same key to encrypt and decrypt data, and that key is called a symmetric key or secret key. Two classes of symmetric encryption exist:

- **Stream cipher:** Encrypts data one bit at a time.
- **Block cipher:** Encrypts data one block at a time.

Symmetric encryption runs faster and more efficiently than asymmetric encryption because symmetric encryption uses shorter keys and simpler operations. Protocols such as HTTPS and IPSec rely on symmetric encryption.

A stream cipher converts the encryption key into a keystream, a continuous stream of bits produced by an algorithm called a keystream generator. Sender and receiver share the same key and keystream generator to encrypt and decrypt. Encryption and decryption use a simple operation such as exclusive or (XOR), which makes stream ciphers computationally efficient. Because a stream cipher works one bit at a time, errors do not propagate: a one-bit error in encryption causes a one-bit error in decryption. Stream ciphers suit communication applications that need fast encryption of a continuous data stream. For example, the A5/1 stream cipher encrypts data exchanged between a mobile phone and a base station.

#### **Real-World Stream Ciphers**

- **RC4:** Ron Rivest designed RC4 in 1987. RC4 accepts keys from 40 to 2048 bits and served in WEP wireless security and early versions of TLS. Cryptanalysts found biases in the RC4 keystream, and those biases broke WEP. RFC 7465 prohibited RC4 in TLS in 2015.
- **A5/1:** GSM mobile networks encrypt calls between a phone and a base station with A5/1. A5/1 combines three linear feedback shift registers of 19, 22, and 23 bits under a 64-bit key. Practical cryptanalysis broke A5/1, and GSM networks moved to stronger ciphers such as A5/3.
- **ChaCha20:** Daniel J. Bernstein published ChaCha20 in 2008. ChaCha20 uses a 256-bit key and 20 rounds of addition, rotation, and XOR. TLS 1.3, OpenSSH, and the WireGuard VPN use ChaCha20, often paired with the Poly1305 authenticator.

#### **How a Stream Cipher Encrypts**

The keystream generator turns the shared key into a stream of bits, and XOR combines each plaintext bit with the matching keystream bit. XOR is its own inverse, so decryption runs the identical operation with the identical keystream.

```text
            shared secret key
                    |
                    v
          +---------------------+
          | keystream generator |
          +---------------------+
                    |
                    v   keystream bits
 plaintext bits --> XOR --> ciphertext bits

XOR truth table: 0 XOR 0 = 0    0 XOR 1 = 1
                 1 XOR 0 = 1    1 XOR 1 = 0

Encryption:

Plaintext:   1 0 1 1 0 0 1 1 1 0 1 0
Keystream:   1 1 0 1 0 1 1 0 1 0 0 1
             -----------------------
Ciphertext:  0 1 1 0 0 1 0 1 0 0 1 1

Decryption (ciphertext XOR the same keystream returns the plaintext):

Ciphertext:  0 1 1 0 0 1 0 1 0 0 1 1
Keystream:   1 1 0 1 0 1 1 0 1 0 0 1
             -----------------------
Plaintext:   1 0 1 1 0 0 1 1 1 0 1 0
```

### 5. Symmetric Encryption: Block Ciphers

A block cipher encrypts a data block of fixed size, one block at a time. For example, the International Data Encryption Algorithm (IDEA) encrypts data in 64-bit blocks. Block ciphers also serve inside other primitives such as hash functions and message authentication codes.

For a variable-length plaintext, the cipher first partitions the data into blocks. If the plaintext length is not a multiple of the block size, the cipher fills the last block with random bits, a process called padding. For example, with a 64-bit block size, an 84-bit plaintext splits into one 64-bit block and one 20-bit block, and the second block receives 44 padding bits to reach 64 bits. A block cipher runs more efficiently than a stream cipher when the plaintext size is known in advance, while a stream cipher runs more efficiently when the size is unknown or the data arrives as a continuous stream.

#### **DES and 3DES**

The Data Encryption Standard (DES) is a symmetric block cipher that encrypts 64-bit blocks with a 56-bit key. **DES is no longer secure** because modern cryptanalytic techniques break a 56-bit key. The Triple Data Encryption Standard (3DES) applies DES three times in a row to each 64-bit block, which raises the key size without a new cipher design. Three keying options exist:

- **Option 1:** Three independent 56-bit keys, for a total of 168 independent key bits. The most secure option.
- **Option 2:** Two independent 56-bit keys, for a total of 112 independent key bits.
- **Option 3:** Three identical 56-bit keys. The most insecure option, kept for backward compatibility with DES.

#### **Advanced Encryption Standard (AES)**

The Advanced Encryption Standard (AES) is a symmetric block cipher that encrypts 128-bit blocks with 128, 192, or 256-bit keys. AES is the most widely used block cipher. VPNs use AES to build encrypted tunnels, wireless networks use AES to encrypt data in transit, storage devices use AES to encrypt data at rest, and HTTPS uses AES to secure Internet communications.

An encryption round is a set of operations applied consecutively to a block of plaintext bits during encryption. The key length sets the round count in AES, and a longer key means more rounds and a larger key space, both of which strengthen the cipher against brute-force attacks.

| Version | Key length (bits) | Encryption rounds |
| ------- | ----------------- | ----------------- |
| AES-128 | 128 | 10 |
| AES-192 | 192 | 12 |
| AES-256 | 256 | 14 |

##### **How AES Works**

Joan Daemen and Vincent Rijmen designed AES under the name Rijndael. NIST selected Rijndael from 15 candidate ciphers in a public competition in 2000 and published AES as FIPS 197 in 2001. The algorithm is royalty free.

AES operates on bytes rather than bits. AES loads each 128-bit block into a 4 by 4 grid of 16 bytes called the state, filling the grid column by column from top to bottom and left to right.

```text
The state matrix:

+-----+-----+-----+-----+
| b0  | b4  | b8  | b12 |
+-----+-----+-----+-----+
| b1  | b5  | b9  | b13 |
+-----+-----+-----+-----+
| b2  | b6  | b10 | b14 |
+-----+-----+-----+-----+
| b3  | b7  | b11 | b15 |
+-----+-----+-----+-----+
```

##### **Round Operations**

Every AES round applies four operations to the state:

- **SubBytes:** Replaces each byte of the state with a byte from a fixed 256-entry lookup table called the S-box. SubBytes is the only non-linear operation in AES and provides the confusion property.
- **ShiftRows:** Rotates each row of the state to the left: row 0 stays in place, row 1 shifts by 1 byte, row 2 shifts by 2 bytes, and row 3 shifts by 3 bytes. ShiftRows spreads bytes across columns.
- **MixColumns:** Multiplies each column of the state by a fixed matrix using arithmetic in the finite field GF(2^8). Changing one input byte changes all 4 bytes of the output column. The final round omits MixColumns.
- **AddRoundKey:** XORs the state with a 128-bit round key derived from the encryption key.

```text
ShiftRows before and after:

+-----+-----+-----+-----+     +-----+-----+-----+-----+
| b0  | b4  | b8  | b12 |     | b0  | b4  | b8  | b12 |
+-----+-----+-----+-----+     +-----+-----+-----+-----+
| b1  | b5  | b9  | b13 | --> | b5  | b9  | b13 | b1  |
+-----+-----+-----+-----+     +-----+-----+-----+-----+
| b2  | b6  | b10 | b14 | --> | b10 | b14 | b2  | b6  |
+-----+-----+-----+-----+     +-----+-----+-----+-----+
| b3  | b7  | b11 | b15 | --> | b15 | b3  | b7  | b11 |
+-----+-----+-----+-----+     +-----+-----+-----+-----+
```

SubBytes provides confusion, and ShiftRows with MixColumns provide diffusion, the two properties of a secure cipher from Section 3. One changed input byte affects all 16 bytes of the state after two rounds.

##### **Round Structure**

Encryption starts with an initial AddRoundKey, runs 9, 11, or 13 full rounds depending on key length, and ends with a final round that skips MixColumns.

```text
plaintext block (16 bytes)
        |
        v
  AddRoundKey (round key 0)
        |
        v
+------------------------------+
| full rounds 1 to N-1:        |
|   SubBytes                   |
|   ShiftRows                  |
|   MixColumns                 |
|   AddRoundKey                |
+------------------------------+
        |
        v
  final round N:
    SubBytes
    ShiftRows
    AddRoundKey (no MixColumns)
        |
        v
ciphertext block (16 bytes)

N = 10 for AES-128, 12 for AES-192, 14 for AES-256
```

##### **Key Expansion**

The key expansion routine, called the Rijndael key schedule, stretches the original key into one 128-bit round key per round plus the initial AddRoundKey step: 11 round keys for AES-128, 13 for AES-192, and 15 for AES-256. The schedule rotates and substitutes 4-byte words of the key and XORs in round constants, so no two round keys are identical.

##### **Security and Hardware Support**

No practical attack breaks full-round AES faster than brute force. The NSA approved AES-256 for TOP SECRET classified information in 2003. Intel and AMD processors have included AES-NI instructions since 2010, so AES runs in hardware on nearly every modern CPU.

#### **Modes of Operation**

A mode of operation defines how a block cipher applies its single-block operation to a plaintext larger than one block. Five common modes exist.

##### **Electronic Code Book (ECB)**

In ECB mode each plaintext block is encrypted separately with the same key to generate a ciphertext block, so the ciphertext depends only on the key and the plaintext. Identical plaintext blocks produce identical ciphertext blocks. Because blocks are independent, an error in one plaintext block corrupts only one ciphertext block.

```text
ECB encryption (E = block cipher with key K):

  P1          P2          P3
   |           |           |
   v           v           v
 [ E ]       [ E ]       [ E ]
   |           |           |
   v           v           v
  C1          C2          C3
```

- **Strengths:** Simple and fast. Encryption and decryption run in parallel because no block depends on another block, and random access to any block is possible.
- **Weaknesses:** Identical plaintext blocks produce identical ciphertext blocks, so the ciphertext leaks plaintext patterns. An image encrypted with ECB keeps its visible outline. Never use ECB for real data.

##### **Cipher Block Chaining (CBC)**

In CBC mode each plaintext block is XORed with the previous ciphertext block before encryption, which chains all ciphertext blocks together. An initialization vector (IV), a random bit sequence the size of one block, encrypts the first block so that every encryption run produces a unique ciphertext. Because of the chaining, an error in one ciphertext block corrupts every subsequent block.

```text
CBC encryption (E = block cipher with key K, (+) = XOR):

       IV            C1             C2
        |             |              |
        v             v              v
P1 --> (+)    P2 --> (+)     P3 --> (+)
        |             |              |
      [ E ]         [ E ]          [ E ]
        |             |              |
        v             v              v
       C1            C2             C3
```

- **Strengths:** Chaining hides plaintext patterns, so identical plaintext blocks encrypt to different ciphertext blocks.
- **Weaknesses:** Encryption runs sequentially because each block waits for the previous ciphertext block. Errors propagate through the chain, and the required padding enables padding oracle attacks: POODLE in 2014 broke SSL 3.0 through its CBC padding. The IV must be random and unique for every message.

##### **Cipher Feedback (CFB)**

In CFB mode the block cipher operates as a stream cipher. An IV encrypts the first plaintext block, blocks stay chained as in CBC, and each plaintext block is encrypted and XORed with the previous ciphertext block. Encryption and decryption are the same process, and errors propagate as in CBC.

```text
CFB encryption (E = block cipher with key K, (+) = XOR):

       IV            C1             C2
        |             |              |
        v             v              v
      [ E ]         [ E ]          [ E ]
        |             |              |
        v             v              v
P1 --> (+)    P2 --> (+)     P3 --> (+)
        |             |              |
        v             v              v
       C1            C2             C3
```

- **Strengths:** Operates as a stream cipher, so no padding is needed and partial blocks encrypt directly.
- **Weaknesses:** Encryption runs sequentially, and errors propagate through the chain. A flipped ciphertext bit flips the matching plaintext bit because CFB provides no integrity.

##### **Output Feedback (OFB)**

In OFB mode the block cipher generates keystream blocks that are XORed with plaintext blocks to produce ciphertext blocks. No chaining dependencies exist because each keystream block is created independently of the plaintext and ciphertext blocks. Encryption and decryption are the same process, and errors do not propagate.

```text
OFB encryption (E = block cipher with key K, (+) = XOR):

        IV
        |
        v
      [ E ] ------> [ E ] ------> [ E ]
        |             |             |
        v             v             v
P1 --> (+)     P2 --> (+)     P3 --> (+)
        |             |             |
        v             v             v
       C1            C2            C3
```

- **Strengths:** The keystream is independent of the plaintext and can be precomputed in advance. Errors do not propagate, which suits noisy channels such as satellite links.
- **Weaknesses:** A flipped ciphertext bit flips the matching plaintext bit because OFB provides no integrity. Reusing a key and IV pair leaks plaintext: the XOR of two ciphertexts equals the XOR of the two plaintexts.

##### **Counter (CTR)**

In CTR mode the block cipher operates as a stream cipher. Sender and receiver use a synchronized counter that computes a new shared value for each ciphertext block exchanged. Each ciphertext block depends on the position of its plaintext block, and because no block depends on any other block, errors do not propagate.

```text
CTR encryption (E = block cipher with key K, (+) = XOR):

      ctr 1          ctr 2         ctr 3
        |             |             |
        v             v             v
      [ E ]         [ E ]         [ E ]
        |             |             |
        v             v             v
P1 --> (+)     P2 --> (+)     P3 --> (+)
        |             |             |
        v             v             v
       C1            C2            C3

Each ctr value combines a nonce with the counter position.
```

- **Strengths:** Encryption and decryption run in parallel, any block can be decrypted without the others, errors do not propagate, and no padding is needed.
- **Weaknesses:** Reusing a counter value with the same key is catastrophic: an attacker recovers both plaintexts by XORing the two ciphertexts. CTR provides no integrity, so a flipped ciphertext bit flips the matching plaintext bit.

##### **Choosing a Mode in Practice**

CTR is the recommended base mode for new designs, always paired with a message authentication code for integrity. AES-GCM, which combines CTR with the GMAC authenticator, is the required cipher in TLS 1.3 and secures most HTTPS traffic. CBC survives in legacy systems when paired with a MAC and a fresh random IV per message. CFB and OFB are rare in new designs. Never use ECB for real data.

## Summary

This chapter defined cryptography, cryptanalysis, and cryptology, and presented the four security services and four cryptographic primitives. The chapter traced substitution and transposition ciphers from historical systems to modern encryption fundamentals: keys, key spaces, key stretching, confusion, diffusion, and Kerckhoffs's principle. The chapter covered symmetric encryption with stream ciphers such as RC4, A5/1, and ChaCha20, and with block ciphers, padding, and the five modes of operation with their strengths and weaknesses. The chapter closed with the DES, 3DES, and AES standards and a detailed walkthrough of the AES rounds.

## Useful References and Resources

- Auguste Kerckhoffs, "La Cryptographie Militaire" (1883), the original statement of Kerckhoffs's principle.
- NIST FIPS 46-3, "Data Encryption Standard (DES)," the withdrawn specification of DES.
- NIST Special Publication 800-67 Rev 2, "Recommendation for the Triple Data Encryption Algorithm (TDEA) Block Cipher," the specification of 3DES and its three keying options.
- NIST FIPS 197, "Advanced Encryption Standard (AES)," the specification of the AES block cipher and its 10, 12, and 14 round variants.
- NIST Special Publication 800-38A, "Recommendation for Block Cipher Modes of Operation," the definitions of the ECB, CBC, CFB, OFB, and CTR modes.
- NIST Special Publication 800-38D, "Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC," the authenticated mode recommended for new designs.
- RFC 8439, "ChaCha20 and Poly1305 for IETF Protocols" (2018), the specification of the ChaCha20 stream cipher.
- NIST Special Publication 800-57 Part 1, "Recommendation for Key Management," guidance on key length, key stretching, and cryptoperiods.
