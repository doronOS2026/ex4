# OS Assignment 4 - Part 2 - Synchronization: Counting Semaphores

> **Note:** This is the **GUIDED** version with step-by-step instructions. 
> If you want more of a challenge with less implementation help, see the **CHALLENGE** version.

---

## 1. The Goal
xv6 provides low-level synchronization primitives: **spinlocks** (for mutual exclusion) and **sleep/wakeup** (for waiting). However, it lacks high-level tools for coordinating multiple processes.

In this assignment, you will implement **Counting Semaphores**.
Unlike a spinlock (which is binary: locked/unlocked), a counting semaphore tracks the number of available "resources".
* **Init(N):** We start with N resources.
* **Down (P):** Attempt to grab a resource. If `count > 0`, decrement and proceed. If `count == 0`, **sleep** until one becomes available.
* **Up (V):** Release a resource. Increment `count`. If anyone is sleeping waiting for this resource, **wake them up**.

This mechanism is essential for solving the "Producer-Consumer" problem and managing limited resources (like a connection pool).

---

## 2. Specifications

### 2.1 The Semaphore Structure
You need to define a new kernel structure in `kernel/sem.c`.

**Do NOT create a separate sem.h file.** The struct should remain private to sem.c.

A semaphore needs:
1.  `int value`: The number of resources available.
2.  `struct spinlock lock`: To protect the value from concurrent access.
3.  `int used`: To track if this semaphore slot is allocated or free.

### 2.2 System Calls
You must implement the following system calls to expose semaphores to user space:

1.  **`int sem_create(int init_value)`**
    * Finds an unused semaphore slot in the kernel.
    * Initializes its value to `init_value` and initializes its spinlock.
    * Returns a `sem_id` (descriptor/index) to the user (or -1 on failure).

2.  **`int sem_free(int sem_id)`**
    * Releases the semaphore slot so it can be reused.
    * Returns 0 on success, -1 on failure.
    * **Safety Note:** For this assignment, `sem_free` should return -1 if any processes are currently using or sleeping on this semaphore. This prevents crashes from freeing an active semaphore.

3.  **`int sem_down(int sem_id)`**
    * **The "Wait" Operation.**
    * Decrements the semaphore's value.
    * **Crucial:** If the value is 0, the process must **sleep** until the value is positive.
    * **Atomicity:** The check-and-decrement logic must be atomic.

4.  **`int sem_up(int sem_id)`**
    * **The "Signal" Operation.**
    * Increments the semaphore's value.
    * Wakes up any sleeping processes waiting on this semaphore.

---

## 3. Implementation Guide

### Step 1: Kernel Infrastructure
Define a global array of semaphores in the kernel to manage them.
* **File:** `kernel/sem.c` (you will need to create this).
* **Limit:** Define `MAX_SEMS` (e.g., 128) in `kernel/param.h`.

**Hint: Global Allocation Lock**
Since `sem_create` searches the global array for a free slot, you must prevent race conditions where two processes grab the same slot.
``` c
struct sem {
  struct spinlock lock;  // Protects this semaphore's value
  int value;             // Number of resources available
  int used;              // Is this slot currently allocated?
};
struct sem sems[MAX_SEMS];
struct spinlock sem_array_lock; // Protects allocation/deallocation
```

### Step 2: Initialization
In `kernel/main.c`, you will need to call a function - `seminit()` to initialize your locks.
Implement `void seminit(void)` in `kernel/sem.c`:
* Initialize `sem_array_lock`.
* Loop through `sems` array and initialize the lock for each semaphore.
* Set all `used` flags to 0.

### Step 3: Implementing `sem_create(int init_value)`
1. Acquire `sem_array_lock`
2. Iterate through `sems` to find a slot where `used == 0`
3. Mark it `used = 1`, set `value = init_value`
4. Release the lock and return the index.
5. If no free slots are available, release the lock and return -1.

### Step 4: Implementing `sem_free(int id)`
1. Validate the `id` (check bounds and that `sems[id].used == 1`)
2. Acquire `sem_array_lock`
3. Acquire the semaphore's own lock `sems[id].lock`
4. Mark `used = 0`
5. Release both locks

### Step 5: Implementing `sem_down(int id)` - The Critical Part
This is where you must combine locks and sleep. 

**Logic Flow:**
1. Validate the `id`
2. Acquire the semaphore's lock: `acquire(&sems[id].lock)`
3. **Use a `while` loop**: `while (sems[id].value == 0)`
    * Call `sleep(&sems[id], &sems[id].lock)`
    * **Sleep Channel:** Use the address of the semaphore struct itself as the unique sleep channel. This must match what you use in `wakeup()`.
    * **Important:** `sleep()` will atomically release `sems[id].lock` and put the process to sleep. When woken up, it re-acquires the lock before returning.
