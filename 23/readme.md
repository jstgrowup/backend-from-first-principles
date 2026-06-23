### **Technical Notes: Concurrency & Parallelism — IO Bound vs. CPU Bound**

---

#### **1. The Fundamental Requirement of Backend Systems**

- **Handling Multiple Users:** A production backend must handle thousands of users simultaneously.
- **Synchronous Failure:** If a server processes only one request at a time, other users face long wait times or "Server Busy" errors.
- **Efficiency & Hardware:** Modern CPUs can execute approximately **3 billion instructions per second** (or 3 million per millisecond).
- **The Cost of Idle Time:** If a server sits idle for 100ms waiting for a database response, it wastes the opportunity to execute **300 million instructions**.

#### **2. IO Bound vs. CPU Bound Tasks**

- **IO Bound (Input/Output):**
  - **Definition:** Tasks where the server spends most of its time waiting for external resources rather than performing computation.
  - **Examples:** Database queries, external API calls (email, cache), file system operations, and standard input/output (logging).
  - **Scale:** In a typical backend application, **70% to 95% of the time** is spent on IO.
  - **Latency Examples:** Local DB query (1–2ms), different availability zone (20–30ms), different region (90–100ms).
- **CPU Bound:**
  - **Definition:** Tasks requiring actual computation and number crunching by the CPU.
  - **Examples:** JSON parsing/serialization, data validation, image processing, matrix multiplication (ML/Graphics), and encryption (JWT verification).
  - **Performance:** Most typical backend CPU tasks finish in 1–2ms because modern CPUs are extremely fast.

---

#### **3. Concurrency vs. Parallelism**

- **Parallelism:**
  - **Concept:** Doing multiple things at exactly the same moment.
  - **Requirement:** Requires hardware support (multiple CPU cores). One core can only execute one instruction at any single moment.
  - **Best For:** CPU-bound workloads like video encoding or heavy encryption.
- **Concurrency:**
  - **Concept:** Dealing with multiple things at once, even if only one is being processed at a single moment.
  - **Mechanism:** Structuring a program to start, pause, resume, and switch between tasks.
  - **Requirement:** Can be achieved with only one CPU core.
  - **Best For:** IO-bound workloads to ensure the CPU isn't idle while waiting for network responses.

---

#### **4. Mechanism 1: Operating System Threads**

- **Definition:** An independent piece of execution managed by the OS.
- **Internal Structure:** Each thread gets its own **Stack** (for function calls and local variables) and an **Instruction Pointer** (to track where it is in the code).
- **Scheduling:** The **OS Scheduler** uses **Preemptive Scheduling**, giving each thread a "time slice" (usually in milliseconds) and pausing it whether it is finished or not.
- **Memory Sharing:** Threads within the **same process** can share memory (the **Heap** and global variables) via pointers, allowing fast communication without copying or serialization.
- **Overhead (The Cost of Threads):**
  - **Memory:** Linux threads have a default virtual stack size of ~**8MB**. Creating 10,000 native threads can exhaust gigabytes of memory.
  - **Creation:** Requires a system call to the kernel to allocate data structures and stack, causing latency.
  - **Context Switch:** Saving CPU registers and bookkeeping state when switching threads takes **1–10 microseconds**. At high concurrency (1,000+ threads), this maintenance work wastes significant CPU time.

#### **5. Mechanism 2: The Event Loop Model**

- **Definition:** A single-threaded model that manages multiple tasks using callbacks and non-blocking IO.
- **Mechanism:** When a task (e.g., Request A) hits an IO operation, it hands control back to the event loop and registers a **Callback** to run once the IO is done.
- **OS Support:** Relies on high-performance primitives like **epoll** (Linux) or **kqueue** (macOS) to monitor thousands of connections simultaneously.
- **Efficiency:** Eliminates context switching and heavy stack memory costs.
- **The Critical Rule:** **Never block the event loop.** A CPU-intensive task (taking 100ms+) will stop the entire loop and prevent other tasks from processing.
- **Syntactic Sugar:** `async/await` and `promises` are tools to help developers deal with callbacks while maintaining a clean mental model.

#### **6. Mechanism 3: Virtual Threads (Go Routines)**

- **Definition:** Lightweight virtual threads managed by a language runtime (like Go) instead of the OS.
- **Runtime Scheduler:** Go uses its own scheduler to map thousands or millions of **Go Routines** onto a small, fixed number of OS threads (usually matching the number of CPU cores).
- **Efficiency:** Switching between Go Routines is a simple pointer switch, making them 100x more lightweight than OS threads.
- **Usage:** Go creates a new Go Routine for **every single HTTP request** because they are so cheap.

---

#### **7. The Problem of Shared State (Race Conditions)**

- **Lost Update Problem:** Occurs when two threads read a variable, increment their local versions, and write back simultaneously, resulting in one increment being "lost".
- **Async/Await Race Conditions:** Even in single-threaded environments like JavaScript, race conditions occur if a shared variable (e.g., a bank balance) is checked before an `await` point and updated after, as another function call might have modified the variable during the `await`.
- **Solutions:**
  - **Locks/Mutexes:** Ensuring only one thread can enter a "Mutual Exclusion" area of code at a time.
  - **Channels (Go):** Communication by passing messages between routines rather than sharing variables.

---

### **Code Snippets**

✅ **Shown in video: The N+1 Concurrency Logic (Go Pseudo-code)**

```go
// From source: Demonstrating the logic of a database handler
func handler(w http.ResponseWriter, r *http.Request) {
    // Phase 1: CPU Operation (Parsing bytes/Loading memory)
    params := parseRequest(r)

    // Phase 2: IO Operation (The blocking call)
    // The thread/routine stops here and waits for the DB
    results := db.Query("SELECT * FROM users WHERE id = $1", params.ID)

    // Phase 3: Resume CPU Operation (Serialization/Response)
    json := serialize(results)
    w.Write(json)
}
```

_This illustrates the three phases of an API call: CPU bound parsing, IO bound waiting, and CPU bound response formatting._

✅ **Shown in video: Go Routine Creation (The `go` keyword)**

```go
// From source: Go's standard library creates a new routine for every request
func (srv *Server) Serve(l net.Listener) error {
    for {
        rw, err := l.Accept()
        if err != nil {
            return err
        }
        // This keyword spins up a new virtual thread (Go Routine)
        go srv.newConn(rw).serve(ctx)
    }
}
```

_This demonstrates how Go scales by creating lightweight virtual threads for every incoming connection._

✅ **Shown in video: The JavaScript Race Condition**

```javascript
// From source: Logic where single-threaded async code still causes a race
let balance = 100;

async function withdraw(amount) {
  if (balance >= amount) {
    // ⚠️ CONTROL IS GIVEN UP HERE
    await processWithdrawalInDB();

    // ⚠️ By the time we reach here, another call might have emptied the balance
    balance = balance - amount;
  }
}

// Result: If called twice simultaneously, balance could become negative.
```

_This snippet proves that `async/await` does not automatically protect against logical race conditions._

🧪 **Generated learning example: Thread Locking (Python)**

```python
import threading

# A shared global variable
counter = 0
lock = threading.Lock()

def increment():
    global counter
    # Mutual Exclusion: Only one thread can enter this block
    with lock:
        # Step 1: Read current value
        # Step 2: Add 1
        # Step 3: Write back
        counter += 1

# This prevents the 'Lost Update' problem described in the notes.
```

_This simulates the locking mechanism described to solve the shared state problem in multi-threaded environments._

---

**Recommended Reading:** _Operating Systems: Three Easy Pieces_.
**Key Tools:** `epoll` (Linux), `kqueue` (macOS).
