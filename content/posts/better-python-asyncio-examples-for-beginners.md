---
title: "Better Python Asyncio Examples for Beginners"
date: 2023-05-14T15:08:46+08:00
tags: [python,async]
---

I remember the first time I read the [official Python asyncio example](https://docs.python.org/3/library/asyncio.html); it was as useful as staring at a blank wall and expecting it to solve world hunger. 

```python
# async_hello.py 
import asyncio

async def main():
    print('Hello ...')
    await asyncio.sleep(1)
    print('... World!')

asyncio.run(main())
```

```sh
$ python async_hello.py
Hello ...
... World!
```

A coroutine is simply a different kind of function (that can be suspended and resumed); `async` and `await` are keyword syntax to deal with it. Diving into the detailed definition and details just make the harder to understand, so we won't.

The whole point of asyncio is to execute tasks concurrently. One way is by using `asyncio.gather`.

```python
# async_tasks.py
import asyncio
import time

async def task_a():
    print("Task A starts")
    await asyncio.sleep(1)  # 1 sec non-blocking task
    print("Task A ends")
    return "A1"

async def task_b():
    print("Task B starts")
    await asyncio.sleep(2)  # 2 secs non-blocking task
    print("Task B ends")
    return "B2"

async def run_tasks_async():
    start = time.perf_counter()
    result = await asyncio.gather(task_a(), task_b())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds") # takes about 2 secs
    print(result)

asyncio.run(run_tasks_async())
```

```sh
$ python async_tasks.py 
Task A starts
Task B starts
Task A ends
Task B ends
Executed in 2.00 seconds
['A1', 'B2']
```

An alternative without the need of `asyncio.gather` is to use `asyncio.create_task`.

```python
# async_tasks_alt.py
import asyncio
import time

async def task_a():
    print("Task A starts")
    await asyncio.sleep(1)  # 1 sec non-blocking task
    print("Task A ends")
    return "A1"

async def task_b():
    print("Task B starts")
    await asyncio.sleep(2)  # 2 secs non-blocking task
    print("Task B ends")
    return "B2"

async def run_tasks_async_alt():
    start = time.perf_counter()

    t_a = asyncio.create_task(task_a())
    print("Task A created...")
    t_b = asyncio.create_task(task_b())
    print("Task B created...")

    print("Do other work here...")
    
    result_a = await t_a
    result_b = await t_b
    end = time.perf_counter()

    print(f"Executed in {end-start:0.2f} seconds") # takes about 2 secs
    print([result_a, result_b])

asyncio.run(run_tasks_async_alt())
```

```sh
$ python async_tasks_alt.py 
Task A created...
Task B created...
Do other work here...
Task A starts
Task B starts
Task A ends
Task B ends
Executed in 2.00 seconds
['A1', 'B2']
```

Most examples would _helpfully_ gave advice like "please don't mix with blocking I/O operations" without clearly demonstrating the how and why. Recently, I spotted some old async code snippets in which yours truly had inadvertently used `time.sleep` instead of `asyncio.sleep` for a retry mechanism. Here's why.

```python
# async_tasks_blocked.py 
import asyncio
import time

async def task_a():
    print("Task A starts")
    time.sleep(1)  # 1 sec BLOCKING task, no longer await asyncio.sleep(1)
    print("Task A ends")
    return "A1"

async def task_b():
    print("Task B starts")
    await asyncio.sleep(2)  # 2 secs non-blocking task
    print("Task B ends")
    return "B2"

async def run_tasks_async_blocked():
    start = time.perf_counter()
    result = await asyncio.gather(task_a(), task_b())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds") # takes about 3 secs!
    print(result)

asyncio.run(run_tasks_async_blocked())
```

```sh
$ python run_tasks_async_blocked.py 
Task A starts
Task A ends
Task B starts
Task B ends
Executed in 3.00 seconds
['A1', 'B2']
```

If the order of the two tasks is reversed, what would be the execution time?

Replace the blocking sleep issue with any of the following and you might discover problems you never knew existed.
1. file read/write operations
1. database read/write operations
1. network operations e.g. HTTP requests