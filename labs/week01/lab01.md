Yes. Since the students already have the **SSH host configuration in their `~/.ssh/config` file**, the lab should avoid exposing or requiring usernames/IP addresses and teach them the simpler `ssh comp301` workflow. I would also update the `scp` commands to use the `comp301` host alias.

# Lab 01 — SSH and Secure File Transfer

## Overview

In this lab, you will learn how to:

* Connect to the COMP 301 server using **SSH**.
* Navigate and manage files on a remote Linux machine.
* Create a directory in your remote home directory.
* Create and edit a text file remotely.
* Copy a file from the remote machine to your local computer.
* Create a file on your local computer.
* Copy a local file to the remote machine using **SCP**.

Your SSH configuration has already been provided. Therefore, you **do not need to enter your username or the server IP address**.

You will use the following commands:

* `ssh`
* `pwd`
* `ls`
* `mkdir`
* `cd`
* `nano`
* `cat`
* `scp`
* `exit`

---

# Part 1 — Connect to COMP 301

Open a terminal on your local computer.

Connect to the COMP 301 server using:

```bash
ssh comp301
```

Because your SSH configuration is already set up, the `comp301` entry contains the necessary username and server information.

If this is your first time connecting to the server, you may see a message asking whether you trust the host.

Type:

```text
yes
```

Enter your password when prompted.

### Verify Your Connection

Once connected, run:

```bash
pwd
```

This displays your current working directory.

Then run:

```bash
ls
```

This displays the files and directories in your home directory.

---

# Part 2 — Create Your COMP 301 Directory

You will create a directory called `comp301` inside your remote home directory.

Run:

```bash
mkdir ~/comp301
```

Verify that the directory was created:

```bash
ls
```

You should see:

```text
comp301
```

Now enter the directory:

```bash
cd ~/comp301
```

Verify your current location:

```bash
pwd
```

You should see a path similar to:

```text
/home/YOUR_USERNAME/comp301
```

---

# Part 3 — Create a File on the Remote Machine

You will now create a text file **on the remote COMP 301 machine**.

Create a file called:

```text
remote.txt
```

Use the `nano` text editor:

```bash
nano remote.txt
```

Enter the following text:

```text
This file was created on the COMP 301 remote machine.

My name is YOUR_NAME.

I am learning how to use SSH and SCP.
```

Replace `YOUR_NAME` with your name.

### Save the File

In `nano`:

1. Press **Ctrl + O** to save the file.
2. Press **Enter** to confirm the filename.
3. Press **Ctrl + X** to exit.

Verify that the file exists:

```bash
ls
```

You should see:

```text
remote.txt
```

Display the contents of the file:

```bash
cat remote.txt
```

Verify that the text you entered is displayed.

---

# Part 4 — Copy the Remote File to Your Local Computer

You will now transfer `remote.txt` from the **remote COMP 301 machine to your local computer**.

First, leave the remote machine:

```bash
exit
```

You should now be back at your **local terminal**.

You can verify this by running:

```bash
pwd
```

Now use `scp` to copy the file from COMP 301 to your current local directory:

```bash
scp comp301:~/comp301/remote.txt .
```

The `.` at the end means:

> Copy the file to my current local directory.

Verify that the file was copied:

```bash
ls
```

You should see:

```text
remote.txt
```

Display the file:

```bash
cat remote.txt
```

You have now successfully transferred a file:

**Remote COMP 301 → Local computer**

---

# Part 5 — Create a File on Your Local Computer

Now create a second file on your **local computer**.

Create a file called:

```text
local.txt
```

Use `nano`:

```bash
nano local.txt
```

Enter the following:

```text
This file was created on my local computer.

My name is YOUR_NAME.

I am learning how to transfer files between my computer and a remote server.
```

Replace `YOUR_NAME` with your name.

Save the file:

1. Press **Ctrl + O**
2. Press **Enter**
3. Press **Ctrl + X**

Verify the file:

```bash
ls
```

Then display its contents:

```bash
cat local.txt
```

---

# Part 6 — Copy the Local File to COMP 301

You will now transfer `local.txt` from your **local computer to the remote COMP 301 machine**.

Run:

```bash
scp local.txt comp301:~/comp301/
```

Enter your password if prompted.

The command transfers:

```text
local.txt
    ↓
Your local computer
    ↓
COMP 301
    ↓
~/comp301/
```

---

# Part 7 — Verify the File on COMP 301

Connect to COMP 301 again:

```bash
ssh comp301
```

Navigate to your COMP 301 directory:

```bash
cd ~/comp301
```

List the files:

```bash
ls
```

You should now see:

```text
local.txt
remote.txt
```

Display the contents of `local.txt`:

```bash
cat local.txt
```

You should see the text that you created on your local computer.

You have now successfully transferred a file:

**Local computer → Remote COMP 301**

---

# Part 8 — Verify Both Files

Run:

```bash
ls -l
```

You should see both files.

Then display both files:

```bash
cat remote.txt
cat local.txt
```

Your `comp301` directory should now contain:

```text
comp301/
├── local.txt
└── remote.txt
```

---

# What You Learned

| Command       | Purpose                                |
| ------------- | -------------------------------------- |
| `ssh comp301` | Connect to the COMP 301 remote machine |
| `pwd`         | Display the current directory          |
| `ls`          | List files and directories             |
| `mkdir`       | Create a directory                     |
| `cd`          | Change directories                     |
| `nano`        | Create/edit a text file                |
| `cat`         | Display the contents of a file         |
| `scp`         | Securely copy files between computers  |
| `exit`        | Close the remote SSH session           |

### SSH

To connect to COMP 301:

```bash
ssh comp301
```

### SCP: Remote → Local

```bash
scp comp301:~/comp301/remote.txt .
```

### SCP: Local → Remote

```bash
scp local.txt comp301:~/comp301/
```

Notice that **no username or IP address is required**. The SSH configuration provided for the course handles this information.

---

# Lab Checklist

Before finishing the lab, make sure you have completed all of the following:

* [ ] Connected to COMP 301 using `ssh comp301`.
* [ ] Created `~/comp301` on the remote machine.
* [ ] Created `remote.txt` on the remote machine.
* [ ] Added your name and text to `remote.txt`.
* [ ] Copied `remote.txt` from COMP 301 to your local machine.
* [ ] Created `local.txt` on your local machine.
* [ ] Added your name and text to `local.txt`.
* [ ] Copied `local.txt` from your local machine to `~/comp301`.
* [ ] Connected to COMP 301 again.
* [ ] Verified that both files exist in `~/comp301`.
* [ ] Displayed the contents of both files using `cat`.

---

# Submission

Your `~/comp301` directory on the COMP 301 server should contain:

```text
comp301/
├── local.txt
└── remote.txt
```

Make sure both files contain the requested information.

**Do not delete the `comp301` directory or the two files until instructed to do so.**

