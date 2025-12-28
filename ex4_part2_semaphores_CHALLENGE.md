# OS Assignment 4 - Part 2 - Synchronization: Counting Semaphores

> **Note:** This is the **CHALLENGE** version with requirement-focused specifications. 
> If you want more step-by-step implementation guidance, see the **GUIDED** version.

---

## 1. The Goal
xv6 provides low-level synchronization primitives: **spinlocks** and **sleep/wakeup**. However, it lacks high-level tools for coordinating multiple processes.

In this assignment, you will implement **Counting Semaphores**.
This mechanism is essential for solving the "Producer-Consumer" problem and managing limited resources.

---

## 2. Requirements & Specifications

### 2.1 The Semaphore Structure
You need to design a semaphore structure to manage the resource count and ensure thread safety.

**Constraints:**
* **Location:** Define your structure directly inside `kernel/sem.c`.
* **No New Headers:** Do **not** create `sem.h`. The internal layout of the semaphore should remain private to `sem.c`.
* **Public Interface:** Only the syscall wrappers (sys_sem_*) need to be declared in `kernel/defs.h`
* **Private Implementation:** The struct definition and helper functions stay in `sem.c`
* **Global Limit:** The kernel should support a fixed maximum number of semaphores (defined as `MAX_SEMS` in `kernel/param.h`).

### 2.2 System Call Interface
You must implement the following system calls. It is your responsibility to determine the necessary locking strategies to ensure these operations are **atomic** and **safe** from race conditions (including deadlock).

1.  **`int sem_create(int init_value)`**
    * Allocates an unused semaphore and initializes it with `init_value`.
    * Returns a generic `sem_id` (an integer descriptor) to the user.
    * Returns -1 if no slots are available or if `init_value` is invalid.

2.  **`int sem_free(int sem_id)`**
    * Frees the semaphore slot so it can be reallocated by future `sem_create` calls.
    * Returns 0 on success, -1 on failure (invalid sem_id).
    * **Safety:** You must handle the edge case where a process tries to free a semaphore that is actively in use or has processes sleeping on it. For this assignment, return -1 if the semaphore is in active use (recommended approach).

3.  **`int sem_down(int sem_id)`**
    * Decrements the semaphore value - the "Wait" operation.
    * **Blocking:** If the value is 0, the calling process must **block (sleep)** until the resource becomes available.
    * **Atomicity:** The check-and-decrement logic must be atomic to prevent race conditions.
    * **Lost Wakeup Prevention:** Your implementation must prevent the lost wakeup problem. Review the practical session material on this critical issue.

4.  **`int sem_up(int sem_id)`**
    * Increments the semaphore value - the "Signal" operation.
    * **Wakeup:** If processes are sleeping on this semaphore, this operation must wake them up.
    * **Correctness:** The while loop in `sem_down` handles spurious wakeups and the "Thundering Herd" problem - when multiple processes wake up, they must re-check the condition and only those with sufficient resources should proceed.

---

## 3. Implementation Guidelines

### Kernel Infrastructure
* **File Creation:** You will need to create `kernel/sem.c`.
* **Global Lock:** Since `sem_create` allocates from a global array, you must ensure two processes cannot allocate the same slot simultaneously.
* **Parameter Definition:** Add `#define MAX_SEMS 128` to `kernel/param.h`.

### `seminit`
Since you are introducing new locks into the kernel, they must be initialized during the boot process.
* Implement a function `void seminit(void)` in `kernel/sem.c` that initializes your global locks and semaphore structures.
* Call this function from `kernel/main.c`.

### `int sem_down(int sem_id)`
This function performs the **wait** operation. It attempts to decrement the semaphore's value by 1.
* **Behavior:** If the semaphore's current value is greater than 0, it decrements the value and returns immediately.
* **Blocking:** If the value is 0, the calling process must go to sleep until the value becomes positive.
* **Requirement:** You must implement this atomically to avoid race conditions. Specifically, you must ensure that a process does not "miss" a wakeup signal that arrives between the moment it checks the value and the moment it actually goes to sleep - **the "Lost Wakeup" problem**. Review your lecture materials on how `sleep(chan, lock)` solves this.
* **Spurious Wakeups:** Use a `while` loop (not `if`) to re-check the condition after waking up. This handles cases where multiple processes wake up but only one should proceed.

### `int sem_up(int sem_id)`
This function performs the **signal** operation.
* **Behavior:** It increments the semaphore's value by 1, indicating that a resource has become available.
* **Wake Up:** If any processes are currently sleeping and waiting for this semaphore, this function must wake them up using the same channel they slept on.

### `int sem_free(int sem_id)`
* **Behavior:** Releases the semaphore resource identified by `sem_id`, marking the slot in your global array as "unused" so it can be reallocated by a future call to sem_create.
* **Return:** Returns 0 on success, or -1 if the `sem_id` is invalid or if the semaphore is currently in use.

