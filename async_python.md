# Master Async Python (asyncio)

> **Note for Beginners:** If you are reading about **Asynchronous Programming in Python** for the very first time, this guide is written specifically for you! Every concept is explained step-by-step with real-life analogies, clear diagrams, code snippets, execution outputs, and common pitfalls.

---

## 📌 Table of Contents
1. [What is Asynchronous Programming? (Real-Life Analogy)](#1-what-is-asynchronous-programming)
2. [Synchronous vs. Asynchronous Execution](#2-synchronous-vs-asynchronous-execution)
3. [When to Use Async Python (I/O-Bound vs. CPU-Bound)](#3-when-to-use-async-python)
4. [The Heart of Async: The Event Loop](#4-the-heart-of-async-the-event-loop)
5. [Coroutines and the `async def` Keyword](#5-coroutines-and-the-async-def-keyword)
6. [The `await` Keyword & Non-Blocking Sleep](#6-the-await-keyword--non-blocking-sleep)
7. [Entry Point: `asyncio.run()` vs. Jupyter Notebooks](#7-entry-point-asynciorun-vs-jupyter-notebooks)
8. [Scheduling Tasks: `asyncio.create_task()`](#8-scheduling-tasks-asynciocreate_task)
9. [Concurrent Execution: `asyncio.gather()` & The `*` Unpacking Operator](#9-concurrent-execution-asynciogather--the--operator)
10. [Modern Concurrency: `asyncio.TaskGroup()` (Python 3.11+)](#10-modern-concurrency-asynciotaskgroup-python-311)
11. [Timeouts & Task Cancellation (`asyncio.wait_for`)](#11-timeouts--task-cancellation-asynciowait_for)
12. [Rate Limiting with `asyncio.Semaphore`](#12-rate-limiting-with-asynciosemaphore)
13. [Producer-Consumer Pattern with `asyncio.Queue`](#13-producer-consumer-pattern-with-asyncioqueue)
14. [Running Blocking Sync Code: `asyncio.to_thread()`](#14-running-blocking-sync-code-asyncioto_thread)
15. [Async Iterators & Streaming (`async for`)](#15-async-iterators--streaming-async-for)
16. [Common Pitfalls & Best Practices](#16-common-pitfalls--best-practices)

---

## 1. What is Asynchronous Programming?

In standard (Synchronous) Python code, tasks execute **one after another in a straight line**. If Task 1 takes 5 seconds to download a webpage, your entire Python program stops and waits—doing absolutely zero work until the download completes.

**Asynchronous Programming (`asyncio`)** allows your program to pause a waiting task and switch to running another task in the meantime.

### 🍽️ Real-Life Analogy: The Single Chef vs. The Async Chef

- **Synchronous Chef (Blocking):**
  1. Put pizza in the oven (wait 15 minutes staring at the oven, doing nothing else).
  2. Once pizza is baked, slice tomatoes (2 minutes).
  3. Prepare lemonade (3 minutes).
  - ⏱️ **Total Time:** 20 minutes.

- **Asynchronous Chef (Non-Blocking / `asyncio`):**
  1. Put pizza in the oven (`await pizza_baking`).
  2. While waiting for the oven, immediately slice tomatoes (2 minutes).
  3. Prepare lemonade (3 minutes).
  4. When oven timer dings, retrieve the baked pizza.
  - ⏱️ **Total Time:** ~15 minutes!

---

## 2. Synchronous vs. Asynchronous Execution

Let's look at the time difference in Python code:

| Metric | Synchronous (Sequential) | Asynchronous (Concurrent) |
| :--- | :--- | :--- |
| **Execution Flow** | One task blocks the next task | Tasks pause & yield control during waiting |
| **3 Tasks (3s, 2s, 1s)** | Takes **3 + 2 + 1 = 6 seconds** | Takes **Max(3, 2, 1) = ~3 seconds** |
| **Resource Efficiency** | Low CPU utilization during I/O | High efficiency during I/O |

```
Synchronous Execution:
[ Task A (3s) ] ------------> [ Task B (2s) ] ------> [ Task C (1s) ] -> DONE (6s)

Asynchronous Execution:
[ Task A (3s) starts... ]
  ↳ [ Task B (2s) starts... ]
      ↳ [ Task C (1s) starts... ]
[ Task C done ] -> [ Task B done ] -> [ Task A done ] ------------------> DONE (3s)
```

---

## 3. When to Use Async Python?

> [!IMPORTANT]
> Async Python is **NOT** multi-threading or multi-core parallel computing. It runs on a **Single Thread**.

- **✅ YES - Use Async for I/O-Bound Tasks:**
  - Network requests (REST APIs, Web Scraping, WebSockets).
  - Database queries (PostgreSQL, MongoDB, Redis).
  - File reading/writing.
  - Interacting with LLM APIs (Groq, OpenAI, Anthropic, Pinecone).

- **❌ NO - Avoid Async for CPU-Bound Tasks:**
  - Heavy mathematical calculations, matrix multiplication.
  - Image processing, video encoding, Machine Learning model training.
  - *Reason:* CPU-bound tasks do not yield control (`await`), so they block the single thread. Use `multiprocessing` for CPU-bound work.

---

## 4. The Heart of Async: The Event Loop

The **Event Loop** is an infinite loop that manages and distributes execution of all asynchronous tasks.

```
       +---------------------------------------------+
       |                 EVENT LOOP                  |
       +---------------------------------------------+
         /                   |                     \
        v                    v                      v
  [Task 1: Waiting]   [Task 2: Running]    [Task 3: Waiting]
   (e.g. HTTP call)     (Executes code)     (e.g. DB Query)
```

1. The Event Loop picks a Task and executes it until it hits an `await`.
2. At `await`, Task 1 tells the Event Loop: *"I'm waiting for I/O. Run something else!"*
3. The Event Loop pauses Task 1 and immediately switches to Task 2.
4. When Task 1's data arrives, the Event Loop resumes Task 1 right where it paused.

---

## 5. Coroutines and the `async def` Keyword

To write an async function, prefix `def` with `async`. This creates a **Coroutine function**.

```python
import asyncio

# Define an async function (Coroutine Function)
async def greet(name: str):
    print(f"Hello, {name}!")

# ⚠️ Calling a coroutine function directly DOES NOT execute it!
result = greet("Sanjeet")
print(result)  
# Output: <coroutine object greet at 0x1024b8940>
```

> [!NOTE]
> Calling an `async def` function returns a **Coroutine Object**. It will not run until it is `await`ed or scheduled on an Event Loop.

---

## 6. The `await` Keyword & Non-Blocking Sleep

`await` pauses execution of the current coroutine until the awaited result is ready.

### Code Example:
```python
import asyncio
import time

async def fetch_api(source: str, delay: float):
    print(f"📡 Fetching from {source}...")
    # ❌ NEVER use time.sleep(delay) in async code! It blocks the entire Event Loop!
    # ✅ ALWAYS use asyncio.sleep(delay) for non-blocking delay:
    await asyncio.sleep(delay)
    print(f"✅ Data received from {source} in {delay}s!")
    return f"Data from {source}"
```

---

## 7. Entry Point: `asyncio.run()` vs. Jupyter Notebooks

How do you kick off an async program?

### 1. Standard Python Script (`.py` file)
In a standard script, use `asyncio.run(main())`:
```python
import asyncio

async def main():
    print("🚀 App Started!")
    await asyncio.sleep(1)
    print("🎉 App Finished!")

if __name__ == "__main__":
    # asyncio.run does 3 things:
    # 1. Creates a new Event Loop
    # 2. Executes main() to completion
    # 3. Closes the Event Loop cleanly
    asyncio.run(main())
```

### 2. Jupyter Notebook (`.ipynb` file)
In Jupyter Notebooks, **an Event Loop is ALREADY running in the background**.
- ❌ Calling `asyncio.run(main())` will crash with: `RuntimeError: asyncio.run() cannot be called from a running event loop`.
- ✅ Simply write: `await main()` directly inside notebook cells!

---

## 8. Scheduling Tasks: `asyncio.create_task()`

If you want to start a task immediately in the background **without blocking** the current line of code, use `asyncio.create_task()`.

```python
import asyncio

async def background_worker(task_name: str, duration: int):
    print(f"⚙️ Task {task_name} started (takes {duration}s)...")
    await asyncio.sleep(duration)
    print(f"✅ Task {task_name} finished!")
    return f"Result of {task_name}"

async def main():
    # Schedule task to run in the background IMMEDIATELY
    task1 = asyncio.create_task(background_worker("Database Backup", 3))
    task2 = asyncio.create_task(background_worker("Email Dispatch", 1))

    print("⚡ Main routine continues executing while background tasks run...")
    
    # Do other work here...
    await asyncio.sleep(0.5)
    print("⏳ Now waiting for task results...")

    # Wait for tasks when their results are needed
    res1 = await task1
    res2 = await task2
    print(f"Outputs: {res1} | {res2}")

asyncio.run(main())
```

---

## 9. Concurrent Execution: `asyncio.gather()` & The `*` Unpacking Operator

`asyncio.gather()` takes multiple coroutines/tasks and runs them concurrently, returning their outputs as a list in **exact order**.

### What does the asterisk `*` mean?
`asyncio.gather()` expects positional arguments (`gather(coro1, coro2, coro3)`), **NOT a single Python list**.
The `*` (star/asterisk) is Python's **unpacking operator**. It converts a list `[coro1, coro2]` into comma-separated positional arguments `coro1, coro2`.

```python
import asyncio

async def fetch_url(url: str):
    print(f"🌐 Requesting {url}...")
    await asyncio.sleep(1.5)  # Simulate network latency
    return f"200 OK from {url}"

async def main():
    urls = [
        "https://api.github.com",
        "https://api.openai.com",
        "https://api.groq.com",
    ]

    # Create list of coroutine objects using list comprehension
    tasks = [fetch_url(url) for url in urls]

    # Unpack the list into gather using *
    results = await asyncio.gather(*tasks)

    print("\nAll Results Received:")
    for res in results:
        print(" ->", res)

asyncio.run(main())
```

---

## 10. Modern Concurrency: `asyncio.TaskGroup()` (Python 3.11+)

Python 3.11 introduced **Structured Concurrency** via `asyncio.TaskGroup()`.

### Why TaskGroup is better than `gather()`:
If one task inside a `TaskGroup` raises an exception, `TaskGroup` **automatically cancels all remaining running tasks in the group**, preventing resource leaks and orphan background tasks.

```python
import asyncio

async def worker(name: str, delay: int):
    print(f"Worker {name} starting...")
    await asyncio.sleep(delay)
    print(f"Worker {name} finished!")
    return f"Done {name}"

async def main():
    try:
        async with asyncio.TaskGroup() as tg:
            # Create tasks inside the context manager
            t1 = tg.create_task(worker("A", 2))
            t2 = tg.create_task(worker("B", 1))
        
        # Once context manager exits, ALL tasks are guaranteed complete
        print(f"Results: {t1.result()}, {t2.result()}")

    except ExceptionGroup as eg:
        print(f"Caught task group errors: {eg}")

asyncio.run(main())
```

---

## 11. Timeouts & Task Cancellation (`asyncio.wait_for`)

Never let an API request hang indefinitely! Use `asyncio.wait_for()` to enforce a timeout limit.

```python
import asyncio

async def slow_database_query():
    try:
        print("🔍 Querying database...")
        await asyncio.sleep(10)  # Takes 10 seconds
        return "Query Results"
    except asyncio.CancelledError:
        print("⚠️ Database query was CANCELLED due to timeout!")
        raise  # Good practice to re-raise CancelledError

async def main():
    try:
        # Enforce a 2-second timeout
        result = await asyncio.wait_for(slow_database_query(), timeout=2.0)
        print("Result:", result)
    except asyncio.TimeoutError:
        print("❌ Request Timed Out! Aborted operation after 2 seconds.")

asyncio.run(main())
```

---

## 12. Rate Limiting with `asyncio.Semaphore`

When making thousands of API requests, servers will block your IP (Error 429: Too Many Requests). Use `asyncio.Semaphore` to cap max concurrent operations.

```python
import asyncio

# Allow maximum 2 concurrent tasks at a time
semaphore = asyncio.Semaphore(2)

async def limited_api_call(user_id: int):
    async with semaphore:  # Acquires lock slot
        print(f"🔒 [Active Slot] Processing request for User {user_id}")
        await asyncio.sleep(2)
        print(f"🔓 [Released Slot] Finished User {user_id}")

async def main():
    # Attempt 5 simultaneous calls, but Semaphore limits to 2 at a time
    tasks = [limited_api_call(i) for i in range(1, 6)]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

---

## 13. Producer-Consumer Pattern with `asyncio.Queue`

`asyncio.Queue` allows async tasks to safely exchange work items without race conditions.

```python
import asyncio

async def producer(queue: asyncio.Queue):
    for i in range(1, 4):
        print(f"📦 Producer adding Item-{i}")
        await queue.put(f"Item-{i}")
        await asyncio.sleep(0.5)

async def consumer(queue: asyncio.Queue, consumer_id: int):
    while True:
        item = await queue.get()
        print(f"  🛠️ Consumer-{consumer_id} processed {item}")
        await asyncio.sleep(1)
        queue.task_done()

async def main():
    queue = asyncio.Queue()

    # Start 2 worker consumers in background
    c1 = asyncio.create_task(consumer(queue, 1))
    c2 = asyncio.create_task(consumer(queue, 2))

    # Run producer
    await producer(queue)

    # Wait until all queue items are processed
    await queue.join()

    # Cancel background consumers cleanly
    c1.cancel()
    c2.cancel()

asyncio.run(main())
```

---

## 14. Running Blocking Sync Code: `asyncio.to_thread()`

What if you have legacy blocking code (like `time.sleep()`, `requests.get()`, or heavy file I/O) that cannot be rewritten as `async`? Use `asyncio.to_thread()` to offload it to a separate background thread!

```python
import asyncio
import time
import requests  # Blocking sync library

def blocking_sync_download(url: str):
    print(f"⏳ Synchronous download starting for {url}...")
    # requests.get is blocking!
    response = requests.get(url)
    return f"Downloaded {len(response.content)} bytes"

async def main():
    print("🚀 Async loop running...")
    
    # Offload blocking sync function to a separate OS thread
    result = await asyncio.to_thread(blocking_sync_download, "https://httpbin.org/get")
    
    print("✅ Result:", result)

asyncio.run(main())
```

---

## 15. Async Iterators & Streaming (`async for`)

Use `async for` when consuming streams of data (e.g., token streaming from LLMs or paginated API results).

```python
import asyncio

class AsyncTicker:
    """An Async Iterator that yields numbers with delay."""
    def __init__(self, count: int):
        self.count = count
        self.current = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.current < self.count:
            self.current += 1
            await asyncio.sleep(0.5)
            return f"Tick #{self.current}"
        else:
            raise StopAsyncIteration

async def main():
    print("📡 Streaming ticks:")
    async for tick in AsyncTicker(3):
        print("  ->", tick)

asyncio.run(main())
```

---

## 16. Common Pitfalls & Best Practices

| Mistake ❌ | Correct Approach ✅ | Why? |
| :--- | :--- | :--- |
| `time.sleep(1)` | `await asyncio.sleep(1)` | `time.sleep` freezes the entire event loop single-thread. |
| `requests.get(url)` | `aiohttp` / `httpx` or `asyncio.to_thread()` | `requests` is blocking; use an async HTTP library. |
| `coro_func()` without `await` | `await coro_func()` or `create_task()` | Calling an async function returns a coroutine object without executing it. |
| `asyncio.gather(tasks)` | `asyncio.gather(*tasks)` | `gather` requires unpacked positional arguments. |
| `asyncio.run()` in Jupyter | `await main()` | Jupyter already runs an active Event Loop. |

---

## 💡 Summary Cheat Sheet

```python
# 1. Define Coroutine
async def my_coro():
    await asyncio.sleep(1)
    return "Done"

# 2. Run from Terminal Script
asyncio.run(my_coro())

# 3. Schedule Background Task
task = asyncio.create_task(my_coro())

# 4. Concurrent List of Tasks
results = await asyncio.gather(*[my_coro() for _ in range(5)])

# 5. Modern TaskGroup (Python 3.11+)
async with asyncio.TaskGroup() as tg:
    t1 = tg.create_task(my_coro())
```

Happy Async Coding with Python! 🚀
