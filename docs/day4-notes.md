
Day 4 — Ubuntu VM Installation

Today i started with istallation process of Ubuntu VM. First i downloaded Ubuntu 26.04 ISO file from crom and started installation process in UTM. I used Virtualize → Linux with the Ubuntu ISO, RAM: 2048 MB, CPU: default, Storage: 25 GB, dynamic. During the installation process the system crashed many times with a firmware error related to TPM/measured-boot logging, and the VM stopped completely.

After that i deleted the entire VM and started again, but this time i switched from Ubuntu 26.04 to **Ubuntu 24.04.4 LTS** because the problem was with the compatibility of Ubuntu 26.04 with the virtualization setup. I downloaded the `ubuntu-24.04.4-live-server-arm64.iso` file and continued the installation with the same configuration.

Before the last boot Ubuntu asked me to remove the installation medium before continuing, unlike Windows' silent boot loop trap. So i ejected the ISO properly through UTM and after that the system booted perfectly and Ubuntu was installed.

I learned that newer is not always better for compatibility, and mature LTS releases can work better with virtualization tools.





