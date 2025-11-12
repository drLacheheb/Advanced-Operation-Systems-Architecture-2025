# Add a Custom System Call

## Implementation of a New System Call

**Objective:** Modify the kernel sources to add a new system call that offers a simple functionality to the user space.

### Theoretical Context

**System Calls** are the secure interface that allows a user program to request a privileged service from the kernel (such as `read()`, `write()`, or `fork()`). Adding a system call involves modifying the kernel's function table.

Adding a new system call (syscall) to the Linux kernel is a complex procedure that requires modifying the kernel sources, recompiling it, and then restarting with the new kernel. Here are the detailed steps to achieve this, with a simple example.

**Warning:** Any error in modifying and compiling the kernel can make your system unstable or unusable. It is strongly recommended to use a virtual machine for this operation.

### Preparation

#### Step 1: Obtain the kernel sources

Download the kernel sources you wish to modify.

##### On a Debian/Ubuntu system, enable the source repositories then download:

```bash
sudo apt-get update
sudo apt-get install build-essential libncurses-dev flex bison libssl-dev libelf-dev
sudo apt-get source linux-image-$(uname -r)
```

#### Step 2: Create the implementation file

1.  **Navigate** to the kernel source directory (`linux-...`).
2.  **Create a new directory** for your system call, for example `kernel/my_syscall/`.
3.  **Create a C file**, `kernel/my_syscall/my_syscall.c`, with the following implementation:

```c
#include <linux/kernel.h>
#include <linux/syscalls.h>

SYSCALL_DEFINE0(simple_syscall) {
    printk(KERN_INFO "simple_syscall: Simple system call received.\n");
    return 0;
}
```

#### Step 3: Add the files to the compilation

1.  **Create a `Makefile`** in your system call directory (`kernel/my_syscall/Makefile`):

```makefile
obj-y += my_syscall.o
```

2.  **Modify the parent `Makefile`** (`kernel/Makefile`) to include your new directory. Look for the `obj-y +=` line and add your directory:

```makefile
obj-y += my_syscall/
```

#### Step 4: Update the system call table

1.  **Find the system call table file** for your architecture, for example `arch/x86/entry/syscalls/syscall_64.tbl` for 64-bit systems.
2.  **Add a new line** at the end of the file, choosing the next available number. The format is `[number] [ABI] [macro_name] [function_name]`.

```
...
334 common simple_syscall sys_simple_syscall
```

*   **334**: the system call number.
*   **common**: indicates it is available for both 64-bit and 32-bit applications.
*   **simple_syscall**: the name of the `SYSCALL_DEFINE0` macro.
*   **sys_simple_syscall**: the name of the generated function.

#### Step 5: Compile and install the new kernel

1.  **Configure the kernel** by running `make menuconfig` at the root of the sources.
2.  **Compile the kernel**: `make -j$(nproc)`.
3.  **Install the modules and the new kernel**: `sudo make modules_install install`.
4.  **Update the bootloader** (GRUB) if necessary: `sudo update-grub`.
5.  **Reboot** your machine.

#### Step 6: Test the system call

1.  **Verify** that you are using the new kernel with `uname -r`.
2.  **Create a test program** in user space (`test_syscall.c`):

```c
#include <stdio.h>
#include <linux/unistd.h>
#include <sys/syscall.h>

#define __NR_simple_syscall 334

int main() {
    printf("Calling the simple_syscall system call...\n");
    long result = syscall(__NR_simple_syscall);

    if (result == 0) {
        printf("Syscall successful.\n");
    } else {
        perror("Error during the call");
    }
    return 0;
}
```

3.  **Compile the program**: `gcc test_syscall.c -o test_syscall`.
4.  **Execute the test**: `./test_syscall`.
5.  **Check the kernel output**: `dmesg | tail`. You should see the message `simple_syscall: Simple system call received.`.