4. Once the loop finishes `(value > 0)`, decrement `value`.
5. Release the lock: `release(&sems[id].lock)`

**Why a while loop?** This handles "spurious wakeups" and the "Thundering Herd" problem - when multiple processes wake up from `wakeup()`, they must re-check the condition. Only processes that see `value > 0` can proceed.

### Step 6: Implementing `sem_up(int id)`
1. Validate the `id`
2. Acquire the semaphore's lock: `acquire(&sems[id].lock)`
3. Increment `value`: `sems[id].value++`
4. Call `wakeup(&sems[id])`
    * Make sure you wake up the **same channel** you slept on: the address of `sems[id]`
    * This wakes up ALL processes sleeping on this semaphore. They will each re-check the `while` condition.
5. Release the lock: `release(&sems[id].lock)`

### Step 7: System Call Wrappers
These are the functions that the kernel calls directly when a user triggers an interrupt. They are responsible for fetching arguments from user registers and verifying them. Implement these in `kernel/sem.c`.

You should implement:
1. `uint64 sys_sem_create(void)`
2. `uint64 sys_sem_free(void)`
3. `uint64 sys_sem_down(void)`
4. `uint64 sys_sem_up(void)`

**Remember to use `argint` to fetch data from the registers!**

Example for `sys_sem_create`:
```c
uint64 sys_sem_create(void) {
  int init_value;
  argint(0, &init_value);
  return sem_create(init_value);
}
```

