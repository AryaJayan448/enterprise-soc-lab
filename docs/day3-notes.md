DAY 3-Windows VM setup

I configured the Windows 11 Pro VM with 4GB RAM and 2 CPU cores because this provides enough resources for Windows and basic SOC lab activities while avoiding excessive resour

ce usage on my Mac. I selected a 64GB dynamic disk so the VM has enough storage for Windows, security tools, logs, and future lab work, while the disk only uses physical storage as needed.

During installation, I encountered a boot loop where the VM kept returning to the Windows installation process instead of booting from the installed system. I fixed this by deleting the leftover disk partitions and performing a clean installation. After Windows finished installing, I ejected the Windows ISO before the reboot, allowing the VM to boot correctly from the virtual disk.

The Windows 11 VM is now running successfully and will serve as the first endpoint in my SOC lab for generating Windows security events and sending logs to the SIEM platforms.
