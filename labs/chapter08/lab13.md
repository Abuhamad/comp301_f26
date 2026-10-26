# Lab 13: Memory Vulnerabilities in C

## Objectives

By the end of this lab, you will be able to:

1. Trigger a buffer overflow in a C program, observe how the default compiler protection responds, and inspect the crash in `gdb`.
2. Demonstrate an integer overflow at the 16-bit boundary from the notes, -32,768 to 32,767.
3. Grow a process's memory footprint with a heap leak, measure the growth, and flatten the footprint with `free`.

## Prerequisites

- SSH access to your assigned course VM (Linux). The leak measurement reads `/proc/self/status`, which exists on Linux.
- The `gcc` compiler and the `gdb` debugger. Part 1 verifies both.
- Chapter 8 notes, Section 6: buffer overflow, integer overflow, memory leak, and pointer dereferencing.

## Before you start

### Background

A buffer overflow writes past the memory a buffer owns, and improper input handling enables the condition. An integer overflow stores a result outside the range of the integer type. A memory leak allocates heap memory without ever releasing the allocation, and the programmer manages heap memory rather than the operating system.

| Concept from the lecture | Where you observe the concept in this lab |
| --- | --- |
| Buffer overflow into adjacent memory | Part 2: a 64-byte argument into a 16-byte buffer |
| Compiler protection against overwrites | Part 2: the default build aborts with a stack smashing message |
| Integer range limits | Part 3: `SHRT_MAX` + 1 wraps to -32,768 |
| Heap leaks grow memory use | Part 4: VmRSS climbs round after round |
| The programmer frees heap memory | Part 4: the `free` argument keeps VmRSS flat |

### Set-up and Implementation

- On the VM, create a folder for this lab in the class folder in the home directory.
  - create folder: `mkdir -p ~/comp301/lab13`
  - go to folder: `cd ~/comp301/lab13`

## Part 1: Toolchain check

```bash
gcc --version
gdb --version
```

If either command is missing on a machine where you hold administrator rights, install the tools with `sudo apt-get update && sudo apt-get install -y gcc gdb`. On the course VM without administrator rights, report the missing tool in your submission and pair with a classmate.

## Part 2: Buffer overflow

Write the vulnerable program:

```bash
cat > bof.c << 'CEOF'
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv) {
    char buf[16];
    if (argc < 2) {
        printf("usage: %s WORD\n", argv[0]);
        return 1;
    }
    strcpy(buf, argv[1]);
    printf("buf holds: %s\n", buf);
    return 0;
}
CEOF
```

The buffer holds 16 bytes, and `strcpy` never checks the input length. Build with the default protections and run twice:

```bash
gcc -o bof bof.c
./bof hello
./bof $(python3 -c "print('A' * 64)")
```

The short word works. The 64-byte input triggers the compiler's stack protector, and the run ends with a `stack smashing detected` message and an abort. The protection caught the overwrite before the overwritten data could redirect execution.

Rebuild without the protector and repeat:

```bash
gcc -g -fno-stack-protector -o bof2 bof.c
./bof2 $(python3 -c "print('A' * 64)")
echo $?
```

The unprotected build crashes with a segmentation fault. Inspect the crash by handing gdb the same 64-byte argument, generated on the shell before gdb starts:

```bash
gdb -q --args ./bof2 $(python3 -c "print('A' * 64)")
```

At the gdb prompt:

```gdb
run
info registers rip
quit
```

The instruction pointer contains `0x4141414141414141`, which is the letter A in hexadecimal. Your input overwrote the saved return address, and the program tried to execute at an address you chose. That control is the reason buffer overflows lead to code execution rather than simple crashes. Record the `rip` value.

## Part 3: Integer overflow

Write the overflow program:

```bash
cat > iof.c << 'CEOF'
#include <stdio.h>
#include <limits.h>

int main(void) {
    short x = SHRT_MAX;
    printf("SHRT_MAX = %d\n", SHRT_MAX);
    x = x + 1;
    printf("after +1: %d\n", x);
    return 0;
}
CEOF

gcc -o iof iof.c
./iof
```

A signed 16-bit integer holds -32,768 to 32,767, exactly the range from the notes. Adding 1 to 32,767 wraps the value to -32,768, and any computation built on the wrapped value is wrong. Record both printed values.

## Part 4: Memory leak

Write the leaking program:

```bash
cat > leak.c << 'CEOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define CHUNK (1024 * 1024)

static long rss_kb(void) {
    FILE *f = fopen("/proc/self/status", "r");
    char line[128];
    long kb = -1;
    if (f == NULL) {
        return -1;
    }
    while (fgets(line, sizeof line, f)) {
        if (strncmp(line, "VmRSS:", 6) == 0) {
            sscanf(line + 6, "%ld", &kb);
        }
    }
    fclose(f);
    return kb;
}

int main(int argc, char **argv) {
    int do_free = argc > 1 && strcmp(argv[1], "free") == 0;
    for (int round = 1; round <= 200; round++) {
        char *p = malloc(CHUNK);
        if (p == NULL) {
            printf("malloc failed at round %d\n", round);
            return 1;
        }
        memset(p, 1, CHUNK);
        if (round % 20 == 0) {
            printf("round %d: VmRSS = %ld kB\n", round, rss_kb());
        }
        if (do_free) {
            free(p);
        }
        sleep(1);
    }
    return 0;
}
CEOF

gcc -o leak leak.c
```

Run the leaking version and watch VmRSS, the resident memory of the process:

```bash
./leak
```

Each report covers 20 rounds, and each round leaks one megabyte, so VmRSS climbs by roughly 20,000 kB per report. Stop the program with `Ctrl+C` after four or five reports. Record the first and last VmRSS values you saw.

Run the fixed version, which frees every allocation:

```bash
./leak free
```

VmRSS stays nearly flat. The heap belongs to the programmer: nothing reclaims the memory until `free` runs or the process exits, which matches the stack versus heap table from the notes.

## Part 5: Analysis

Answer in one sentence each, from what you observed:

1. What did the default build of `bof.c` do that the unprotected build did not?
2. Why did the `gdb` register dump show the pattern `41`?
3. Why does a leaking server process hurt the whole host rather than the leaking process alone?

## Assignment

Create a `submission.txt` file with the following entries:

- `bof-protected-result: <the message from the default build with 64 A's>`
- `bof-rip: <the rip value gdb reported for bof2>`
- `iof-shrt-max: <the printed SHRT_MAX value>`
- `iof-after-plus-one: <the printed value after the increment>`
- `leak-first-rss: <the first VmRSS value in kB>`
- `leak-last-rss: <the last VmRSS value before you stopped the program>`
- `leak-free-rss: <the last VmRSS value from the free run>`
- `analysis-1: <one sentence>`
- `analysis-2: <one sentence>`
- `analysis-3: <one sentence>`

## Submission

- Place `submission.txt` in this lab folder (`~/comp301/lab13`) and name the file exactly `submission.txt`.

## Useful References and Resources

- CWE-120, "Classic Buffer Overflow," CWE-190, "Integer Overflow or Wraparound," and CWE-401, "Missing Release of Memory after Effective Lifetime."
- `man gcc` for `-fno-stack-protector`, and `man gdb` for `run` and `info registers`.
- The proc(5) manual page for `/proc/self/status` and the VmRSS field.
- Chapter 8 notes, Section 6: improper input handling and the stack versus heap comparison.

## Optional Challenges

- Run `valgrind --leak-check=full ./leak` for a few rounds if valgrind is installed, and read the definitely lost line in the summary.
- Modify `iof.c` to check for the overflow before the addition with `if (x == SHRT_MAX)`, and print a warning instead of wrapping.
