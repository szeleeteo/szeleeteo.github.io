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

Slowly, I learned that:
1. A single, invisible event loop is created by `asyncio.run` as top-level entry point to run the coroutine `main`.
1. A _coroutine_ is a kind of non-blocking function that can be suspended and resumed; both `main` and `asyncio.sleep` above are coroutines.
1. `async` and `await` are keywords that deal with coroutine declaration and calling respectively.

Delving into the intricate definition is futile and meaningless, at least for now.

The whole point of asyncio is to execute tasks concurrently, e.g. by using `asyncio.gather`.

```python
# async_tasks.py
import asyncio
import time

async def task_a():
    print("Task A starts")
    await asyncio.sleep(2)  # 2 sec non-blocking task
    print("Task A ends")
    return "A1"

async def task_b():
    print("Task B starts")
    await asyncio.sleep(4)  # 4 secs non-blocking task
    print("Task B ends")
    return "B2"

async def task_c():
    print("Task C starts")
    await asyncio.sleep(1)  # 1 secs non-blocking task
    print("Task C ends")
    return "C3"

async def run_tasks_async():
    start = time.perf_counter()
    result = await asyncio.gather(task_a(), task_b(), task_c())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds") # 4 secs
    print(result)

asyncio.run(run_tasks_async())
```

```sh
$ python async_tasks.py 
Task A starts
Task B starts
Task C starts
Task C ends
Task A ends
Task B ends
Executed in 4.00 seconds
['A1', 'B2', 'C3']
```

An alternative without the need of `asyncio.gather` is to use `asyncio.create_task`.

```python
# async_tasks_create.py
import asyncio
import time

async def task_a():
    print("Task A starts")
    await asyncio.sleep(2)  # 2 secs non-blocking task
    print("Task A ends")
    return "A1"

async def task_b():
    print("Task B starts")
    await asyncio.sleep(4)  # 4 secs non-blocking task
    print("Task B ends")
    return "B2"

async def run_tasks_async_create():
    start = time.perf_counter()

    t_a = asyncio.create_task(task_a())
    print("Task A created...")
    t_b = asyncio.create_task(task_b())
    print("Task B created...")

    print("Do other work here...")
    
    result_a = await t_a
    result_b = await t_b
    end = time.perf_counter()

    print(f"Executed in {end-start:0.2f} seconds") # 4 secs
    print([result_a, result_b])

asyncio.run(run_tasks_async_create())
```

```sh
$ python async_tasks_create.py 
Task A created...
Task B created...
Do other work here...
Task A starts
Task B starts
Task A ends
Task B ends
Executed in 4.00 seconds
['A1', 'B2']
```

Most examples would _helpfully_ gave advice like "please don't mix with blocking I/O operations" without clearly demonstrating the how and why. Recently, I spotted some old async code snippets in which yours truly had inadvertently used `time.sleep` instead of `asyncio.sleep` for a retry mechanism.

```python
# async_tasks_blocked.py 
import asyncio
import time

async def task_a():
    print("Task A starts")
    time.sleep(2)  # 2 secs BLOCKING task!
    print("Task A ends")
    return "A1"

async def task_b():
    print("Task B starts")
    await asyncio.sleep(4)  # 4 secs non-blocking task
    print("Task B ends")
    return "B2"

async def task_c():
    print("Task C starts")
    await asyncio.sleep(1)  # 1 secs non-blocking task
    print("Task C ends")
    return "C3"

async def run_tasks_async_blocked():
    start = time.perf_counter()
    result = await asyncio.gather(task_a(), task_b(), task_c())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds") # 6 secs!
    print(result)

asyncio.run(run_tasks_async_blocked())
```

```sh
$ python run_tasks_async_blocked.py 
Task A starts
Task A ends
Task B starts
Task C starts
Task C ends
Task B ends
Executed in 6.01 seconds
['A1', 'B2', 'C3']
```
While non-blocking tasks (like B and C) are cooperative to give way to one and another; blocking tasks (like A) are basically road hoggers.

Replace the blocking task A above with any of the following and you might discover problems you never knew existed.
1. file read/write operations
1. database read/write operations
1. network operations e.g. HTTP requests

Basically, it's turtles all the way down.

:turtle:<br>
:turtle::turtle:<br>
:turtle::turtle::turtle:<br>
:turtle::turtle::turtle::turtle: