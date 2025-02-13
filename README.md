![Proxmox logo dark](./misc/screenshots/proxmox_logo_black.png#gh-dark-mode-only)
![Proxmox logo light](./misc/screenshots/proxmox_logo_white.png#gh-light-mode-only)

# Simplify your infrastructure operations!

Proxmox is an open-source virtualization platform that offers a user-friendly web interface for creating and managing virtual machines and containers. It provides advanced features like high availability and backup, supports various storage solutions for flexible infrastructure management, and has a community-driven model that makes it appealing to organizations looking to optimize their IT operations.

![Screenshot](./misc/screenshots/proxmox_06.png)

When installing Proxmox VE, there are two main methods to consider:

- Manual Installation via Terminal (CLI) or Ansible (and Puppet) if you prefer, this method involves installing Proxmox VE on a base operating system using [command-line](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm) ([Debian](https://www.debian.org/distrib/) is the optimal choice). While this is less common for most users, it can be very useful in certain situations where more control over the installation environment is needed.
- Installation via ISO, this is the most straightforward and common way to install Proxmox VE. You [download](https://www.proxmox.com/en/downloads) the Proxmox VE ISO image and boot from it, in order to initiate the installation process. This method provides a graphical installer that guides you through the setup steps.

## Motivation

Recognizing the need for clearer guidance in the installation of Proxmox VE, I created this custom installation guide to provide users with straightforward, step-by-step instructions. My goal is to simplify the process, making it more accessible for individuals of all skill levels. By sharing these tailored steps, I hope to empower users to efficiently set up Proxmox and unlock its full potential in their virtualized environments.

## Environment

If there is an available bare-metal server or a desktop computer (because Proxmox is designed to be installed directly on physical hardware), start by installing first the Debian operating system (the usual standard installation) to prepare an environment for installing Proxmox. Alternatively, you can burn the Proxmox ISO to a USB drive `dd bs=1M conv=fdatasync if=./proxmox-ve_*.iso of=/dev/sdb` and boot from it to begin the installation.

For this guide, I will be using VirtualBox. Please use the following minimum VM settings for optimal performance:

## Network Settings

This [manual](https://www.virtualbox.org/manual/) contains all the essential details you need about VirtualBox. Now, navigate to **Tools** in VirtualBox and choose **Network**.

#### Adapter

**Host-only Networks:** `vboxnet0`  
**IPv4 Address:** `192.168.56.1`  
**IPv4 Network Mask:** `255.255.255.0`  
**IPv6 Address:** `fd00::1`  
**IPv6 Prefix Length:**`64`

This IPv6 address follows the Unique Local Address (ULA) format, which is commonly used for private networks (not routable on the public internet). If you encounter issues saving the values, ensure that you define it in the `/etc/vbox/networks.conf` file first (create the file if it does not exist), then add the following configuration `* fd00::/64` (add additional new lines and entries as needed for each Host-Only Network, for `vboxnet1` add `* fd01::/64` and so on, if you need more).

#### DHCP Server

The DHCP configuration follows the usual procedure, in fact, once enabled, it should be pre-filled for you.

**Enable Server:** `Checked`  
**Server Address:** `192.168.56.100`  
**Server Mask:** `255.255.255.0`  
**Lower Address Bound:** `192.168.56.101`  
**Upper Address Bound:** `192.168.56.254`

## VM

| Name    | Type  | Subtype | Version         | Memory Size | Virtual disk type             | Fixed disk-size | Nested VT-x/AMD-V |
| ------- | ----- | ------- | --------------- | ----------- | ----------------------------- | --------------- | ----------------- |
| Proxmox | Linux | Debian  | Debian (64-bit) | 2048 MB     | VMDK (Pre-allocate Full Size) | 50 GB           | Enabled           |

Check your BIOS settings, if you are unable to enable Nested **VT-x/AMD-V** through the GUI. Additionally (assuming that virtualization is enabled in the BIOS settings), you can use the following command `VBoxManage list vms && VBoxManage modifyvm "Proxmox" --nested-hw-virt on` (to view all available options, use `VBoxManage -h`).

We need to set up two network adapters, one configured as NAT `enp0s3` (which is the default configuration for the first adapter) and the other as Host-only `enp0s8`.

## CLI dual-stack installation

```bash
# Spin your Debian VM and SSH on it as root
$ ssh root@192.168.56.10

# Update
$ apt update
$ apt install -y vim sudo ifupdown2

# Let's add some Vim configuration
$ vim /root/.vimrc
syntax on
set mouse=r
colorscheme murphy

# Add the new entry immediately after the existing root superuser entry
$ vim /etc/sudoers
your_username ALL=(ALL:ALL) ALL # Save and quit :wq!
exit # Or press Ctrl+D

# From now on, use SSH to connect with your own username, rather than relying on root
$ ssh your_username@192.168.56.10
$ sudo su -

# Edit hosts, if your IP address is 192.168.56.10 and your hostname is proxmox, your
# /etc/hosts file might look like this:
$ vim /etc/hosts
# IPv4
127.0.0.1       localhost
192.168.56.10   proxmox.local proxmox

# IPv6
::1             localhost ip6-localhost ip6-loopback
fd00::10        proxmox.local proxmox
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

# Create a static interface (at the bottom of the file)
$ vim /etc/network/interfaces

auto enp0s8
iface enp0s8 inet static
    address 192.168.56.10
    netmask 255.255.255.0
    gateway 192.168.56.254

iface enp0s8 inet6 static
    address fd00::10/64
    gateway fd00::1

# You will need to customize hostnamectl in order to match your own hostname (in this example,
# it is set to proxmox.local, as specified in the /etc/hosts file)
$ hostnamectl set-hostname proxmox.local

# Ensure that at least one non-loopback IP address is returned
$ hostname --ip-address
# fd00::10 192.168.56.10

# Restart the networking unit
$ ifreload -a
# Or
$ systemctl restart networking.service
# Or
$ ifdown enp0s8 && ifup enp0s8
# Or
$ reboot

# You now have two options for SSH access to the box (dual-stack), via IPv4 or IPv6
$ ssh your_username@192.168.56.10
# Or
$ ssh your_username@fd00::10

$ sudo su -

# Check internet access
$ ping -c2 8.8.8.8
# PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
# From 192.168.56.10 icmp_seq=1 Destination Host Unreachable
# From 192.168.56.10 icmp_seq=2 Destination Host Unreachable

# In case of NAT problems (troubleshooting)
$ ip route
# default via 192.168.56.254 dev enp0s8 proto kernel onlink
# 10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15
# 192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.10

# It appears that the default route is incorrectly set to 192.168.56.254 via enp0s8,
# that's the problem, because enp0s8 is part of the host-only network, does not provide
# internet access. The default route should instead point to 10.0.2.2, which is the
# VirtualBox NAT gateway for enp0s3.

# Let's test it first
$ ip route del default via 192.168.56.254 dev enp0s8
$ ip route add default via 10.0.2.2 dev enp0s3
$ ping -c2 8.8.8.8
# PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
# 64 bytes from 8.8.8.8: icmp_seq=1 ttl=255 time=16.7 ms
# 64 bytes from 8.8.8.8: icmp_seq=2 ttl=255 time=14.8 ms

# Issue resolved, to ensure that the changes are persistent, comment out the gateway
# from the enp0s8 configuration
$ vim /etc/network/interfaces
auto enp0s8
iface enp0s8 inet static
    address 192.168.56.10
    netmask 255.255.255.0
    # gateway 192.168.56.254

iface enp0s8 inet6 static
    address fd00::10/64
    gateway fd00::1

# Apply without downtime
$ ifreload -a

# Install some necessary packages
$ apt install -y curl software-properties-common apt-transport-https ca-certificates gnupg2

# Add the repository
$ echo "deb [arch=amd64] http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list

# Add the repository key and verify
$ wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
$ sha512sum /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
7da6fe34168adc6e479327ba517796d4702fa2f8b4f0a9833f5ea6e6b48f6507a6da403a274fe201595edc86a84463d50383d07f64bdde2e3658108db7d6dc87 /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

# Update
$ apt update && apt full-upgrade

# Install the Proxmox VE Kernel
$ apt install -y proxmox-default-kernel
$ systemctl reboot

# Install the Proxmox VE packages
$ ssh your_username@192.168.56.10
$ sudo su -
$ apt install -y proxmox-ve postfix open-iscsi chrony
```

![Screenshot](./misc/screenshots/postfix.png)

Regarding the Postfix configuration, note that chrony can be replaced with any other NTP daemon, however, using `systemd-timesyncd` is not recommended for server systems. The `ntpsec-ntpdate` option may conflict with bringing up networking on boot on certain hardware. It's better to configure packages that require user input during installation based on specific needs.

For networks with a mail server, configuring Postfix as a **satellite system** is advisable. In such setup, the existing mail server acts as the relay host, routing emails sent by Proxmox VE to their final recipients.

If there is uncertainty about what to enter, selecting **local only** and keeping the system name unchanged is a suitable choice. In this instance, I will proceed with a **local only** setup, since this is a test environment and there is no need to configure it on this VM at this time. If you need to modify the configuration in the future, you can do so using the command `dpkg-reconfigure postfix`.

```bash
# Look for the Proxmox port
$ ss -tunelp | grep 8006

# Remove the Linux kernel and update GRUB
$ apt remove -y linux-image-amd64 'linux-image-6.1*'
$ update-grub

# Uninstall os-prober to avoid listing virtual machines in the boot menu and reboot
$ apt remove -y os-prober
$ reboot

```

That's it :smile: Access the UI at [https://192.168.56.10:8006](https://192.168.56.10:8006) or [https://[fd00::10]:8006](https://[fd00::10]:8006). If this is a fresh installation without users, select the PAM authentication realm and login using the root user account.

## ISO installation

The graphical installation of Proxmox VE using the ISO is a straightforward process. After booting from the installation media, select "Install Proxmox VE (Graphical)" from the boot menu. The installer will guide you through essential steps, including disk partitioning, basic system configurations (e.g., timezone, language, network settings), and package installation. This user-friendly method is recommended for both new and experienced users, typically taking just a few minutes to complete.

| Graphical UI 01                              | Graphical UI 02                              | Graphical UI 03                              | Graphical UI 04                              | Graphical UI 05                              |
| -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- |
| ![Screenshot](./misc/screenshots/GUI_01.png) | ![Screenshot](./misc/screenshots/GUI_02.png) | ![Screenshot](./misc/screenshots/GUI_03.png) | ![Screenshot](./misc/screenshots/GUI_04.png) | ![Screenshot](./misc/screenshots/GUI_05.png) |

| Graphical UI 06                              | Graphical UI 07                              | Graphical UI 08                              | Graphical UI 09                              | Graphical UI 10                              |
| -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- |
| ![Screenshot](./misc/screenshots/GUI_06.png) | ![Screenshot](./misc/screenshots/GUI_07.png) | ![Screenshot](./misc/screenshots/GUI_08.png) | ![Screenshot](./misc/screenshots/GUI_09.png) | ![Screenshot](./misc/screenshots/GUI_10.png) |

Alternatively, as illustrated in the example that follows, I make use of the **Terminal UI**, which functions in the same way as the **Graphical**.

| Terminal UI 01                               | Terminal UI 02                               | Terminal UI 03                               | Terminal UI 04                               | Terminal UI 05                               |
| -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- |
| ![Screenshot](./misc/screenshots/TUI_01.png) | ![Screenshot](./misc/screenshots/TUI_02.png) | ![Screenshot](./misc/screenshots/TUI_03.png) | ![Screenshot](./misc/screenshots/TUI_04.png) | ![Screenshot](./misc/screenshots/TUI_05.png) |

| Terminal UI 06                               | Terminal UI 07                               | Terminal UI 08                               | Terminal UI 09                               | Terminal UI 10                               |
| -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- | -------------------------------------------- |
| ![Screenshot](./misc/screenshots/TUI_06.png) | ![Screenshot](./misc/screenshots/TUI_07.png) | ![Screenshot](./misc/screenshots/TUI_08.png) | ![Screenshot](./misc/screenshots/TUI_09.png) | ![Screenshot](./misc/screenshots/TUI_10.png) |

The installation is complete. Remove the ISO file from the VM to start it successfully, otherwise it will be rebooted from the ISO again.

![Screenshot](./misc/screenshots/vb_remove_iso.png)

Now start the VM, you can of course login via CLI to carry out any necessary checks or adjustments.

| Start the VM                                     | VM Running (you can also login from CLI)         |
| ------------------------------------------------ | ------------------------------------------------ |
| ![Screenshot](./misc/screenshots/proxmox_01.png) | ![Screenshot](./misc/screenshots/proxmox_02.png) |

After completing the installation, you'll be ready to explore your new Proxmox environment. Access the web interface by navigating to [https://192.168.56.10:8006](https://192.168.56.10:8006) and log in using the root credentials you created during installation. Take time to familiarize yourself with the dashboard, explore virtual machine creation options, and configure your network and storage settings. The Proxmox UI, provides a powerful and intuitive platform for managing your virtualization infrastructure.

| Proxmox UI                                       | Proxmox UI                                       | Proxmox UI                                       | Proxmox UI                                       |
| ------------------------------------------------ | ------------------------------------------------ | ------------------------------------------------ | ------------------------------------------------ |
| ![Screenshot](./misc/screenshots/proxmox_03.png) | ![Screenshot](./misc/screenshots/proxmox_04.png) | ![Screenshot](./misc/screenshots/proxmox_05.png) | ![Screenshot](./misc/screenshots/proxmox_06.png) |

## Dual-stack networking for the ISO installation

Follow the example below to enable dual-stack support for IPv6, by editing the `/etc/network/interfaces` file (as `vim` is unavailable in this situation, we can use either `vi` or `nano` instead):

```bash
# Spin your Debian VM and SSH on it
$ ssh root@192.168.56.10

# Update
$ nano /etc/network/interfaces

# Include only the IPv6 section
auto lo
iface lo inet loopback

iface enp0s8 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.56.10/24
        gateway 192.168.56.254
        bridge-ports enp0s8
        bridge-stp off
        bridge-fd 0

# IPv6
iface vmbr0 inet6 static
        address fd00::10/64
        gateway fd00::1

iface enp0s3 inet manual


source /etc/network/interfaces.d/*

# If ifupdown2 is installed (default in Proxmox VE 7.0 and later), you can apply the changes live without rebooting, this command reloads all network interfaces with the updated configuration, it's of course safe and avoids disrupting running VMs unnecessarily.
ifreload -a

# Or restart the networking unit
$ systemctl restart networking.service

# Or reboot
$ reboot

```

Next, test the connection by running the command `ping -6 fd00::10 -c2`. Additionally, try to SSH into the box using `ssh root@fd00::10`. Finally, test it in your browser by navigating to [https://[fd00::10]:8006](https://[fd00::10]:8006).

| Dual-stack 01                                       | Dual-stack 02                                       | Dual-stack 03                                       | Dual-stack 04                                       |
| --------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------- |
| ![Screenshot](./misc/screenshots/dual_stack_01.png) | ![Screenshot](./misc/screenshots/dual_stack_02.png) | ![Screenshot](./misc/screenshots/dual_stack_03.png) | ![Screenshot](./misc/screenshots/dual_stack_04.png) |

## Proxmox technologies

Proxmox Virtual Environment (PVE) is built on a diverse set of technologies, combining multiple programming languages and open-source components. The core of Proxmox is primarily written in [Perl](https://www.perl.org), with [Rust](https://www.rust-lang.org) being utilized for certain aspects of development. The platform is based on a modified [Debian](https://www.debian.org) LTS kernel, leveraging Linux technologies and [GNU](https://www.gnu.org) userland. For virtualization, Proxmox employs [KVM (Kernel-based Virtual Machine)](https://linux-kvm.org/page/Main_Page) for full virtualization and [LXC (Linux Containers)](https://linuxcontainers.org) for container-based virtualization. The web-based management interface is constructed using [JavaScript](https://ecma-international.org/publications-and-standards/standards/ecma-262/), [HTML](https://www.w3.org/html/) and [CSS](https://www.w3.org/Style/CSS/). Additionally, Proxmox incorporates [PostgreSQL](https://www.postgresql.org) for database management, indicating the use of [SQL](https://www.iso.org/standard/76583.html). This combination of technologies enables Proxmox to offer a comprehensive, open-source virtualization platform with a user-friendly interface and robust functionality.

Proxmox Virtual Environment (PVE) offers multiple interfaces for management and interaction, catering to different user preferences and needs.

- **Web Interface:** The primary method for managing Proxmox VE, accessible through a [web browser](https://pve.proxmox.com/wiki/Graphical_User_Interface)
- **Command Line Interface (CLI):** Proxmox VE provides a comprehensive CLI for advanced users and system administrators. This interface offers intelligent tab completion and full [UNIX man page documentation](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
- **RESTful API:** Proxmox VE utilizes a [RESTful API](https://pve.proxmox.com/wiki/Proxmox_VE_API), which allows for easy integration with third-party management tools and custom hosting environments

## License

MIT

## Disclaimer

This project is distributed FREE & WITHOUT ANY WARRANTY. Report any bugs or suggestions here as an [issue](https://github.com/ncklinux/proxmox/issues/new).

## Contributing

Please read the [contribution](https://github.com/ncklinux/.github/blob/main/CONTRIBUTING.md) guidelines.

## Commit Messages

This repository follows the [Conventional Commits](https://www.conventionalcommits.org) specification, the commit message should never exceed 100 characters and must be structured as follows:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

## Powered by

<img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/unix/unix-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/linux/linux-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/debian/debian-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/bash/bash-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/ssh/ssh-original-wordmark.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/perl/perl-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/rust/rust-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/javascript/javascript-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/html5/html5-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/css3/css3-original.svg" /><img height="33" style="margin-right: 3px;" src="https://cdn.jsdelivr.net/gh/devicons/devicon@latest/icons/vim/vim-original.svg" />
