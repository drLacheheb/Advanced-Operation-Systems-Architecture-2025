# Scheduling under Linux

### 1. General Objective

The objective of this project is to understand how the Linux kernel manages processor sharing between multiple processes and threads, through the manipulation and observation of the different scheduling policies: `SCHED_OTHER`, `SCHED_RR`, and `SCHED_FIFO`.

Students will need to develop an application that:

1.  Creates several concurrent processes or threads.
2.  Assigns a specific scheduling policy and priority to each.
3.  Measures the CPU time consumed by each entity.
4.  Compares the results and draws conclusions about the kernel's behavior.

### 2. Project Description

#### 2.1 Context

The Linux kernel has several scheduling classes, including:

*   **`SCHED_OTHER`**: default policy, based on the *Completely Fair Scheduler (CFS)*.
*   **`SCHED_RR`**: real-time policy with a fixed time quantum (*Round Robin*).
*   **`SCHED_FIFO`**: real-time policy without a time quantum (runs until it blocks).

This project invites you to **experimentally observe and analyze** these mechanisms through a small multi-threaded program.

#### 2.2 Tasks to be performed

**Step 1 — Creating a CPU load**

*   Write a program in **C language** that creates **N threads** (or processes).
*   Each thread executes an intensive calculation loop (simulating a CPU-bound job).
*   Each thread periodically displays a message indicating its activity, for example:

```
[Thread 1] Execution in progress...
[Thread 2] Execution in progress...
```

**Step 2 — Application of scheduling policies**

*   For each thread, apply one of the following policies:
    *   `SCHED_OTHER`
    *   `SCHED_RR`
    *   `SCHED_FIFO`
*   Define a specific **priority** via the `sched_param` structure.
*   Use the call:

```c
pthread_setschedparam(thread_id, policy, &param);
```

*   Test different combinations (for example: one FIFO thread, one RR, one OTHER).

**Step 3 — CPU time measurement**

*   For each thread, measure:
    *   the **total execution time**;
    *   the **CPU time consumed**.
*   You can use:

```c
clock_gettime(CLOCK_THREAD_CPUTIME_ID, &timespec);
```

or the function:

```c
getrusage(RUSAGE_THREAD, &usage);
```

**Step 4 — Recording and displaying results**

*   Record the measurements in a CSV file:

```csv
Thread,Policy,Priority,CPU_Time(ms)
1,SCHED_RR,70,1250
2,SCHED_OTHER,0,80
3,SCHED_FIFO,90,2400
```

*   Display a summary table at the end of the program.

**Step 5 — Analysis and interpretation**

In a report, you must:

*   Describe the tests performed (number of threads, policies, priorities).
*   Present the experimental results (tables or graphs).
*   Interpret the observed differences:
    *   Which threads monopolized the CPU?
    *   What is the difference between `SCHED_RR` and `SCHED_FIFO`?
    *   Is the kernel's behavior fair?

### 3. Technical Indications

*   **Compilation:**
```bash
 gcc scheduling.c -o scheduling -pthread
```

*   **Execution (requires sudo for real-time policies):**
```bash
sudo ./scheduling
```

*   **Observation:**
    *   `top`, `htop`, `ps -eLo pid,tid,class,rtprio,pri,cmd`
    *   `cat /proc/sys/kernel/sched_rr_timeslice_ms`

### 4. Expected Deliverables

1.  **Commented source code** (.c + Makefile)
2.  **CSV file** containing the experimental measurements
3.  **Summary report (4 to 6 pages)** including:
    *   Introduction and objective
    *   Description of the method
    *   Experimental results (tables/graphs)
    *   Analysis and interpretation
    *   Conclusion

### 6. Example of Expected Results (indicative)

| Thread | Policy      | Priority | CPU Time (ms) | % CPU |
|--------|-------------|----------|---------------|-------|
| 1      | SCHED_RR    | 70       | 2300          | 97%   |
| 2      | SCHED_OTHER | 0        | 100           | 3%    |
| 3      | SCHED_FIFO  | 80       | 2400          | 99%   |

**Interpretation:**
Real-time threads largely dominate the CPU.
`SCHED_FIFO` runs without yielding, while `SCHED_RR` shares according to its time quantum.
`SCHED_OTHER` is almost starved.