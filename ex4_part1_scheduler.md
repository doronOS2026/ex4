# OS Assignment 4: Part 1 - Priority Scheduling

## 1. The Goal
The default xv6 scheduler is **Round Robin**, which treats all processes equally. In this task, you will implement a **Priority Scheduler** that allows some processes to get more CPU time than others.

We will use an **Accumulator-based** algorithm (similar to a "Fair Share" scheduler):
* Every process has a **Priority Value** (cost).
    * **High Priority** = Low Cost (e.g., 1).
    * **Low Priority** = High Cost (e.g., 10).
* Every time a process runs for a full time slice (quantum), we add its priority value to its total bill (`accumulator`).
* **The Rule:** The scheduler always picks the process with the **smallest bill** (lowest `accumulator`) to run next.

This ensures that "cheap" (high priority) processes get picked more often because their bill grows slowly, while "expensive" (low priority) processes get picked less often.

---

## 2. Specifications

### 2.1 New Process Fields
You must modify the Process Control Block (`struct proc`) in `kernel/proc.h` to include:
1.  `long long accumulator`: Tracks the total "cost" of CPU time used.
2.  `int ps_priority`: The priority of the process.
    * **Range:** 1 to 10.
    * **1** is the Highest Priority (grows slowly).
    * **10** is the Lowest Priority (grows fast).
    * **Default:** New processes start with priority **5**.

### 2.2 System Call
Implement a new system call to allow processes to change their own priority:
`int set_ps_priority(int priority)`
* **Input:** An integer between 1 and 10.
* **Behavior:** Updates the `ps_priority` of the calling process.
* **Return:** 0 on success, -1 on invalid input.

### 2.3 Logic & Rules

**Rule A: Charging the Process (The Tick)**
Whenever a process finishes its time quantum (i.e., a Timer Interrupt forces it to yield):
* Update its accumulator: `p->accumulator += p->ps_priority`
* **Note:** Do not charge the process if it yields voluntarily (e.g., calls `sleep` or waits for I/O). We only charge for consuming a full CPU slice.

**Rule B: Fairness for New/Waking Processes**
When a process is created (`allocproc`) or wakes up from sleeping (`wakeup`):
* To prevent it from having a huge advantage (accumulator 0) or a huge disadvantage (if it slept for a long time), we reset its accumulator.
* **Logic:** Set `p->accumulator` to the **minimum accumulator value** currently held by any `RUNNABLE` or `RUNNING` process in the system.
* *Edge Case:* If there are no other runnable processes, set it to 0.

**Rule C: The Scheduling Loop**
Modify the `scheduler()` function in `kernel/proc.c`:
* Instead of picking the *first* runnable process (Round Robin), you must scan the entire list.
* Pick the process with the **lowest `accumulator`**.
* **Tie-Breaking:** If two processes have the same `accumulator`, pick the one with the lowest index in the process array (the first one you find).
* **Optimization Note:** Since accumulators for high-priority processes grow slowly, the scheduler will naturally favor them.

---

## 3. Implementation Guide

### Step 1: Header Files (`kernel/proc.h`)
Add the fields to `struct proc`.
```
// kernel/proc.h
struct proc {
  // ... existing fields ...
  long long accumulator; // Accumulated run cost
  int ps_priority;       // Priority (1-10)
};
```

### Step 2: Initialization (kernel/proc.c)
Find the allocproc function. This is called when a new process is created (or during fork).

* Set the default ps_priority to 5.

* Initialize the accumulator to 0 (Note: The fairness logic in Step 4 will overwrite this, which is fine).

### Step 3: The System Call (kernel/sysproc.c)
Implement sys_set_ps_priority.

* Use argint to retrieve the argument.

* Validate that it is between 1 and 10.

* Update myproc()->ps_priority.

### Step 4: The Fairness Logic (kernel/proc.c)
You need to implement the "Find Minimum" logic in two places:

* 1. **allocproc:** When a process is created.

* 2. **wakeup:** When a process transitions from SLEEPING to RUNNABLE.

* **Implementation Hint:** You will need a loop that scans proc[NPROC] to find the smallest accumulator among all processes where state is RUNNABLE or RUNNING.
* **Why find the minimum?**
  * **The Problem:** Running processes have large accumulator values (e.g., 50,000). A new or waking process has a small value (e.g., 0).
  * **The Risk:** Since the scheduler picks the lowest value, a process starting at 0 would monopolize the CPU until it catches up to 50,000, starving everyone else.
  * **The Solution:** By setting the new process to the current system minimum, you place it at the "back of the line" rather than giving it a free pass to hog the CPU.


