# Interrupt Handling and Real-Time Management

## Educational Objectives

At the end of this workshop, the student will be able to:

*   Analyze and observe the distribution and frequency of hardware interrupts (IRQs) on a Linux system.
*   Distinguish between deferred execution mechanisms: Tasklets and Workqueues (Bottom-Half techniques).
*   Control and optimize interrupt affinity (smp_affinity) to distribute load across CPU cores.
*   Understand the execution context differences between atomic operations (Tasklets) and process context (Workqueues).

## Prerequisites

*   **System**: Linux virtual machine with multicore CPU (2+ cores recommended)
*   **Kernel headers**: `linux-headers-$(uname -r)` installed
*   **Tools**: `make`, `gcc`, `cat`, `watch`, `grep`, `htop`
*   **Privileges**: `sudo` access for module loading and system parameter modification
*   **Knowledge**: Basic understanding of kernel modules (see TP2 and TP3)

## Workshop Plan

### Part 1 – Observing Hardware Interrupts

This section focuses on understanding how the Linux kernel distributes hardware interrupts (IRQs) across multiple CPU cores.

#### Step 1.1: Analyzing the IRQ State

1.  **Display the current interrupt statistics:**

```bash
cat /proc/interrupts
```

This file shows:
*   **Column 1**: IRQ number
*   **CPU columns**: Number of times each CPU core handled the interrupt
*   **Last columns**: Device name and description

2.  **Identify a significant hardware interrupt**
    Look for active interrupts such as:
    *   Network interface (e.g., `eth0`, `enp0s3`)
    *   Local APIC timer
    *   I/O APIC
    
    Note the **IRQ number** and observe how it's distributed across CPU cores.

3.  **Monitor interrupts dynamically:**

```bash
watch -n 1 'cat /proc/interrupts | head -n 20'
```

This updates the display every second, allowing you to observe interrupt patterns in real-time.

#### Step 1.2: Understanding Interrupt Affinity

**Interrupt affinity** determines which CPU cores are allowed to handle a specific interrupt.

1.  **Check the current affinity mask:**

```bash
cat /proc/irq/IRQ_NUMBER/smp_affinity
```

Replace `IRQ_NUMBER` with your chosen interrupt number (e.g., `29`).

The value is a hexadecimal bitmask:
*   `ffffffff` = all cores enabled
*   `00000001` = CPU0 only
*   `00000002` = CPU1 only
*   `00000003` = CPU0 and CPU1

2.  **Restrict interrupt handling to CPU0:**

```bash
echo 1 | sudo tee /proc/irq/IRQ_NUMBER/smp_affinity
```

3.  **Verify the change:**

```bash
watch -n 1 'cat /proc/interrupts | grep IRQ_NUMBER'
```

4.  **Generate activity** (for network-related IRQs):

```bash
ping -c 100 8.8.8.8
```

or

```bash
wget http://kernel.org
```

**Expected behavior:** Only the counter for CPU0 should increase significantly.

#### Step 1.3: Restoring Default Affinity

To restore interrupt distribution across all cores:

```bash
echo ffffffff | sudo tee /proc/irq/IRQ_NUMBER/smp_affinity
```

### Part 2 – Implementing Bottom-Halves

When a hardware interrupt occurs, the kernel splits the work into two parts:

*   **Top-Half**: Executes immediately in interrupt context (fast, atomic).
*   **Bottom-Half**: Deferred work executed later in a safer context.

This part implements both **Tasklets** (atomic context) and **Workqueues** (process context).

#### Step 2.1: Creating the Kernel Module

Create a file named `irq_bh_mod.c`:

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/workqueue.h>  // For Workqueues
#include <linux/interrupt.h>  // For Tasklets
#include <linux/init.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LinuxKernelLabStudent");
MODULE_DESCRIPTION("Bottom-Half demonstration: Tasklets vs Workqueues");

// ========== WORKQUEUE ==========
// Workqueues run in process context (can sleep)
static void workqueue_function(struct work_struct *work) {
    printk(KERN_INFO "BH_MOD: WORKQUEUE executed in process context (kworker).\n");
    printk(KERN_INFO "BH_MOD: Workqueue can sleep and use mutexes.\n");
}

static DECLARE_WORK(my_work, workqueue_function);

// ========== TASKLET ==========
// Tasklets run in atomic context (cannot sleep)
void tasklet_function(unsigned long data) {
    printk(KERN_INFO "BH_MOD: TASKLET executed in atomic/softirq context.\n");
    printk(KERN_INFO "BH_MOD: Tasklet is fast but cannot sleep.\n");
}

DECLARE_TASKLET(my_tasklet, tasklet_function, 0);

// ========== INITIALIZATION ==========
static int __init irq_bh_init(void) {
    printk(KERN_INFO "BH_MOD: Module loaded. Scheduling Bottom-Halves...\n");

    // Schedule the Workqueue
    schedule_work(&my_work);

    // Schedule the Tasklet
    tasklet_schedule(&my_tasklet);

    printk(KERN_INFO "BH_MOD: Both Bottom-Halves scheduled.\n");
    return 0;
}

