# Lab 02 — Historical Cryptosystems

## Overview

In this lab, you will study two classic historical ciphers:

* The Caesar cipher, a substitution cipher
* The Rail Fence cipher, a transposition cipher

You will implement both ciphers in Python and test them with example messages. This lab introduces the basic idea of encryption, key-based transformations, and how simple historical systems work before modern cryptography.

By the end of this lab, you should be able to:

* explain the difference between substitution and transposition ciphers
* implement Caesar cipher encryption and decryption
* implement Rail Fence cipher encryption and decryption
* test your code with sample inputs and verify the output

---

# Part 1 — Caesar Cipher

## What is a Caesar Cipher?

The Caesar cipher is a substitution cipher where each letter in the plaintext is shifted forward by a fixed number of positions in the alphabet.

For example, with a shift of 3:

* A → D
* B → E
* C → F
* ...
* X → A
* Y → B
* Z → C

This is called a substitution cipher because letters are replaced by other letters based on a pattern.

## Example

Plaintext:

```text
HELLO
```

Shift = 3

Ciphertext:

```text
KHOOR
```

## Your Task

Write a Python program that implements the following functions:

```python
def caesar_encrypt(plaintext, shift):
    pass


def caesar_decrypt(ciphertext, shift):
    pass
```

### Requirements

* Keep only alphabetic characters.
* Preserve the original letter case (uppercase stays uppercase, lowercase stays lowercase).
* Ignore spaces and punctuation when encrypting/decrypting.
* Use a wraparound alphabet, so letters continue from Z to A and from z to a.

### Example Input and Output

```python
print(caesar_encrypt("HELLO WORLD", 3))
# Output: KHOOR ZRUOG

print(caesar_decrypt("KHOOR ZRUOG", 3))
# Output: HELLO WORLD
```

### Challenge Questions

1. What happens if the shift value is larger than 26?
2. How can you normalize the shift so it always stays within the alphabet size?
3. Why is the Caesar cipher easy to break?

---

# Part 2 — Rail Fence Cipher

## What is a Rail Fence Cipher?

The Rail Fence cipher is a transposition cipher. Instead of replacing letters with other letters, it rearranges the order of the letters.

The message is written in rows, like a fence, and then read row by row.

## Example

Plaintext:

```text
WEAREDISCOVEREDFLEEATONCE
```

Rail count = 3

The text is arranged as:

```text
W   E   C   R   L   T   E
 E A I E D S O E E A I O
  A   R   D   F   E   C   S
```

Ciphertext:

```text
WECRLTEERDSOEEFEAOCAIVDEN
```

## Your Task

Write a Python program that implements the following functions:

```python
def rail_fence_encrypt(plaintext, rails):
    pass


def rail_fence_decrypt(ciphertext, rails):
    pass
```

### Requirements

* The function should accept a plaintext string and the number of rails.
* The encryption should arrange the text in zig-zag order.
* The decryption should recover the original text from the encrypted form.
* Spaces may be kept or removed depending on your design, but be consistent.

### Example Input and Output

```python
print(rail_fence_encrypt("WEAREDISCOVEREDFLEEATONCE", 3))
# Output: WECRLTEERDSOEEFEAOCAIVDEN

print(rail_fence_decrypt("WECRLTEERDSOEEFEAOCAIVDEN", 3))
# Output: WEAREDISCOVEREDFLEEATONCE
```

### Challenge Questions

1. Why is this considered a transposition cipher instead of a substitution cipher?
2. What happens when the number of rails is 1 or 2?
3. Why is the Rail Fence cipher still vulnerable to cryptanalysis?

---

# Assignment

Create a Python file named:

```text
lab02.py
```

Your program must include the following functions:

```python
def caesar_encrypt(plaintext, shift):
    pass


def caesar_decrypt(ciphertext, shift):
    pass


def rail_fence_encrypt(plaintext, rails):
    pass


def rail_fence_decrypt(ciphertext, rails):
    pass
```

Then write a small test section that demonstrates each cipher.

## Required Test Cases

### Caesar Cipher Tests

```python
print(caesar_encrypt("ATTACK AT DAWN", 5))
print(caesar_decrypt("FXXF?K?F?I?", 5))
print(caesar_encrypt("COMP301", 7))
print(caesar_decrypt(caesar_encrypt("COMP301", 7), 7))
```

### Rail Fence Tests

```python
print(rail_fence_encrypt("HELLO WORLD", 3))
print(rail_fence_decrypt("HOLEL WDLRO", 3))
print(rail_fence_encrypt("SECRETSHARED", 4))
print(rail_fence_decrypt(rail_fence_encrypt("SECRETSHARED", 4), 4))
```

Your output should be clearly readable and your code should be well organized.

---

# Submission Guidelines

Submit the following:

1. Your Python source file: `lab02.py`


---

# Bonus Challenge

If you finish early, try this extension:

* Add a menu-driven program that lets the user choose between Caesar and Rail Fence encryption.
* Allow the user to enter their own message and shift/rail count.
* Add a function to detect whether the input is valid for a chosen cipher.

---

# What You Learned

In this lab, you explored:

* substitution ciphers
* transposition ciphers
* alphabet wrapping
* message rearrangement patterns
* basic encryption and decryption logic in Python

These are foundational ideas used to understand modern cryptography and cybersecurity.