### Step 5: Charging the Accumulator (kernel/trap.c)
The time quantum expires when a Timer Interrupt occurs. This is handled in usertrap() and kerneltrap(). Look for the section checking which_dev == 2 (Timer).
```
if(which_dev == 2) {
    // This implies the process finished a full quantum
    // TODO: Add p->ps_priority to p->accumulator here!
    yield();
}
```

### Step 6: The Scheduler (kernel/proc.c)
Rewrite the loop in scheduler().

* Current (Round Robin):
```
for(p = proc; p < &proc[NPROC]; p++) {
   if(p->state == RUNNABLE) { ... run it ... }
}
```

* New Logic (Priority):
* 1. Iterate through all processes to find the RUNNABLE candidate with the lowest accumulator.
* 2. After the loop finishes, if you found a min_p:
  * **2.1** Switch state to RUNNING.
  * **2.2** Call swtch.

**Critical Warning (Locking): To read p->accumulator or p->state, you must hold p->lock.**

**The Challenge: You cannot hold locks for multiple processes blindly. You must design a strategy to scan the list, keep the lock of your "current best" candidate, and release the locks of rejected candidates immediately. failing to do so will lead to deadlocks or acquire panics.**


## 4. Verification
To test your scheduler, you should create a user program:

**Setup:** Fork 3 children.

**Priorities:** Set their priorities to 1, 5, and 10 using your new syscall.

**Work:** Have them run a long calculation loop (CPU bound).

**Measurement:** Print the start time and end time (using uptime()) for each child.

**Expected Result:** The child with Priority 1 (low cost) should finish significantly faster than the child with Priority 10 (high cost), as it will be chosen by the scheduler more frequently.

### **You should run more tests! this is just a basic one!** 

You may use the next code or write your own to test your scheduler
(take a look at the end of the scheduling presentation to see how to fully implement these codes) - 
* **user/spin.c**
```
// user/spin.c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int i;
  volatile int x = 0; 
  int count = 500000000; 

  printf("Spinning... (pid %d)\n", getpid());

  for(i = 0; i < count; i++){
    x += 1;
  }

  printf("Done! (pid %d)\n", getpid());
  exit(0);
}
```

* **user/stress.c**
```
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int
main(int argc, char *argv[])
{
  int i;
  int n = 3; 
  int pid;
  int priorities[3] = {5, 10, 1}; 

  printf("\n--- Stress test: Launching %d instances of spin ---\n", n);

  for(i = 0; i < n; i++){
    pid = fork();
    
    if(pid < 0){
      printf("fork failed\n");
      exit(1);
    }
    
    if(pid == 0){
      if(set_ps_priority(priorities[i]) != 0){
        printf("Error, priority is larger than 10 or less than 1");
        exit(-1);
      } 
      
      printf("Debug: Child (PID %d) launched with priority %d\n", getpid(), priorities[i]);

      char *args[] = { "spin", 0 };
      exec("spin", args);
      
      printf("exec failed\n");
      exit(1);
    }
  }
  printf("Parent: Waiting for children to finish...\n\n");
  
  int finished_pid;
  for(i = 0; i < n; i++){
    finished_pid = wait(0);
    
    if(finished_pid > 0){
        printf(">>> Child with PID %d has finished.\n", finished_pid);
    }
  }

  printf("\nStress test finished.\n");
  exit(0);
}
```

* An example for an output with int priorities[3] = {5, 10, 1};
```
--- Stress test: Launching 3 instances of spin ---
Parent: Waiting for children to finish...

Debug: Child (PID 4) launched with priority 5
Debug: Child (PID 5) launched with priority 10
Debug: Child (PID 6) launched with priority 1
Spinning... (pid 4)
Spinning... (pid 5)
Spinning... (pid 6)
Done! (pid 6)
>>> Child with PID 6 has finished.
Done! (pid 4)
>>> Child with PID 4 has finished.
Done! (pid 5)
>>> Child with PID 5 has finished.

Stress test finished.
```


## 5. Submission
**All modified files:**
* kernel/proc.h

* kernel/proc.c

* kernel/sysproc.c

* kernel/syscall.h

* kernel/syscall.c

* user/user.h

* user/usys.pl

* Makefile

**There is no need to submit your testing files such user/spin.c or user/stress**