// ========== CLEANUP ==========
static void __exit irq_bh_exit(void) {
    // Wait for workqueue completion
    flush_scheduled_work();

    // Kill the tasklet
    tasklet_kill(&my_tasklet);

    printk(KERN_INFO "BH_MOD: Module unloaded. Bottom-Halves cleaned up.\n");
}

module_init(irq_bh_init);
module_exit(irq_bh_exit);
```

#### Step 2.2: Creating the Makefile

Create a file named `Makefile`:

```makefile
obj-m += irq_bh_mod.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

**Note:** The indentation before `make` must be a TAB character, not spaces.

#### Step 2.3: Compiling and Loading the Module

1.  **Compile the module:**

```bash
make
```

This generates `irq_bh_mod.ko`.

2.  **Load the module:**

```bash
sudo insmod irq_bh_mod.ko
```

3.  **Observe kernel messages:**

```bash
dmesg | tail -n 10
```

**Expected output:**

```
[  123.456789] BH_MOD: Module loaded. Scheduling Bottom-Halves...
[  123.456790] BH_MOD: Both Bottom-Halves scheduled.
[  123.456791] BH_MOD: TASKLET executed in atomic/softirq context.
[  123.456792] BH_MOD: Tasklet is fast but cannot sleep.
[  123.456793] BH_MOD: WORKQUEUE executed in process context (kworker).
[  123.456794] BH_MOD: Workqueue can sleep and use mutexes.
```

**Note:** The order may vary, as both mechanisms are scheduled asynchronously.

#### Step 2.4: Understanding the Execution Contexts

| Mechanism   | Context       | Can Sleep? | Synchronization     | Use Case                           |
|-------------|---------------|------------|---------------------|------------------------------------|
| **Tasklet** | Atomic/Softirq| No         | Spinlocks only      | Fast, short deferred operations    |
| **Workqueue**| Process      | Yes        | Mutexes, semaphores | Longer operations, I/O allowed     |

**Key differences:**

*   **Tasklets**:
    *   Run in interrupt context (IRQs disabled or softirq)
    *   Execute on the same CPU that scheduled them
    *   Cannot block or sleep
    *   Use `spin_lock` for synchronization

*   **Workqueues**:
    *   Run in process context (kernel threads named `kworker`)
    *   Can be scheduled on any CPU
    *   Can sleep, wait for I/O, use blocking operations
    *   Use standard synchronization primitives (mutexes, semaphores)

#### Step 2.5: Observing Kernel Workers

While the module is loaded:

```bash
ps aux | grep kworker
```

You'll see kernel threads like:

```
root         5  0.0  0.0      0     0 ?        I<   10:00   0:00 [kworker/0:0H]
root       123  0.0  0.0      0     0 ?        I    10:01   0:00 [kworker/u256:1]
```

These are the threads that execute workqueue functions.

#### Step 2.6: Removing the Module

```bash
sudo rmmod irq_bh_mod
dmesg | tail -n 5
```

**Expected output:**

```
[  456.789012] BH_MOD: Module unloaded. Bottom-Halves cleaned up.
```

### Part 3 – Advanced Analysis

#### Step 3.1: Monitoring Softirqs

Softirqs are the mechanism used by tasklets. You can observe softirq statistics:

```bash
cat /proc/softirqs
```

This shows the number of softirqs handled by each CPU core, categorized by type:

*   `HI`: High-priority tasklets
*   `TIMER`: Timer interrupts
*   `NET_TX`: Network transmission
*   `NET_RX`: Network reception
*   `TASKLET`: Regular tasklets

#### Step 3.2: Measuring Interrupt Load

To see which CPU is handling the most interrupts:

```bash
watch -n 1 'grep "^CPU" /proc/stat'
```

The last column shows time spent handling interrupts (softirq + hardirq).

#### Step 3.3: Testing with CPU Stress

1.  **Install stress tools:**

```bash
sudo apt install stress-ng
```

2.  **Generate interrupt load:**

```bash
stress-ng --cpu 2 --timeout 60s
```

3.  **Observe interrupt distribution** while stress is running:

```bash
watch -n 1 'cat /proc/interrupts | head -n 20'
```

### Summary and Key Concepts

| Concept                  | Description                                                     |
|--------------------------|----------------------------------------------------------------|
| **IRQ**                  | Hardware interrupt request handled by the kernel               |
| **/proc/interrupts**     | File showing interrupt statistics per CPU                      |
| **smp_affinity**         | CPU mask controlling which cores handle an interrupt           |
| **Top-Half**             | Fast, atomic interrupt handler                                 |
| **Bottom-Half**          | Deferred work executed later (Tasklet or Workqueue)           |
| **Tasklet**              | Atomic context, cannot sleep, fast execution                   |
| **Workqueue**            | Process context, can sleep, uses kernel threads               |
| **Softirq**              | Low-level deferred execution mechanism used by tasklets        |

### Possible Extensions

*   Implement a real interrupt handler (requires hardware access).
*   Compare performance between Tasklets and Workqueues under load.
*   Use `perf` to profile interrupt handling overhead.
*   Experiment with `isolcpus` kernel parameter to isolate cores for real-time tasks.
*   Implement a custom workqueue (instead of using the system default).
*   Test with `IRQBALANCE` daemon and observe its effect on interrupt distribution.