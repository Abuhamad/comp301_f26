# SSH in macOS
## 1. Check whether OpenSSH is installed

Open Terminal. macOS normally includes the OpenSSH client by default, so you generally do not need to install anything.

First, confirm that the SSH and SCP executables are available:
```sh
which ssh
which scp
```

You should see paths similar to:
```sh
/usr/bin/ssh
/usr/bin/scp
```

Check the SSH version:

```sh
ssh -V
scp -V
```

Typical output will look something like:

```
OpenSSH_9.xp1, LibreSSL ...
```


## 2. Find the config location

For your own macOS user account, OpenSSH reads:

`/Users/<YourMacUserName>/.ssh/config`


The easiest way to refer to your home directory is:

`$HOME/.ssh/config`


To check whether the configuration file already exists:

```sh
test -f "$HOME/.ssh/config" && echo "Config exists" || echo "Config does not exist"
```

To check whether the `.ssh` directory exists:

```sh
test -d "$HOME/.ssh" && echo ".ssh directory exists" || echo ".ssh directory does not exist"
```

## 3. Create and edit config in Terminal

This creates the .ssh directory if needed, creates the config file if missing, and opens it in the default macOS text editor:

```sh
mkdir -p "$HOME/.ssh"
touch "$HOME/.ssh/config"
open -e "$HOME/.ssh/config"
```

Alternatively, you can edit the file directly in Terminal with nano:
```sh
nano "$HOME/.ssh/config"
```

Paste an entry such as this, changing the values:
```sh
Host comp301
    HostName 192.168.XXX.XXX
    User {WRITE YOUR USERNAME}
```

Save the file and close the editor.

**If you used nano**:

* Press Control + O to save.
* Press Enter to confirm the filename.
* Press Control + X to exit.

Then test the configuration:
```sh
ssh comp301
```

If the configuration is correct, this is equivalent to typing:

ssh {WRITE YOUR USERNAME}@192.168.XXX.XXX

## 4. Copy files with SCP

Once the comp301 SSH alias works, you can copy a local file to the remote computer.

For example:

```sh
scp ./example.txt comp301:/home/{WRITE YOUR USERNAME}/comp301/
```

On macOS, ./example.txt means example.txt in your current Terminal directory.

You can check your current directory with:
```sh
pwd
```

And list its files with:
```sh
ls
```

For example, if the file is on your Desktop:
```sh
scp "$HOME/Desktop/example.txt" comp301:/home/{WRITE YOUR USERNAME}/comp301/
```
