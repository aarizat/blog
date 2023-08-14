---
author: Andres Ariza-Triana
author_gh_user: aarizat
read_time: 5 mins
publish_date: 07-08-2023
---

# Keeping track of `asyncio.run` execution.

Have you ever been curious about the inner workings of the `#!python asyncio.run()` function? If the answer is no, then get ready for a fascinating journey. This blog post will guide you through the process that takes place behind the scenes.

## Before we begin.
We will be using as reference the `CPython 3.11.4` implementation. Please note that in earlier versions of CPython, the event loop might have been implemented differently. This is an important note because we will be looking at the CPython implementation to see step-by-step the execution proccess followed when `#!python asyncio.run()` function is called.

## `asyncio.run(coro, *, debug=None)`
`#!python asyncio.run()` is used to run a coroutine in the event loop, it also creates a new event loop and closes it at the end.
Allright, what's next ? well, things are getting insteresting here. Actually, `#!python asyncio.run()` is shortcut for:

```py
with asyncio.Runner(debug=True) as runner:
    runner.run(main())
```

This means if we want to know what is happening when `#!python asyncio.run()` is called we shoud go to the `#!python Runner` class. Let's go there: 🙀

???+ note

    I've deleted some code, comments and documentation from the below class, just to make it shorter.

``` py title="cpython/Lib/asyncio/runner.py" linenums="1" hl_lines="21 36 55"
class Runner:

    def __init__(self, *, debug=None, loop_factory=None):
        ...
        self._loop = None
        ...
    ...

    def run(self, coro, *, context=None):
        if not coroutines.iscoroutine(coro):
            raise ValueError("a coroutine was expected, got {!r}".format(coro))

        if events._get_running_loop() is not None:
            raise RuntimeError(
                "Runner.run() cannot be called from a running event loop")

        self._lazy_init()

        if context is None:
            context = self._context
        task = self._loop.create_task(coro, context=context)

        if (threading.current_thread() is threading.main_thread()
            and signal.getsignal(signal.SIGINT) is signal.default_int_handler
        ):
            sigint_handler = functools.partial(self._on_sigint, main_task=task)
            try:
                signal.signal(signal.SIGINT, sigint_handler)
            except ValueError:
                sigint_handler = None
        else:
            sigint_handler = None

        self._interrupt_count = 0
        try:
            return self._loop.run_until_complete(task)
        except exceptions.CancelledError:
            if self._interrupt_count > 0:
                uncancel = getattr(task, "uncancel", None)
                if uncancel is not None and uncancel() == 0:
                    raise KeyboardInterrupt()
            raise  # CancelledError
        finally:
            if (sigint_handler is not None
                and signal.getsignal(signal.SIGINT) is sigint_handler
            ):
                signal.signal(signal.SIGINT, signal.default_int_handler)

    def _lazy_init(self):
        if self._state is _State.CLOSED:
            raise RuntimeError("Runner is closed")
        if self._state is _State.INITIALIZED:
            return
        if self._loop_factory is None:
            self._loop = events.new_event_loop()
            if not self._set_event_loop:
                events.set_event_loop(self._loop)
                self._set_event_loop = True
        else:
            self._loop = self._loop_factory()
        if self._debug is not None:
            self._loop.set_debug(self._debug)
        self._context = contextvars.copy_context()
        self._state = _State.INITIALIZED

    ...
```

Yes, I know that is long class, but we're going to focus on `self.run()` & `self._lazy_init()` methods. As we saw previously, when `asyncio.run` is exuceted actually we're creating an instance of `Runner` class and then we call its `run` instance method.

When the `Runner` class instance is created some instance attributes are created as well, one of most important is `self._loop` which is set to `None`. Now when the instance is created we can call its `run` instance method, for that, we need to pass it a required argument `coro` which is coroutine that we want to run in the event loop. As we see in the `run` implementation (__line 9__) it starts making some validations, like:

1. check if `coro` is actually a coroutine.
   ```py
    if not coroutines.iscoroutine(coro):
        raise ValueError("a coroutine was expected, got {!r}".format(coro))
   ```
2. check if there is already an event loop running.
   ```py
    if events._get_running_loop() is not None:
        raise RuntimeError(
            "Runner.run() cannot be called from a running event loop")
   ```

if the previous checks pass succesfully, so the `self._lazy_init()` is called. This method also makes some checks but the most important here is the __line 55__:

```py
self._loop = events.new_event_loop()
```

The above line creates the event loop and assigs it to the instance variable `self._loop`. Tracking the flow throught the code and assumming that `self._lazy_init()` ran successfully (__line 17__), we can skip some irrelevant lines of code and go to the __line 21__:

```py
task = self._loop.create_task(coro, context=context)
```

it uses the previous created event loop and calls its `create_task` instance method for wrapping the `coro` passed to the `run` method into task.

## Digging a bit into `self._loop.create_task`

To know a bit about what's happening when `self._loop.create_task` is called, we need to go to `BaseEventLoop` class which we can find in the file __base_events.py__. Let's bring the class from there and take a look at it.

???+ note

    I've deleted some code, comments and documentation from the below class, just to make it shorter.

```py title="cpython/Lib/asyncio/base_events.py" linenums="1" hl_lines="7"
class BaseEventLoop(events.AbstractEventLoop):
    ...

    def create_task(self, coro, *, name=None, context=None):
        self._check_closed()
        if self._task_factory is None:
            task = tasks.Task(coro, loop=self, name=name, context=context)
            if task._source_traceback:
                del task._source_traceback[-1]
        else:
            if context is None:
                task = self._task_factory(self, coro)
            else:
                task = self._task_factory(self, coro, context=context)

            tasks._set_task_name(task, name)

        return task
    ...
```
The highlighted line shows that with the coroutine `coro` pass to the `self._loop.created_task` method an instance of `Task` is created with the same coroutine `coro` then, it's returned it at the end of __BaseEventLoop's__ `create_task` method. With this in mind, we can return to the `run` execution.

## Run `coro` until complete.
After `coro` is wrapped into a `Task` (__line 21__), `run` makes some other types of validations and finally the task is passed to `run_until_complete` event loop instance method.

The `run_until_complete` methods run the task passed as parameter until it's finished.


## Sum up!

As you could see, there is nothing special underhood when the `#!python asyncio.run()` function is run, it only created and instance of `#!python Runner` class and then its `#!python run` instance method is executed, when this happens, there are some other methods executed, like `!python create_task` and finally `#!python run_until_complete`.