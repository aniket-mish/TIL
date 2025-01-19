# python concurrency

references

1. https://realpython.com/python-concurrency/#:~:text=Concurrency%20refers%20to%20the%20ability,unique%20benefits%20and%20trade%2Doffs
2. https://fastapi.tiangolo.com/async/

## concurrency ⇒ handle multiple tasks at once

- threading
- async tasks

always run on a single processor, which means they can only run one at a time

## parallelism

- multiprocessing

true parallelism by using multiple CPU cores

one way to think about it is each process runs its own Python interpreter

<img width="796" alt="Screenshot 2025-01-19 at 8 44 29 AM" src="https://github.com/user-attachments/assets/9189029c-311e-4ba7-92e3-2d7785ed92a5" />

*bypass GIL by using experimental free threading ⇒ python 3.13*

concurrency is useful in tasks like:

- I/O bound → the program slows down as it waits for I/O from some external resource. slower than CPU. e.g. file system or network connections.
- CPU bound → computation

![iovscpu-bound](https://github.com/user-attachments/assets/ffc7ca00-1673-4cf1-bc8f-2e572aa7ae6b)

## speeding up an I/O-bound program

e.g. downloading content over the network

### **synchronous version** → normal paradigm

following code downloads content from URLs

```python
import time

import requests

def main():
    sites = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
    ] * 80
    start_time = time.perf_counter()
    download_all_sites(sites)
    duration = time.perf_counter() - start_time
    print(f"Downloaded {len(sites)} sites in {duration} seconds")

def download_all_sites(sites):
    with requests.Session() as session: # retains state across requests and reuse the connection to speed things up
        for url in sites:
            download_site(url, session)

def download_site(url, session):
    with session.get(url) as response:
        print(f"Read {len(response.content)} bytes from {url}")

if __name__ == "__main__":
    main()
```

***it’s slow*** → takes 10-20s depending on your internet connection, network congestion, etc.

### moving on to a **multi-threaded version** → use `threading` and `concurrent.futures` modules

```python
import threading
import time
from concurrent.futures import ThreadPoolExecutor

import requests

# specific to each thread. need to be created only once.
thread_local = threading.local()

def main():
    sites = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
    ] * 80
    start_time = time.perf_counter()
    download_all_sites(sites)
    duration = time.perf_counter() - start_time
    print(f"Downloaded {len(sites)} sites in {duration} seconds")

def download_all_sites(sites):
		# manages threads: you can create 100s/1000s threads for I/O-bound tasks
    with ThreadPoolExecutor(max_workers=5) as executor:
        executor.map(download_site, sites)

def download_site(url):
    session = get_session_for_thread()
    with session.get(url) as response:
        print(f"Read {len(response.content)} bytes from {url}")

# each thread creates a single session for the first time and then uses that session on each subsequent call
def get_session_for_thread():
    if not hasattr(thread_local, "session"):
        thread_local.session = requests.Session()
    return thread_local.session

if __name__ == "__main__":
    main()
```

`ThreadPoolExecutor`

thread - sequence of instructions

pool - a pool of threads, each of which runs concurrently

executor - controls how and when each of the threads in the pool will run

how to obtain a session object?

as OS controls when your task gets interrupted and another one starts, any data shared between the threads needs to be *thread-safe* to avoid data corruption. because threads may interfere with session while another thread is still using it.

how to make data access thread-safe?

use thread-safe data structure → `queue.Queue` or `multiprocessing.Queue` - they make sure only one thread can access a block of code at a time (uses lock objects)

***it’s fast***  → takes like 3-4s. uses multiple threads to have many open requests at the same time. there's an overlap in the waiting times.

any issues?

- requires more code
- can cause race conditions

### asynchronous version → use `asyncio` module for async I/O

well suited for I/O-bound tasks

avoids overhead of context switching between threads

needs only 1 thread

**event loop ⇒ controls how and when each async task gets to execute**

loops through tasks while continuously monitoring them

if some task waits for an I/O, the loop suspends it and immediately switches to another task

once the expected *event* occurs, it switches back/resumes

**coroutine ⇒ similar to thread but lightweight and cheaper to suspend/resume**

less overhead when switching/spawning many more

there should not be a blocking function (synchronous) in your coroutines

```python
import asyncio

# async def: creates a coroutine function
async def main():
	# await: pauses until task in completed
	await asyncio.sleep(5)
```

`requests` library is blocking. so switch to a non-blocking one like `aiohttp`

```python
import asyncio
import time

import aiohttp

async def main():
    sites = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
    ] * 80
    start_time = time.perf_counter()
    await download_all_sites(sites)
    duration = time.perf_counter() - start_time
    print(f"Downloaded {len(sites)} sites in {duration} seconds")

async def download_all_sites(sites):
    async with aiohttp.ClientSession() as session:
        tasks = [download_site(url, session) for url in sites]
        await asyncio.gather(*tasks, return_exceptions=True)

async def download_site(url, session):
    async with session.get(url) as response:
        print(f"Read {len(await response.read())} bytes from {url}")

if __name__ == "__main__":
    asyncio.run(main())
```

code looks very similar to the synchronous one

- scales far better
- takes fewer resources
- less time to create
- it’s really fast

***it’s fast*** → takes like 0.5s 

forces you to think which task will get swapped out → helps you create a better design

*NOTE requires a library meant for async programming like aiohttp and a minor mistake in the code can cause a take to hold the processor*

### process-based version ⇒ using `multiprocessing` module

till now everything runs on a single CPU core because of GIL

how? it creates a separate Python interpreter to run on each CPU

this is a heavyweight op but can make a huge difference on a correct problem

```python
import atexit
import multiprocessing
import time
from concurrent.futures import ProcessPoolExecutor

import requests

session: requests.Session

def main():
    sites = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
    ] * 80
    start_time = time.perf_counter()
    download_all_sites(sites)
    duration = time.perf_counter() - start_time
    print(f"Downloaded {len(sites)} sites in {duration} seconds")

def download_all_sites(sites):
		# by default it determines the no.of CPUs available and sets it to how many proccesses needed
		# using initializer: creates session for each process
    with ProcessPoolExecutor(initializer=init_process) as executor:
        executor.map(download_site, sites)

def download_site(url):
    with session.get(url) as response:
        name = multiprocessing.current_process().name
        print(f"{name}:Read {len(response.content)} bytes from {url}")

def init_process():
    global session
    session = requests.Session()
    atexit.register(session.close) # creates a cleanup function that helps prevent memory leak

if __name__ == "__main__":
    main()
```

*NOTE  If you need to exchange data between your processes, then it’ll require expensive [inter-process communication (IPC)](https://en.wikipedia.org/wiki/Inter-process_communication) and [data serialization](https://realpython.com/python-serialize-data/), which increases the overall cost even further. Besides this, serialization isn’t always possible because Python uses the [`pickle`](https://realpython.com/python-pickle-module/) module under the surface, which supports only a few data types.*

***it takes 3-4s***→ I/O-bound problems aren’t why multiprocessing exists. it exists for CPU-bound tasks

## speeding up CPU-bound programs

with CPU-bound tasks, there’s no waiting time

this is the part where you will use multiprocessing

```python
import time
from concurrent.futures import ProcessPoolExecutor

def main():
    start_time = time.perf_counter()
    with ProcessPoolExecutor() as executor:
        executor.map(fib, [35] * 20)
    duration = time.perf_counter() - start_time
    print(f"Computed in {duration} seconds")

def fib(n):
    return n if n < 2 else fib(n - 2) + fib(n - 1)

if __name__ == "__main__":
    main()
```

this makes parallel execution possible

cons:

- dividing problem into segments can be difficult
- many solutions require communication between processes

## deciding when to use concurrency?

1. decide if you should use concurrency module? understand performance impact and then determine
2. figure out if it is a I/O-bound (spend most time waiting for something to happen) or CPU-bound problem (spend time on processing data or crunching numbers)
3. CPU-bound problems ⇒ process-based concurrency ⇒ `multiprocessing`
4. I/O-bound problems ⇒ “Use `asyncio` when you can, `threading` or `concurrent.futures`when you must” 
    
    *NOTE asyncio ⇒ gives best speed-up but require critical libraries to be ported*
    

## async programming w/ FastAPI

why?

scalable handle many concurrent connections

performant for I/O bound

simple compared to threading

for CPU-bound problem, FastAPI can offload the computation to a thread or process pool

```python
from fastapi import FastAPI
from concurrent.futures import ProcessPoolExecutor
import asyncio

app = FastAPI()

# handled by uvicorn
process_pool = ProcessPoolExecutor()

def cpu_bound_task(n):
    # a CPU-bound task
    total = 0
    for i in range(n):
        total += i * i
    return total

@app.get("/compute/{n}")
async def compute(n: int):

    # offload the CPU-bound task to a separate process
    loop = asyncio.get_running_loop()
    # await: waits to return before completion
    result = await loop.run_in_executor(process_pool, cpu_bound_task, n) # allows FastAPI server to handle other requests while CPU-bound task is handled in parallel
    return {"result": result}
```

in web apps responsiveness is the key. if you run the CPU-bound task directly inside the FastAPI event loop it will block the server from handling other requests. so we offload.
