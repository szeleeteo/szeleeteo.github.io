---
title: "A Better Python Asyncio Example for Beginners"
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

Naturally, I looked for better examples tucked away elsewhere and refactored them to my own understanding.

```python
# async_tasks.py 
import asyncio
import time

async def task_a():
    print("Task A starts")
    await asyncio.sleep(1) # non-blocking
    print("Task A ends")

async def task_b():
    print("Task B starts")
    await asyncio.sleep(1) # non-blocking
    print("Task B ends")

async def run_tasks_async():
    start = time.perf_counter()
    await asyncio.gather(task_a(), task_b())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds")

asyncio.run(run_tasks_async())
```

```sh
$ python async_tasks.py 
Task A starts
Task B starts
Task A ends
Task B ends
Executed in 1.00 seconds
```

Recently, I came across some old code snippets in which yours truly had inadvertently used `time.sleep` within a coroutine :face_palm:. 

(By the way, any function that uses `async def` syntax is a coroutine - it can be paused and resumed during its execution.)

Nevertheless, applying this mistake to the original code - substituting `asyncio.sleep` with `time.sleep` and studying its output - is more insightful and self-explanatory than the lengthy advice of _"please use asynchronous non-blocking I/O, avoid blocking I/O operations"_. 

```python
# async_tasks_blocked.py 
import asyncio
import time

async def task_a():
    print("Task A starts")
    time.sleep(1) # blocking
    print("Task A ends")

async def task_b():
    print("Task B starts")
    time.sleep(1) # blocking
    print("Task B ends")

async def run_tasks_async_blocked():
    start = time.perf_counter()
    await asyncio.gather(task_a(), task_b())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds")

asyncio.run(run_tasks_async_blocked())
```

```sh
$ python run_tasks_async_blocked.py 
Task A starts
Task A ends
Task B starts
Task B ends
Executed in 2.01 seconds
```

Replace the sleep block that takes some time with any of the following and you might discover problems you never knew you had.
1. file read/write operations
1. database read/write operations
1. network operations e.g. HTTP requests

At the expense of dishing out yet another vague advice, it finally dawned on me the meaning of [it's turtles all the way down](https://en.wikipedia.org/wiki/Turtles_all_the_way_down). 

:turtle:<br>:turtle::turtle:<br>:turtle::turtle::turtle:<br>:turtle::turtle::turtle::turtle: