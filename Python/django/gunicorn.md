# django gunicorn gevent greenlet daphne What are they? 

# Part2 gunicorn workers

We've learned `SyncWorker` for gunicorn in part1, now let's see how other workers work

![workers](./workers.png)

## Eventlet

If you visite the official site of [eventlet](https://eventlet.net/)

> Eventlet is a concurrent networking library for Python that allows you to change how you run your code, not how you write it.
>
> - It uses epoll or kqueue or libevent for [highly scalable non-blocking I/O](http://en.wikipedia.org/wiki/Asynchronous_I/O#Select.28.2Fpoll.29_loops).
> - [Coroutines](http://en.wikipedia.org/wiki/Coroutine) ensure that the developer uses a blocking style of programming that is similar to threading, but provide the benefits of non-blocking I/O.
> - The event dispatch is implicit, which means you can easily use Eventlet from the Python interpreter, or as a small part of a larger application.

`EventletWorker` inherit from `Worker`, it override the `init_process` method and `run` method

```python3
def patch(self):
    hubs.use_hub()
    eventlet.monkey_patch()
    patch_sendfile()

def init_process(self):
    self.patch()
    super().init_process()
```

After `fork` from the master process, the `init_process` calls `eventlet.monkey_patch()`  , which replace the following modules by the corresponding `eventlet` support module by default

```python3
for name, modules_function in [
    ('os', _green_os_modules),
    ('select', _green_select_modules),
    ('socket', _green_socket_modules),
    ('thread', _green_thread_modules),
    ('time', _green_time_modules),
    ('MySQLdb', _green_MySQLdb),
    ('builtins', _green_builtins),
    ('subprocess', _green_subprocess_modules),
]
```

Eventlet replaced to default IO module by it's `green` module, when you calls the `socket` function, you are actually calling `_green_socket_modules`  , which implements nonblocking IO

On every `socket` read/write, or `time.sleep`, it actually save the current context and add the current gthread to the pooling list, and then calls pool to wait for next ready IO event

It's like the `async` keyword in python3, but with less code invasion



If you run your app in eventlet mode

```python3
gunicorn --workers 2 --worker-class eventlet mysite.wsgi
```

![evenlet](./evenlet.png)

`EventletWorker` will spawn a new `gthread`, which in charge of accept connection from socket, after accept a new connection from socket, the `gthread` pass the django handle function to the `greenpool`, and use the `greenpool` to start the django function

Thanks for `eventlet`, we can simply change `--worker-class` to make our django application blocking IO to nonblocking 

IO

Compare to define `async` function directly, your code can run both in blocking and nonblocking mode, and easier to debug

But defining `async` function with `async` keyword directly, require you to design your code in `async` style from the top down, gives you more power about `async` control. For example, `eventlet` with django parallel two different request, while `async` function is able to parallel different IO operation in the same request
