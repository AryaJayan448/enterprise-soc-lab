# Day 7

Today i worked on SSH in my Ubuntu VM.

### What I set up

First i checked SSH and found that the SSH server was not installed. So i installed it using:

`sudo apt install openssh-server`

After that i started and enabled SSH using:

`sudo systemctl enable --now ssh`

At first i saw `inactive (dead)` when i checked SSH and thought it was not working. But on Ubuntu 24.04 this does not always mean SSH is broken because `ssh.socket` can listen for the connection and start SSH when it is needed.

After that i connected to the Ubuntu VM using SSH.

### What I found in auth.log

I checked `/var/log/auth.log` and found different SSH login messages.

`Accepted password` means the login was successful.

`Failed password for <user>` means the password was wrong for that user.

`Failed password for invalid user <user>` means the user does not exist.

I also saw my own `sudo` commands in `auth.log`. The log showed the full command that i used.

The first successful SSH login came from the Ubuntu VM itself with the IP `192.168.64.5`. The failed SSH attempts came from `192.168.64.1`, which was my Mac.

### What went wrong

My first `grep` command did not find anything because of the letter case. The log had `Failed password`, but my search was different.

I fixed it by using:

`grep -i`

The `-i` makes the search ignore upper and lower case.

### Linux and Windows

Linux and Windows have similar login events.

`Accepted password` in Linux is similar to Windows Event ID 4624 because both show a successful login.

`Failed password` in Linux is similar to Windows Event ID 4625 because both show a failed login.

`invalid user` in Linux is similar to `0xC0000064` in Windows because both mean the user does not exist.

Windows gives structured fields like Event ID, `Sub Status` and `Logon Type`. Linux gives a text log line, so we normally search it using `grep`. This also affects how a SIEM reads and parses the logs.

### Message repeated 2 times

I also saw `message repeated 2 times`.

This is important because it means the same log message happened more than once. If we only count the lines, we might think there was only one failed login when there were actually more.

The main thing i learned today is how SSH logs work in Linux and how they are similar to Windows login events.

