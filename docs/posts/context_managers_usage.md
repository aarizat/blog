---
title: Useful Applications of Python Context Managers.
author: Andres Ariza-Triana
author_gh_user: aarizat
read_time: 5 min
publish_date: August 14, 2023
---


As everyone could know, Python Context Managers are a type of object that we can use in a `with` block. What it does is to execute some clean up code after exiting from `with` block. it's mostly used to make sure to free up resources when a exception occurs inside a `with` block.

You could be familiar with this example:
```python
with open('my_file.ext', 'rw') as file:
    # proccess file

assert file == file.closed # True
```

the above snippet of code opens a file and then make some operations in it, but underhood when the `with` block ends it makes sure that the file is correctly closed, actually, if an exception happens while proccessing the file it also closes it properly. This isn't only used for context managers, you can use it for dealing with database connections and networking stuff.

## Writting context managers

There two ways to write context mangers in Python, the first one is to write a Python class which implements the two dunder methods `__enter__` and `__exit__`, like so:

```python
class MyContextManager:
    def __init__(self, filename, mode)
        self.filename = filename
        self.mode = mode

    def __enter__(self)
        self.file = open(self.filename, mode=self.mode)
        return self.file

    def __exit__(self, exc_type, exc_value, exc_traceback)
        if self.file:
            self.file.close()
```

Another way to achieve the same thing is using the `contextlib`, like this:
```python
from contextlib import contextmanager


def my_context_manager(filename, mode):
    file = open(filename, mode=mode)
    try:
        yield file
    finally:
        file.closed()
```

The two previous implementations are doing similar things.


## Other useful usages of Context Managers.

### 1. Update an environment variable temporary
Imagine that you have a Python function which is using an environment variable to make some type of operation.

```python
def use_env_variable(arg1, arg2, ...):
    # some code before
    variable = os.environ.get('MY_VAR')
    # some code after which uses the environment variable `variable`
    ...
```

But you realize that the environment variable doesn't have the right value before executing `use_env_variable` function, and also, that you cannot update `MY_VAR` because other functions depend on the old value of `MY_VAR`.

A way to solve this problem woud be:

- Before executing `use_env_variable`:
    - Get the current value of environment variable and store it in a new variable
    - Set the environment variable to new value

- Run `use_env_variable`:
    - It uses the environment variable with the new value

- After executing `use_env_variable`:
    - Use the old environment variable fetched before executing the function and set it to the environment variable again.

- Run more code.

In code this would be:

```python
old_value = os.environ.get('MY_VAR')
os.environ['MY_VAR'] = 'My new value'

use_env_variable(arg1, arg2, ..)

os.environ['MY_VAR'] = old_value
```

The previous solution does its work, that's what we need. But now imagine that just like `use_env_variable`, there are other functions that also need to do the same things and other code needs to use the old value, we can continue using the previous solution but the code gets meessy. Here context mangers come in handy.

To avoid using the previous solution, We can write a context manager which temporarily updates the environment variable. When entering the `with` block, the variable is set to a new value. Upon exiting, the variable is reverted to its original value.

Let's code it:

???+ note

    We'll write the context manager using the `contextlib` libray for the sake of simplicity.


```python
from contextlib import contextmanager


@contextmanager
def set_environ(var_name, new_value):
    """Set a `new_value`` to the environment variable `var_name`"""
    old_value = os.environ.get(var_name)
    os.environ[var_name] = new_value
    try:
        yield
    finally:
        os.environ[var_name] = old_value
```

now, we can use our previous context manager in a `with` block to update an environment variable temporarly.

```python
with set_environment('MY_VAR', 424):
    # use functions that depen on the new value of the environment variable
    ...

print(os.environ.get('MY_VAR')) # it should print the old value, not 424.
```

it was clear and beatiful solution using context manager, isn't it ?

### 2. Timing Python code
The second useful usage of a context manager is for timing how long a python piece of code takes to run.

Imagine that you want to time how long some python functions in your code take to run, a solution would be to use the python `time` module, something like so:

```python
import time


start = time.time()

run_function()

elasep = time.time() - start
```

The previous solution works okay, but we could use a context manager to achieve the same thing in a fancy and reusable way. Let's code this by using the classic Context Manager.

```python
import time


class Timer:
    def __init__(self):
        self._start = None
        self._end = None

    def __enter__(self):
        self._start = time.time()

    def __exit__(self, exc_type, exc_value, exc_traceback):
        self._end = time.time()
        print(f'it takes {self._end - self._start} to finish!')
```

We can use the above `Timer` context manager to measure the execution time of our functions. For example, to determine the runtime of a function called `square` that returns a list of squared integers, we'd use it as follows.

```python
def square(array):
    return [num**2 for num in array]


with Timer():
    square([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
```

After running the previous snippet we should see on the standard out the following:

```shell
it took 8.344650268554688e-06 to finish! # (1)
```

1.  The time that this function took to run on your machine should be different.


Look at how easy was to write a reausable context manager to time our python code. 😎
