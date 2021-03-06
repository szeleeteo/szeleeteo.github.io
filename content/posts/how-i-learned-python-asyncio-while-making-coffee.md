---
title: "How I Learned Python Asyncio While Making Coffee"
date: 2023-05-14T15:08:46+08:00
tags: [python,async]
draft: true
---
## Hello World Async
I remember the first time I read the [official Python asyncio example](https://docs.python.org/3/library/asyncio.html); it was as useful as staring at a blank wall and expecting it to solve world hunger. 

```python
# hello_async.py 
import asyncio

async def main():
    print('Hello ...')
    await asyncio.sleep(1)
    print('... World!')

asyncio.run(main())
```

```sh
$ python hello_async.py
Hello ...
... World!
```

Slowly, I understood that:
1. A single event loop is created by `asyncio.run` as top-level entry point.
1. That entry point allows asynchronous tasks to run, such as the `main` coroutine above.
1. A _coroutine_ is a kind of non-blocking function that can be suspended and resumed; both `main` and `asyncio.sleep` above are coroutines.
1. `async` and `await` are keywords that deal with coroutine declaration and calling respectively.

In other words, there is a giant while loop that allows for functions to take turn to run, pause and resume that makes concurrency possible.

## Gather Tasks
The whole point of asyncio is to execute tasks concurrently because it is usually faster than rather in sequence.

The most straightforward way to do so is by using `asyncio.gather`.

```python
# gather_tasks.py
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

async def gather_tasks():
    start = time.perf_counter()
    result = await asyncio.gather(task_a(), task_b(), task_c())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds") # 4 secs
    print(result)

asyncio.run(gather_tasks())
```

```sh
$ python gather_tasks.py 
Task A starts
Task B starts
Task C starts
Task C ends
Task A ends
Task B ends
Executed in 4.00 seconds
['A1', 'B2', 'C3']
```

## Create Tasks
An alternative way is to use `asyncio.create_task`.

```python
# create_tasks.py
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

async def create_tasks():
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

asyncio.run(create_tasks())
```

```sh
$ python create_tasks.py 
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

## Tasks Dependencies and Blocking I/O
Most examples would _helpfully_ advise on "please don't mix with blocking I/O operations" without clearly demonstrating the how and why. 

Imagine you're making a cup of coffee. In a linear approach, you would wait for the water to boil before grinding the coffee beans and then wait again for the brewing process to complete. It's clearly an inefficient step-by-step process with a lot of waiting time in between.

In asyncio coffee-making, while waiting for the water to boil, you can start grinding the beans. You take advantage of the waiting time to make progress on other tasks. However, the potential pitfall is that not all tasks can be run concurrently.

```python
# make_coffee_wrong.py 
import asyncio
import time

async def boil_water():
    print("Boil water starts")
    await asyncio.sleep(3)
    print("Boil water ends")
    return "Boiled water"

async def grind_coffee_bean():
    print("Grind coffee bean starts")
    await asyncio.sleep(2)
    print("Grind coffee bean ends")
    return "Ground coffee"

async def brew_coffee():
    print("Brew coffee manually starts")
<<<<<<< HEAD
    await asyncio.sleep(1)
=======
    time.sleep(1) # blocking function
>>>>>>> d62c3b5 (Set post to draft)
    print("Brew coffee manually ends")
    return "Coffee not ready!"

async def make_coffee():
    start = time.perf_counter()
    result = await asyncio.gather(
        boil_water(), grind_coffee_bean(), brew_coffee()
    )
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds")
    print(result)

asyncio.run(make_coffee())
```

```sh
$ python make_coffee_wrong.py 
Boil water starts
Grind coffee bean starts
Brew coffee manually starts
Brew coffee manually ends
Grind coffee bean ends
Boil water ends
Executed in 3.00 seconds
['Boiled water', 'Ground coffee', 'Coffee not ready!']
```

Obviously, both water boiling and coffee bean grinding need to be completed before the brewing can start! 

Furthermore, if you prefer a manual pourover coffee method like me, you will be stuck to the brewing process and unable to do anything else concurrently. 

In this scenario, `brew_coffee` becomes a blocking function. It will cause the event loop to stop and disallow other coroutines to run in the background. 

Let's say I have another task `toast_bread` which also can only be run after `boil_water` and `grind_coffee_bean` are completed (due to limited power socket). We are unable to run both `brew_coffee` and `toast_bread` concurrently.

```python
# make_coffee_toast_blocked.py
import asyncio
import time

async def boil_water():
    print("Boil water starts")
    await asyncio.sleep(3) 
    print("Boil water ends")
    return "Boiled water"

async def grind_coffee_bean():
    print("Grind coffee bean starts")
    await asyncio.sleep(2) 
    print("Grind coffee bean ends")
    return "Ground coffee"

def brew_coffee(ingredients):
    print(f"Brew coffee manually with {ingredients} starts")
    time.sleep(1) # blocking!
    print(f"Brew coffee manually with {ingredients} ends")
    return "Coffee ready!"
    
async def toast_bread():
    print("Toast bread starts")
    await asyncio.sleep(0.9)
    print("Toast bread ends")
    return "Toast bread ready!"
    
async def make_coffee_toast():
    start = time.perf_counter()
    result = await asyncio.gather(boil_water(), grind_coffee_bean())
    result.append(brew_coffee(result))
    result.append(await toast_bread())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds")
    print(result)

asyncio.run(make_coffee_toast())
```

```sh
$ python make_coffee_toast_blocked.py
Boil water starts
Grind coffee bean starts
Grind coffee bean ends
Boil water ends
Brew coffee manually with ['Boiled water', 'Ground coffee'] starts
Brew coffee manually with ['Boiled water', 'Ground coffee'] ends
Toast bread starts
Toast bread ends
Executed in 4.90 seconds
['Boiled water', 'Ground coffee', 'Coffee ready!', 'Toast bread ready!
```

## Converting from Blocking to Non-Blocking
On the other hand, if I have an automatic coffee brewing machine, the `brew_coffee` function could turn into a non-blocking operation, functioning as an asynchronous coroutine. Although it would still necessitate waiting for the completion of boiling water and ground coffee preparation, `brew_coffee` could then be executed concurrently with `toast_bread`.

```python
# make_coffee_toast_concurrent.py
import asyncio
import time

async def boil_water():
    print("Boil water starts")
    await asyncio.sleep(3)
    print("Boil water ends")
    return "Boiled water"

async def grind_coffee_bean():
    print("Grind coffee bean starts")
    await asyncio.sleep(2)
    print("Grind coffee bean ends")
    return "Ground coffee"

async def brew_coffee(ingredients):
    print(f"Brew coffee automatically with {ingredients} starts")
    await asyncio.sleep(1)
    print(f"Brew coffee automatically with {ingredients} ends")
    return "Coffee ready!"

async def toast_bread():
    print("Toast bread starts")
    await asyncio.sleep(0.9)
    print("Toast bread ends")
    return "Toast bread ready!"

async def make_coffee_toast():
    start = time.perf_counter()
    result1 = await asyncio.gather(boil_water(), grind_coffee_bean())
    result2 = await asyncio.gather(brew_coffee(result1), toast_bread())
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds")
    print(result1+result2)

asyncio.run(make_coffee_toast())
```
```sh
$ python make_coffee_toast_concurrent.py
Boil water starts
Grind coffee bean starts
Grind coffee bean ends
Boil water ends
Brew coffee automatically with ['Boiled water', 'Ground coffee'] starts
Toast bread starts
Toast bread ends
Brew coffee automatically with ['Boiled water', 'Ground coffee'] ends
Executed in 4.00 seconds
['Boiled water', 'Ground coffee', 'Coffee ready!', 'Toast bread ready!']
```

Common scenarios where the issue of blocking vs non-blocking I/O are often encountered in real development:
* Package for http request - [requests](https://requests.readthedocs.io/) versus [httpx](https://www.python-httpx.org/) and [aiohttp](https://docs.aiohttp.org/)
* Postgres database driver - [psycopg2](https://www.psycopg.org/) versus [asyncpg](https://magicstack.github.io/asyncpg/)
* Database ORM solutions like [SQLAlchemy](https://www.sqlalchemy.org/) (version 1.4 and later supports asyncio) or [Tortoise](https://tortoise.github.io/)

Now, if you will kindly excuse me, I must return to attending my boiling water to make some coffee!
