# Lab 14: Race Conditions and TOCTOU

## Objectives

By the end of this lab, you will be able to:

1. Reproduce a race condition on a shared counter and measure the lost updates.
2. Remove the race with a mutex and verify exact results.
3. Exploit a time-of-check to time-of-use window with a symbolic link swap, and count the unauthorized reads.

## Prerequisites

- SSH access to your assigned course VM (Linux). No administrator rights are required.
- Python 3 with the standard library only.
- Chapter 8 notes, Section 7: race conditions, target of evaluation, TOCTOU, and synchronization defenses.

## Before you start

### Background

A race condition occurs when concurrently executing processes or threads access a shared resource at the same time, and the outcome depends on timing. The shared resource is the target of evaluation (TOE). A TOCTOU error appears when the target's state changes between the time-of-check and the time-of-use, invalidating the check.

| Concept from the lecture | Where you observe the concept in this lab |
| --- | --- |
| Shared resource under concurrent access | Part 1: four threads increment one counter |
| Timing-dependent outcome | Part 1: the lost count changes from run to run |
| Mutex synchronization | Part 2: the lock argument produces exact totals |
| TOCTOU window | Part 3: the attacker swaps the link between realpath and open |
| Unauthorized access from TOCTOU | Part 3: the victim reads the secret file hundreds of times |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab14`
  - go to folder: `cd ~/comp301/lab14`

## Part 1: The racy counter

Write the counter program:

```bash
cat > race.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 14 counter race. Run with no argument for the racy version, or with
the argument lock for the mutex-protected version. The sleep(0) call forces
a thread switch between the read and the write, the exact window a race
condition needs."""
import sys
import threading
import time

counter = 0
lock = threading.Lock()
THREADS = 4
ROUNDS = 2000


def work(use_lock):
    global counter
    for _ in range(ROUNDS):
        if use_lock:
            with lock:
                x = counter
                x += 1
                time.sleep(0)
                counter = x
        else:
            x = counter
            x += 1
            time.sleep(0)
            counter = x


def main():
    global counter
    use_lock = len(sys.argv) > 1 and sys.argv[1] == "lock"
    threads = [threading.Thread(target=work, args=(use_lock,))
               for _ in range(THREADS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    expected = THREADS * ROUNDS
    print(f"expected {expected}, got {counter}, lost {expected - counter}")
    counter = 0


if __name__ == "__main__":
    main()
PYEOF
```

Each thread runs 2,000 increments across four threads, so the correct total is 8,000. The `sleep(0)` call yields the processor between the read and the write, the same window that exists without warning on a multi-core system. Run the racy version three times:

```bash
python3 race.py
python3 race.py
python3 race.py
```

The totals fall far below 8,000 and vary between runs. Two threads read the same value, both add one, and both write the same result, so one increment vanishes. Record the three lost counts.

## Part 2: The mutex fix

Run the protected version twice:

```bash
python3 race.py lock
python3 race.py lock
```

Every run reports `lost 0`. The lock admits one thread at a time into the read-modify-write sequence, which is the synchronization defense from the notes. Record the totals.

## Part 3: TOCTOU with a symbolic link

Write the demonstration:

```bash
cat > toctou.py << 'PYEOF'
#!/usr/bin/env python3
"""Lab 14 TOCTOU. A victim thread checks a symlink target before reading, and
an attacker thread swaps the link between the check and the read."""
import os
import threading

os.makedirs("toctou", exist_ok=True)
with open("toctou/safe.txt", "w") as fh:
    fh.write("public data\n")
with open("toctou/secret.txt", "w") as fh:
    fh.write("TOP SECRET\n")

LINK = "toctou/current"
SECRET = os.path.realpath("toctou/secret.txt")
if os.path.lexists(LINK):
    os.unlink(LINK)
os.symlink("safe.txt", LINK)

stop = False
leaks = 0


def attacker():
    while not stop:
        try:
            os.unlink(LINK)
        except FileNotFoundError:
            pass
        os.symlink("safe.txt", LINK)
        try:
            os.unlink(LINK)
        except FileNotFoundError:
            pass
        os.symlink("secret.txt", LINK)


def victim():
    global leaks
    for _ in range(20000):
        # time-of-check: refuse the secret file
        if os.path.realpath(LINK) == SECRET:
            continue
        # time-of-use: open and read the file
        try:
            with open(LINK) as f:
                data = f.read()
        except FileNotFoundError:
            continue
        if "TOP SECRET" in data:
            leaks += 1


t = threading.Thread(target=attacker)
t.start()
victim()
stop = True
t.join()
print(f"leaked secret reads: {leaks}")
PYEOF

python3 toctou.py
python3 toctou.py
```

The victim checks the link target and refuses the secret file, yet the victim reads the secret hundreds of times. The attacker swaps the link after the check and before the open, exactly the gap the TOCTOU figure in the notes shows. Record both leaked counts.

The proper defense opens the file first and checks the opened file descriptor, or refuses symlinks entirely with `os.open(LINK, os.O_NOFOLLOW)`. Both approaches remove the window between check and use.

## Part 4: Analysis

Answer in one sentence each, from what you observed:

1. Why did the lost count in Part 1 change between runs?
2. In Part 3, which two calls form the time-of-check and the time-of-use?
3. Why does checking the path before opening fail, while checking the opened file descriptor succeeds?

## Assignment

Create a `submission.txt` file with the following entries:

- `race-lost-1: <lost count from the first racy run>`
- `race-lost-2: <lost count from the second racy run>`
- `race-lost-3: <lost count from the third racy run>`
- `race-lock-lost: <lost count from a lock run>`
- `toctou-leaks-1: <leaked reads from the first run>`
- `toctou-leaks-2: <leaked reads from the second run>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab14`) and name the file exactly `submission.txt`.

## Useful References and Resources

- CWE-362, "Concurrent Execution using Shared Resource with Improper Synchronization (Race Condition)."
- CWE-367, "Time-of-check Time-of-use (TOCTOU) Race Condition."
- Python documentation for `threading.Lock` and the `os.O_NOFOLLOW` flag.
- Chapter 8 notes, Section 7: the TOCTOU sequence diagram and the synchronization defenses.

## Optional Challenges

- Rewrite the victim to use `os.open` with `os.O_NOFOLLOW`, rerun the attack, and record the new leaked count.
- Remove the `sleep(0)` call, run the racy counter ten times, and explain why the race still appears on some runs and vanishes on others.
