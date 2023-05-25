---
title: "How I Learned Python Asyncio While Making Coffee"
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

Slowly, I understood that:
1. A single, invisible event loop is created by `asyncio.run` as top-level entry point to run the coroutine `main`.
1. A _coroutine_ is a kind of non-blocking function that can be suspended and resumed; both `main` and `asyncio.sleep` above are coroutines.
1. `async` and `await` are keywords that deal with coroutine declaration and calling respectively.

Delving into the intricate definition is futile and meaningless, at least for now.

The whole point of asyncio is to execute tasks concurrently; one of the most common ways is by using `asyncio.gather`.

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
    print("Brew coffee starts")
    await asyncio.sleep(1)
    print("Brew coffee ends")
    return "Coffee not ready!"

async def make_coffee_wrong():
    start = time.perf_counter()
    result = await asyncio.gather(
        boil_water(), grind_coffee_bean(), brew_coffee()
    )
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds")
    print(result)

asyncio.run(make_coffee_wrong())
```

```sh
$ python make_coffee_wrong.py 
Boil water starts
Grind coffee bean starts
Brew coffee starts
Brew coffee ends
Grind coffee bean ends
Boil water ends
Executed in 3.00 seconds
['Boiled water', 'Ground coffee', 'Coffee not ready!']
```

Obviously, both water boiling and coffee bean grinding need to be completed before the brewing can start! Furthermore, if you prefer a manual pourover coffee method like me, you will be restricted to the brewing process and unable to do anything else concurrently, so `brew_coffee` becomes a blocking I/O.

```python
# make_coffee_correct.py
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
    time.sleep(1)
    print(f"Brew coffee manually with {ingredients} ends")
    return "Coffee ready!"

async def make_coffee():
    start = time.perf_counter()
    result = await asyncio.gather(boil_water(), grind_coffee_bean())
    result.append(brew_coffee(result))
    end = time.perf_counter()
    print(f"Executed in {end-start:0.2f} seconds")
    print(result)

asyncio.run(make_coffee())
```
```sh
$ python make_coffee_correct.py
Boil water starts
Grind coffee bean starts
Grind coffee bean ends
Boil water ends
Brew coffee manually with ['Boiled water', 'Ground coffee'] starts
Brew coffee manually with ['Boiled water', 'Ground coffee'] ends
Executed in 4.00 seconds
['Boiled water', 'Ground coffee', 'Coffee ready!']
```

If I have an automatic coffee brewing machine, the `brew_coffee` function would switch back into a non-blocking operation, functioning as an asynchronous coroutine. Although it would still necessitate waiting for the completion of boiling water and ground coffee preparation, `brew_coffee` could then be executed concurrently with other tasks, such as `toast_bread`.

Some common scenarios of blocking vs non-blocking I/O often encountered in actual development:
* Package for http request - [requests](https://requests.readthedocs.io/en/latest/) versus [httpx](https://www.python-httpx.org/)
* Postgres database driver - [psycopg2](https://www.psycopg.org/) versus [asyncpg](https://magicstack.github.io/asyncpg/current/)
* Database ORM solutions like [SQLAlchemy](https://www.sqlalchemy.org/) (version 1.4 and later supports asyncio) or [Tortoise](https://tortoise.github.io/)

Now, if you will kindly excuse me, I must return to attending my boiling water and make some :coffee:!