### Step 8: System Call Registration
Don't forget the standard xv6 ritual for adding syscalls:
* `kernel/syscall.h`: Add syscall numbers (choose numbers that don't conflict with existing ones - check the file for the next available).
* `kernel/syscall.c`: Add function pointers to the syscall table.
* `user/user.h`: Add user-space prototypes.
* `user/usys.pl`: Add stubs for the system call interface.

### Crucial Compilation Steps:
1. **Makefile:** You created `kernel/sem.c`. You must add `$K/sem.o` to the `OBJS` list in the `Makefile`. If you don't, your code won't be compiled!
2. **Kernel Definitions:** To make your new kernel functions visible, add their prototypes to `kernel/defs.h`:
```c
// sem.c
void            seminit(void);
int             sem_create(int);
int             sem_free(int);
int             sem_down(int);
int             sem_up(int);
```

---

## 4. Testing
You may use the following code to test your semaphores.
* **Keep in mind, this is a basic test, you should run more of your own!**
* **To compile this, simply add `$U/_sem_test` to the `UPROGS` list in your `Makefile`**

``` c
#include "kernel/types.h"
#include "user/user.h"

void
worker(int sem_id, int child_id)
{
  int pid = getpid();
  
  // 1. Attempt to acquire the semaphore
  // If count is 0, this will block until a resource is freed.
  if(sem_down(sem_id) < 0){
    printf("Child %d: sem_down failed\n", pid);
    exit(1);
  }

  // 2. Critical Section (Simulate heavy work)
  printf("[Child %d] Acquired resource. (Doing work...)\n", pid);
  
  // Sleep for 20 ticks to simulate a task and hold the resource
  // allowing us to see if other children are blocked.
  sleep(20);

  // 3. Release the semaphore
  printf("[Child %d] Released resource.\n", pid);
  if(sem_up(sem_id) < 0){
    printf("Child %d: sem_up failed\n", pid);
    exit(1);
  }

  exit(0);
}

int
main(int argc, char *argv[])
{
  int sem_id;
  int i;
  int pid;
  int num_children = 3;
  int init_value = 2; // Only 2 resources available

  printf("[Parent] Creating semaphore with value %d.\n", init_value);

  // Initialize semaphore with 2 resources
  sem_id = sem_create(init_value);
  if(sem_id < 0){
    printf("[Parent] Error: Could not create semaphore.\n");
    exit(1);
  }

  // Fork 3 children
  for(i = 0; i < num_children; i++){
    pid = fork();
    if(pid < 0){
      printf("[Parent] Error: Fork failed.\n");
      exit(1);
    }

    if(pid == 0){
      // Child process
      worker(sem_id, i);
    }
  }

  // Parent waits for all children to finish
  for(i = 0; i < num_children; i++){
    wait(0);
  }

  // Cleanup
  if(sem_free(sem_id) < 0){
    printf("[Parent] Error: Could not free semaphore.\n");
  } else {
    printf("[Parent] Semaphore freed. Test finished.\n");
  }

  exit(0);
}
```

**Example Output:**
```
$ sem_test
[Parent] Creating semaphore with value 2.
[Ch[ilCdh i4l]d  A5c]q uAicrqeudi rreeds oruerscoeu.r ce(.D o(iDnogi wngo rwko.r.k.).
..)
[[CChhiildl d 54]]  RReelleeaasseedd  rreessoouurrccee..

[Child 6] Acquired resource. (Doing work...)
[Child 6] Released resource.
[Parent] Semaphore freed. Test finished.
```

**Explanation:**
* The line `[Ch[ilCdh i4l]d A5c]q...` looks broken, but it is exactly what you want to see for the first two children.
* **What happened:** Child 4 and Child 5 both successfully called sem_down. Since the semaphore value was 2, they both got through at the exact same time.
* **Why it looks messy:** Because they are running in parallel on different CPUs (or time-slicing very fast), they both tried to write to the console simultaneously. Their letters got mixed up. This proves Process 4 and Process 5 were in the Critical Section together.
* The line `[Child 6] Acquired resource. (Doing work...)` appeared cleanly at the bottom after the others released their resources.
* This proves that `sem_down` correctly put Child 6 to sleep when the counter hit 0, and `sem_up` correctly woke it up later.

**What if the output is different?**
- If you see 3 processes in critical section: Check your decrement logic in `sem_down`
- If processes never wake up: Check your wakeup channel matches sleep channel exactly
- If Child 6 never prints "Acquired": It's stuck sleeping - check your `wakeup()` call
- If you get random crashes: Check you're not freeing semaphores that are in use

---

## 5. Common Pitfalls to Avoid

1. **Using `if` instead of `while` in sem_down:** 
   - Always use `while` to re-check conditions after wakeup
   - Multiple processes can wake up, but only those with `value > 0` should proceed

2. **Wrong sleep channel:** 
   - Each semaphore needs its own unique channel
   - Use `&sems[id]` consistently in both `sleep()` and `wakeup()`

3. **Forgetting to initialize locks:** 
   - Call `seminit()` from `main.c` during boot
   - Initialize both `sem_array_lock` and each semaphore's individual lock

4. **Not testing edge cases:** 
   - Test with `init_value=0` 
   - Test with many concurrent processes
   - Test rapid acquire/release cycles

5. **Holding multiple locks during sleep:**
   - `sleep()` only releases ONE lock (the one you pass to it)
   - Never hold `sem_array_lock` when calling `sleep()` - this will cause a deadlock!

---

## 6. Common Mistakes

1. **Panic: acquire** 
   - If you see a kernel panic saying `acquire`, it often means you tried to sleep while holding a lock other than the one you passed to sleep. 
   - **Remember:** `sleep()` releases only the lock you give it. If you hold other locks (like the global array lock), you will deadlock or panic.

2. **"remap" or Page Faults** 
   - If you try to pass the semaphore struct itself (a kernel pointer) to user space, the kernel will crash.
   - **User Space:** Only knows the `int id`.
   - **Kernel Space:** Maps that id to `&sems[id]`.

3. **Race Conditions in Test** 
   - If your `sem_test` output is scrambled (e.g., 3 children enter a critical section meant for 2), check your `sem_down` logic. 
   - Did you release the lock after decrementing the value? 
   - Did you re-check `value > 0` after waking up from sleep (using `while`, not `if`)?

4. **Lost Wakeup Problem:**
   - This happens if you check the condition, release the lock, THEN sleep (two separate steps)
   - Solution: Use `sleep(channel, lock)` which atomically releases the lock and sleeps
   - Review the practical session slides on this!

---

## 7. Debugging Tips

Add printf statements to trace execution:
```c
// In sem_down, before sleep:
printf("[sem %d] down: value=%d (pid %d sleeping)\n", 
       id, sems[id].value, myproc()->pid);

// In sem_up, before wakeup:
printf("[sem %d] up: value=%d (waking up processes)\n", 
       id, sems[id].value);
```

This helps you see:
- When processes go to sleep
- When they wake up
- The semaphore value at each step
- If the value is being decremented/incremented correctly

---

## 8. Submission
Please submit a zip file containing the following modified/created files:
1. `kernel/sem.c` (Your new semaphore implementation)
2. `kernel/syscall.h`
3. `kernel/syscall.c`
4. `kernel/defs.h`
5. `kernel/main.c` (Where you added seminit)
6. `kernel/param.h` (Where you defined MAX_SEMS)
7. `user/user.h`
8. `user/usys.pl`
9. `Makefile`

**Good luck!** Remember to start early and test incrementally. Build one system call at a time, test it, then move to the next.
