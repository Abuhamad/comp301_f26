# SSH in Windows

## 1. Check whether OpenSSH is installed

Open **PowerShell**. Administrator rights are recommended for checking/removing Windows capabilities, although normal PowerShell is enough to use SSH and edit your personal config.

First, confirm that the SSH executable is available:

```powershell
Get-Command ssh -ErrorAction SilentlyContinue
Get-Command scp -ErrorAction SilentlyContinue
```

If installed and on your `PATH`, you should see commands such as:

```text
CommandType     Name     Source
-----------     ----     ------
Application     ssh.exe  C:\Windows\System32\OpenSSH\ssh.exe
Application     scp.exe  C:\Windows\System32\OpenSSH\scp.exe
```

Check the Windows optional-feature state:

```powershell
Get-WindowsCapability -Online |
    Where-Object Name -like 'OpenSSH*' |
    Select-Object Name, State
```

Typical output:

```text
Name                                      State
----                                      -----
OpenSSH.Client~~~~0.0.1.0                 Installed
OpenSSH.Server~~~~0.0.1.0                 NotPresent
```

Interpret it as follows:

| Result | Meaning | What you need to do |
|---|---|---|
| `OpenSSH.Client` = `Installed` | You can use `ssh` and `scp` from PowerShell | Nothing; proceed with config |
| `OpenSSH.Client` = `NotPresent` | SSH/SCP client is not installed | Install it |
| `OpenSSH.Server` = `Installed` | Your PC may accept incoming SSH connections | Keep only if you need that |
| `OpenSSH.Server` = `NotPresent` | Windows does not run an SSH server | Normal for a client-only PC |

Microsoft’s PowerShell guidance uses `Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'` to inspect the client/server capability state.  [learn.microsoft](https://learn.microsoft.com/en-us/troubleshoot/windows-server/system-management-components/cant-install-openssh-features)

## Install or uninstall it

Run these commands in an **Administrator PowerShell** window.

### Install the client

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

After installation, close and reopen PowerShell, then verify:

```powershell
ssh -V
scp -V
```


### [SKIP - NOT NEEDED NOW] Uninstall the client

Only do this if you no longer want `ssh` or `scp` on the Windows computer:

```powershell
Remove-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

Verify removal:

```powershell
Get-WindowsCapability -Online |
    Where-Object Name -like 'OpenSSH*' |
    Select-Object Name, State
```


## 2. Find the config location

For your own Windows user account, OpenSSH reads:

```text
C:\Users\<YourWindowsUserName>\.ssh\config
```

In PowerShell, that is:

```powershell
$HOME\.ssh\config
```

To see the exact full path on your computer:

```powershell
Join-Path $HOME '.ssh\config'
```

To check whether it already exists:

```powershell
Test-Path (Join-Path $HOME '.ssh\config')
```

To open it if it exists:

```powershell
notepad (Join-Path $HOME '.ssh\config')
```


## 3. Create and edit config in PowerShell

This creates the `.ssh` directory if needed, creates the config file if missing, and opens it in Notepad:

```powershell
$sshDir = Join-Path $HOME '.ssh'
$config = Join-Path $sshDir 'config'

New-Item -ItemType Directory -Path $sshDir -Force | Out-Null
New-Item -ItemType File -Path $config -Force | Out-Null

notepad $config
```

Paste an entry such as this, changing the values:

```sshconfig
Host comp301
    HostName 192.168.101.200
    User {WRITE YOUR USERNAME}
```

Save it, close Notepad, then test:

```powershell
ssh comp301
```

And copy files, an example:

```powershell
scp .\example.txt comp301:/home/{WRITE YOUR USERNAME}/comp301/
```
