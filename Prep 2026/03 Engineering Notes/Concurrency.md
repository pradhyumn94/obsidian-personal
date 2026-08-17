### Tools
- **Atomics** 
	- Atomics provide thread-safe operations on single variables without locks. Under the hood, they use CPU instructions like compare-and-swap (CAS) that complete in one uninterruptible step.
	- Python lacks native atomics
- **Locks**
	- Locks provide mutual exclusion. When a thread holds a lock, other threads trying to acquire it block until the first thread releases.
	- creates a critical section where only one thread executes at a time.
- **Semaphores** 
	- Counting locks
```python
import threading
permits = threading.Semaphore(5)  # Allow 5 concurrent operations
permits.acquire()  # Block if no permits available
try:
    do_work()
finally:
    permits.release()  # Always release, even on exception
```

- **Condition variables** 
	- let threads wait efficiently for a condition to become true
	- A thread acquires a lock, checks a condition, and if not satisfied, waits
	- This atomically releases the lock and puts the thread to sleep. 
	- When another thread signals, waiters wake up and re-check.
```python
import threading
condition = threading.Condition()
with condition:
    while not ready:
        condition.wait()  # Release lock and sleep
    # Condition is now true
```
- **Blocking Queues**
	- combines a queue with condition variables to provide thread-safe producer-consumer handoff. 
	- Producers call put() to add items; if full, they block. Consumers call take() to remove items; if empty, they block.
```python
import queue
q = queue.Queue(maxsize=100)
q.put(task)   # Blocks if queue is full
t = q.get()   # Blocks if queue is empty
```



| Problem Type                                                                                   | What Breaks                          | Solutions                            | Common Problems                                          |
| ---------------------------------------------------------------------------------------------- | ------------------------------------ | ------------------------------------ | -------------------------------------------------------- |
| [Correctness](https://www.hellointerview.com/learn/low-level-design/concurrency/correctness)   | Shared state is updated concurrently | Locks, atomics, thread confinement   | Check-then-act, read-modify-write                        |
| [Coordination](https://www.hellointerview.com/learn/low-level-design/concurrency/coordination) | Threads need ordering or handoff     | Blocking queues, actors, event loops | Async request processing, bursty traffic                 |
| [Scarcity](https://www.hellointerview.com/learn/low-level-design/concurrency/scarcity)         | Resources are limited                | Semaphores, resource pools           | Concurrent op limits, resource consumption, object reuse |
|                                                                                                |                                      |                                      |                                                          |


### Correctness
- [Coarse-grained locking](https://www.hellointerview.com/learn/low-level-design/concurrency/correctness#coarse-grained-locking) protects all related state with one lock
	- Read- write locks : when reads vastly outnumber writes.
- [Fine-grained locking](https://www.hellointerview.com/learn/low-level-design/concurrency/correctness#fine-grained-locking) allows concurrent access to independent resources while protecting related ones
	- E.g : Alice and Bob can book different seats simultaneously. 
	- per-seat locks to allow concurrent bookings for different seats
- [Atomic variables](https://www.hellointerview.com/learn/low-level-design/concurrency/correctness#atomic-variables) work for single variables but fail for multi-field invariants
- [Thread confinement](https://www.hellointerview.com/learn/low-level-design/concurrency/correctness#thread-confinement) eliminates concurrency entirely for related data