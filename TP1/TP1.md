# Install & Build a Linux Kernel

## Educational Objectives

At the end of this workshop, the student will be able to:

*   Install and configure a Linux virtual machine on a Windows host.
*   Download, configure, compile, and install a custom Linux kernel.
*   Understand the complete Linux build system chain (Makefile, modules, initrd, GRUB).
*   Verify the version and functionality of the compiled kernel.

## Prerequisites

*   Host System: **Windows 10 or 11 (64-bit)**
*   **VirtualBox** or **VMware Workstation Player** installed
*   Linux ISO Image (e.g., **Ubuntu Server 22.04 LTS** or **Debian 12**)
*   Stable Internet Connection
*   Minimum: **4 GB of RAM** and **20 GB of free disk space**

## Workshop Plan

### Step 1 – Virtual Machine Preparation

1.  **Download the Linux ISO image**
    *   Ubuntu: https://ubuntu.com/download/server

2.  **Create a VM in VirtualBox**
    *   Name: LinuxKernelLab
    *   Type: Linux
    *   Version: Ubuntu (64-bit) or Debian (64-bit)
    *   Memory: **4096 MB**
    *   Virtual disk: **20 Go** (VDI format, dynamic)
    *   Boot from the downloaded ISO file

3.  **Install the Linux system in the VM (classic installation).**
    *   Choose a username (e.g., student) and a password.
    *   Install the SSH server if prompted.
    *   At the end, restart into the Linux system.

### Step 2 – Preparation of the Compilation Environment

1.  Connect to the VM (directly or via SSH).
2.  Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

3.  Install the necessary dependencies:

```bash
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev bc git -y
```

- These packages install:
	*   **gcc**: C compiler for the kernel,
	*   **make**: automation tool,
	*   **libncurses**: menuconfig configuration interface,
	*   **bison** and **flex**: syntax analyzers,
	*   **libelf**: management of ELF symbols,
	*   **bc**: calculations necessary for compilation.
### Step 3 – Downloading the Kernel Source Code

1.  Go to the working directory:

```bash
cd /usr/src
```

2.  Download the kernel source code from kernel.org:

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.8.tar.xz
tar -xvf linux-6.8.tar.xz
cd linux-6.8
```

3.  Verify the version:

```bash
make kernelversion
```

The `tar` command decompresses the kernel source files.

### Step 4 – Kernel Configuration

1.  Launch the configurator:

```bash
make menuconfig
```

2.  Navigate the menu and observe the main categories:

    Textual interface (ncurses) with menus:

    *   **Processor type and features**: CPU architecture
    *   **Device Drivers**: peripherals
    *   **File Systems**: ext4, FAT, NTFS
    *   **Networking support**: network options

    Each option can be:
    *   **[Y]** built-in to the kernel,
    *   **[M]** compiled as a module,
    *   **[ ]** disabled.

3.  To save time, we can start from the current configuration (→ reuse the existing kernel's configuration):

```bash
cp /boot/config-$(uname -r) .config
make oldconfig
```

### Step 5 – Kernel Compilation

1.  Compile the kernel:

```bash
make -j$(nproc)
```

`-j$(nproc)`: uses all available CPU cores.

During compilation:
*   **vmlinux**: uncompressed binary image,
*   **bzImage**: compressed image used at startup,
*   **Modules (.ko)**: external drivers.

**This step can take between 20 and 60 minutes depending on the power of the machine.**

2.  Compile and install the modules:

```bash
sudo make modules_install
```

The modules will go into `/lib/modules/<version>/`.

3.  Install the kernel:

```bash
sudo make install
```

This copies:
*   the kernel (`vmlinuz-6.8.x`) into `/boot`,
*   the `System.map` file,
*   the `.config` configuration,
*   and updates the GRUB bootloader.

### Step 6 – Updating the Bootloader (GRUB)

After installation:

```bash
sudo update-grub
```

Restart the machine:

```bash
sudo reboot
```

During reboot, the GRUB menu will display several kernels:
*   the old (generic Ubuntu) kernel,
*   the new personalized kernel (6.8.0-custom).

### Step 7 – Verification

*   After rebooting:

```bash
uname -r
```

The result should display the compiled kernel version (6.8.0 or custom).

*   Verify that everything is working:

```bash
dmesg | head
lsmod | head
```

### Step 8 – Optional Personalization

1.  Add a custom signature to the kernel banner:

```bash
 echo "Student LinuxKernelLab" | sudo tee /etc/motd
```

2.  Modify the `EXTRAVERSION` variable in the `Makefile`:

```
EXTRAVERSION = -student1
```

the kernel will appear as `6.8.0-student1`.

## Possible Extensions (Homework)

*   Compile a minimalist kernel for an **embedded system**.
*   Add a personalized **LKM module** to the compiled kernel.
*   Test **real-time preemption** with a `PREEMPT_RT` kernel.
*   Compare the performance of the generic vs. compiled kernel (`uname -a`, `dmesg`, `time make`).