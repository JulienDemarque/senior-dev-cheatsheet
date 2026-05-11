# Async, concurrency & I/O

How work is scheduled, where the **GIL** bites, and how to choose tools for **I/O-bound** vs **CPU-bound** workloads.

## Event loop

Single-threaded scheduler: registers I/O sources and timers; when `await` hits a waiting future, control returns to the loop so other tasks run. **asyncio** in Python is the standard library loop; **uvloop** (used by **Uvicorn**) is a faster drop-in.

### Example (asyncio basics)

```python
import asyncio

async def main():
    a, b = await asyncio.gather(
        asyncio.sleep(0.1, result="A"),
        asyncio.sleep(0.1, result="B"),
    )
    print(a, b)

asyncio.run(main())
```

## Async

`async def` coroutines and `await` non-blocking waits. Great for many concurrent sockets on one thread. Pure-Python CPU work still blocks the loop—offload with `run_in_executor` or **multiprocessing**.

### Example (offload CPU)

```python
import asyncio

def heavy(n):
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, heavy, 5_000_000)
```

## Concurrency

Logical progress overlap: coroutines, green threads, or time-sliced OS threads on one core. Throughput can improve without **parallelism**.

## Parallelism

True simultaneous execution on multiple cores. Needs multiple processes, multiple native threads (with released **GIL**), or vectorized/GPU code.

## Multiprocessing

Separate Python interpreters, separate memory. Bypasses **GIL** for CPU-heavy Python. Cost: pickling IPC, higher memory, harder shared state.

### Example

```python
from multiprocessing import Pool

def square(x):
    return x * x

if __name__ == "__main__":
    with Pool(4) as p:
        print(p.map(square, range(10)))
```

## Multithreading

`threading` shares memory; good for blocking **I/O** in libraries that release the **GIL** during waits (socket, some DB drivers). Not for parallel CPU-heavy pure Python.

## GIL (global interpreter lock)

Only one thread runs Python bytecode at a time in CPython. I/O-bound threads still help because they release the GIL while blocked. **numpy**/crypto extensions often release the GIL during heavy native sections.

## I/O

Waiting on network, disk, peripherals. Dominated by latency; overlap waits with **async**, thread pools, or pipelining.

## CPU-bound

Work scales with clock speed / core count: compression, parsing megabytes of JSON in Python, big matrix math. Scale with **multiprocessing**, Rust extensions, or better algorithms.

## I/O-bound

Work spends time blocked in **I/O**; more threads or **async** tasks improve utilization without needing more cores for Python bytecode.

### Example (rough mental model)

```text
If p = fraction of time CPU-busy per request:
- Many concurrent I/O waits → low p → async shines
- p ≈ 1 on one core → you need parallelism or faster/native code
```