### Synchronization Logic
The core challenge of this assignment is correct synchronization.
* **Sleep/Wakeup:** Use the kernel's `sleep()` and `wakeup()` primitives correctly.
* **Sleep Channels:** Each semaphore needs a unique sleep channel. The address of the semaphore itself is a good choice.
* **Atomicity:** The combination of checking a condition and going to sleep must be atomic. Use `sleep(channel, lock)` which atomically releases the lock and sleeps.
* **Spurious Wakeups:** Remember that `sleep()` returns when a process is woken up, but it does not guarantee the condition you waited for is true. Your logic must handle this with a `while` loop.
* **Deadlocks:** Be mindful of the order in which you acquire/release locks, especially when sleeping. Never hold multiple locks when calling `sleep()` - `sleep()` only releases the one lock you pass to it.

---

## 4. System Call Integration

You are responsible for "wiring up" the new system calls into the xv6 kernel.

### System Call Wrappers
These are the functions that the kernel calls directly when a user triggers an interrupt. They are responsible for fetching arguments from user registers and verifying them. Implement these in `kernel/sem.c`.

You should implement:
1. `uint64 sys_sem_create(void)`
2. `uint64 sys_sem_free(void)`
3. `uint64 sys_sem_down(void)`
4. `uint64 sys_sem_up(void)`

Use `argint()` to fetch integer arguments from user space.

### System Call Registration
Don't forget the standard xv6 ritual for adding syscalls:
* `kernel/syscall.h`: Add syscall numbers (choose numbers that don't conflict - check existing ones and pick the next available).
* `kernel/syscall.c`: Add function pointers to the syscall table.
* `user/user.h`: Add user-space prototypes.
* `user/usys.pl`: Add stubs for the system call interface.

---

## 5. Crucial Compilation Steps

1. **Makefile:** You created `kernel/sem.c`. You must add `$K/sem.o` to the `OBJS` list in the `Makefile`. If you don't, your code won't be compiled!
2. **Kernel Definitions:** To make your new kernel functions (like sys_sem_create) visible, add their prototypes to `kernel/defs.h`:
```c
// sem.c
void            seminit(void);
```

---

## 6. Testing
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
- If you see 3 processes in critical section: Check your decrement logic and while loop in `sem_down`
- If processes never wake up: Check your wakeup channel matches sleep channel exactly
- If you get "acquire" panic: You're sleeping while holding the wrong lock
- Review the "Common Pitfalls" section below for more debugging guidance

---

## 7. Common Pitfalls to Avoid

1. **Using `if` instead of `while` in sem_down:** 
   - Always use `while` to re-check conditions after wakeup
   - Multiple processes wake up when `wakeup()` is called
   - Only those that still see `value > 0` should proceed

2. **Wrong sleep channel:** 
   - Each semaphore needs its own unique channel
   - The address `&sems[id]` used in both `sleep()` and `wakeup()` must match exactly

3. **Forgetting to initialize locks:** 
   - All locks must be initialized during boot via `seminit()`
   - This includes both your global allocation lock and each semaphore's lock

4. **Holding multiple locks during sleep:**
   - `sleep(channel, lock)` only releases ONE lock
   - Never hold your global allocation lock when calling `sleep()` - instant deadlock!
   - Only hold the specific semaphore's own lock when sleeping on it

5. **Not testing edge cases:** 
   - Test with `init_value = 0`
   - Test with many concurrent processes (more than MAX_SEMS)
   - Test rapid acquire/release cycles
   - Test freeing while processes are waiting

---

## 8. Common Mistakes

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
   - Occurs when you check a condition, release a lock, and THEN sleep as separate operations
   - Solution: Use `sleep(channel, lock)` which atomically releases the lock and sleeps
   - This is why the lock parameter exists in `sleep()`!

---

## 9. Debugging Tips

Add strategic printf statements to understand the execution flow:

```c
printf("[sem %d] down: value=%d (pid %d)\n", id, value, myproc()->pid);
printf("[sem %d] up: value=%d, waking up\n", id, value);
```

This helps you verify:
- Semaphore values are being updated correctly
- Processes are sleeping at the right times
- Wakeups are happening when expected
- The order of operations

---

## 10. Additional Testing Suggestions

Beyond the provided test, consider:

1. **Stress test:** Create 10+ children competing for 2 resources
2. **Zero initialization:** Create a semaphore with `init_value=0`, have one process wait, another signal
3. **Multiple semaphores:** Create several semaphores and have processes use different ones
4. **Rapid cycling:** Repeatedly acquire and release in a tight loop
5. **Error handling:** Try to use invalid semaphore IDs and verify proper error returns

---

## 11. Submission
Submit a zip file containing the following modified/created files:
1.  `kernel/sem.c`
2.  `kernel/syscall.h`
3.  `kernel/syscall.c`
4.  `kernel/defs.h`
5.  `kernel/main.c`
6.  `kernel/param.h`
7.  `user/user.h`
8.  `user/usys.pl`
9.  `Makefile`

---

**Final Tips:**
- Start by implementing the basic structure and `sem_create`
- Test incrementally - don't write everything at once
- Pay special attention to the lock ordering and sleep/wakeup channels
- Review the practical session slides on the lost wakeup problem
- When in doubt, use `while` loops, not `if` statements

Good luck!
