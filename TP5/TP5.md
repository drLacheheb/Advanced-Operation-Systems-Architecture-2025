# Linux Scheduling & Process Control

## Educational Objectives

*   Understand the real dynamics of the Linux scheduler.
*   Manipulate and compare the scheduling classes (SCHED_OTHER, SCHED_RR, SCHED_FIFO).
*   Observe the behavior of priorities and preemption in a multitasking environment.
*   Experiment with diagnostic tools: ps, htop, chrt, nice, schedtool, and perf.

## Lab Structure

### Part 1 – Experimental Study of Priority and Preemption

#### 1.1. Description

The task is to launch several CPU-intensive processes with different scheduling policies and observe how the processor is shared.

#### 1.2. Program to Test

```c
#include <stdio.h>
#include <unistd.h>
#include <sched.h>
#include <pthread.h>

void *compute(void *arg) {
    int id = *(int*)arg;
    while (1) {
        printf("Thread %d active...\n", id);
        for (volatile int i = 0; i < 100000000; i++); // CPU load
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;
    // For SCHED_RR or SCHED_FIFO, priorities range from 1 to 99, with 99 being the highest.
    struct sched_param p1 = {.sched_priority = 70}; 
    struct sched_param p2 = {.sched_priority = 10}; 

    int id1 = 1, id2 = 2;
    // Two threads are created, each executing the compute() function.
    pthread_create(&t1, NULL, compute, &id1);
    pthread_create(&t2, NULL, compute, &id2);

    // Apply different policies
    pthread_setschedparam(t1, SCHED_RR, &p1); // real-time round-robin
    pthread_setschedparam(t2, SCHED_OTHER, &p2); // standard policy
                                                 // For SCHED_OTHER, the priority is ignored (always 0).
    while (1) pause();
    return 0;
}
```

#### 1.3. Required Work

1.  **Compile the program:**
    ```bash
    gcc sched_test.c -o sched_test -pthread
    sudo ./sched_test
    ```

2.  **Open another terminal and observe the threads:**
    ```bash
    ps -eLo pid,tid,class,rtprio,pri,cmd
    ```

3.  **Compare the CPU load in htop:**
    *   T1 (SCHED_RR) should dominate the CPU.
    *   T2 (SCHED_OTHER) only runs when T1 is blocked.

#### 1.4. Thought Questions

*   What is the difference between SCHED_RR and SCHED_FIFO?
*   What happens if you lower the real-time priority of T1 to 1?
*   What happens if you reverse the policies (T1 = OTHER, T2 = RR)?
*   Why must this program be run as a superuser (sudo)?

### Part 2 – Dynamic Observation with Linux Tools

#### 2.1. Using `chrt`

**Display the policy and priority of a process:**
```bash
chrt -p <PID>
```

**Dynamically change the policy:**
```bash
sudo chrt -f -p 80 <PID>  # FIFO 80
sudo chrt -r -p 50 <PID>  # RR 50
sudo chrt -o -p 0 <PID>   # OTHER
```
Observe the effect in `htop` or `top`.

#### 2.2. Using `nice` and `renice`

**Test standard priorities:**
```bash
nice -n -10 ./cpu_task &
nice -n 10 ./cpu_task &
```
Show how the `nice` value influences the process weight in the CFS (Completely Fair Scheduler).

#### 2.3. Visualizing with `perf sched`

**Observe task switches:**
```bash
sudo perf sched record sleep 2
sudo perf sched latency
sudo perf sched map
```
Allows obtaining a timeline view of the actual scheduling.

### Part 3 – Advanced Experiment: Impact of Quantum and Load

#### 3.1. Objective

Show the effect of the time quantum in SCHED_RR and the CPU competition between tasks of the same class.

#### 3.2. Experiment

1.  Launch two threads with SCHED_RR and the **same priority**.
2.  Observe in `htop` that the CPU is shared equally.
3.  Increase the priority of one of the two and notice that the one with higher priority runs more often.

#### 3.3. Possible Extension

Modify the program to create **n threads** and measure:

*   The **total CPU time** consumed by each thread.
*   The **execution order** observed in `perf sched map`.

### Key Concepts Consolidated

| Concept                       | Skill Developed                          |
| ----------------------------- | ---------------------------------------- |
| Linux Scheduling Classes      | SCHED_OTHER, SCHED_FIFO, SCHED_RR        |
| Absolute vs. Relative Priorities | Understanding the kernel hierarchy       |
| Preemption and quantum        | Practical observation                    |
| Performance analysis          | Use of `perf` and `htop`                 |
| Dynamic management            | Manipulation with `chrt`, `nice`, `renice` |
