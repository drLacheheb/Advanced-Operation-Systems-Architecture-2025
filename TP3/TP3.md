# Create Your First Kernel Module

## Educational Objectives

Learn to extend the functionality of the Linux kernel without recompiling the entire system, thanks to dynamically Loadable Kernel Modules (LKM).

This workshop introduces **kernel programming**, a central pillar for:

*   Writing device drivers,
*   Understanding kernel/hardware interactions,
*   Experimenting with memory, interrupts, or networking.

## Learning Goals

At the end of this workshop, the student will be able to:

*   Create, compile, and load a simple kernel module.
*   Analyze the module's behavior using `dmesg`.
*   Manage the complete life cycle of a module (insertion, execution, removal).
*   Understand the basic structure of a kernel module (init, exit, license).

## Workshop Plan

### 1) Environment Preparation

Use the same Ubuntu/Debian virtual machine as in the previous workshop.

Install the necessary tools:

```bash
sudo apt install build-essential linux-headers-$(uname -r)
```

### 2) Creating a Simple Kernel Module

**File:** `hello_module.c`

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello, Kernel! Module loaded successfully.\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye, Kernel! Module unloaded.\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LinuxKernelLabStudent");
MODULE_DESCRIPTION("Simple kernel module for demonstration");
```

### 3) Creating the Makefile

**File:** `Makefile`

```makefile
obj-m += hello_module.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### 4) Compiling the Module

In the module's directory:

```bash
make
```

A file named `hello_module.ko` is generated.

### 5) Loading and Executing the Module

a. **Load the module:**

```bash
sudo insmod hello_module.ko
```

b. **Check the kernel messages:**

```bash
dmesg | tail
```

c. **Remove the module:**

```bash
sudo rmmod hello_module
dmesg | tail
```

### 6) Exploration and Observation

**List the loaded modules:**

```bash
lsmod | head
```

**Module details:**

```bash
modinfo hello_module.ko
```

### 7) Advanced Variant: Parameterized Module

**Add a parameter:**

```c
static char *name = "MyModule";
module_param(name, charp, 0000);
MODULE_PARM_DESC(name, "Name of the module");

static int __init hello_init(void) {
    printk(KERN_INFO "Module %s initialized.\n", name);
    return 0;
}
```

**Compile, then load with:**

```bash
sudo insmod hello_module.ko name="KernelStudent"
```