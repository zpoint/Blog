# gunicorn workers

我们在 [第一篇](https://github.com/zpoint/Blog/blob/master/Python/django/django_cn.md) 里已经了解过 gunicorn 的 `SyncWorker` 原理, 现在我们来看下其他的 workers 是如何工作的

![workers](./workers.png)

# 目录

* [eventlet](#Eventlet)
* [gevent](#Gevent)
* [thread](#thread)

* [tornado](#tornado)

## Eventlet

如果你打开 [eventlet](https://eventlet.net/) 的官网

> Eventlet 是一个 Python 网络库, 支持并发访问, 使用这个库可以在不改变代码写法的情况下更改代码的运行方式
>
> * 它使用了 epoll/kqueue/libevent , 这样可以支持 [可扩展的非阻塞式 I/O](http://en.wikipedia.org/wiki/Asynchronous_I/O#Select.28.2Fpoll.29_loops)
> * [协程](http://en.wikipedia.org/wiki/Coroutine)  的支持可以让开发者像使用线程一样编写顺序性代码, 但是运行时又提供了非阻塞IO的运行方式
> * 对于事件的派发/回调是集成在库中的, 开发者不需要关注这部分逻辑, 所以你可以很方便地在 Python 解释器中使用 Eventlet, 或者在一个大型应用的一个模块中使用

`EventletWorker` 继承自 `AsyncWorker`, 它覆写了 `init_process` 方法和 `run` 方法

```python3
def patch(self):
    hubs.use_hub()
    eventlet.monkey_patch()
    patch_sendfile()

def init_process(self):
    self.patch()
    super().init_process()
```

在从主进程 `fork` 之后, `init_process` 方法会调用 `eventlet.monkey_patch()`  , 这个方法会默认把下面的模块替换成 `eventlet` 提供的对应的模块

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

Eventlet 把默认的 IO 模块替换成自己的模块, 当你调用 `socket` 方法时, 你实际上调用的是实现了非阻塞式IO的`_green_socket_modules`模块中的方法

对于每一个 `socket` 的读写操作, 或者 `time.sleep` 操作, 实际上 `eventlet` 把当前的上下文保存起来, 并把当前的 `gthread` 加到等待列表种, 之后调用 pool 去等待下一个可读/可写的 IO 事件 

整体的调用流程和 python3 中的 `async` 方法类似, 但是这种方式几乎没有代码入侵



如果你用 `eventlet` 模式运行你的应用

```python3
gunicorn --workers 2 --worker-class eventlet mysite.wsgi
```

![evenlet](./evenlet.png)

`EventletWorker` will spawn a new `gthread`, which in charge of accept connection from socket, after accept a new connection from socket, the `gthread` pass the django handle function to the `greenpool`, and use the `greenpool` to start the django function

Thanks for `eventlet`, we can simply change `--worker-class` to make our django application blocking IO to nonblocking 

IO

Compare to define `async` function directly, your code can run both in blocking and nonblocking mode, and easier to debug

But defining `async` function with `async` keyword directly, require you to design your code in `async` style from the top down, gives you more power about `async` control. For example, `eventlet` with django parallel two different request, while `async` function is able to parallel different IO operation in the same request

## Gevent

If you visite the official site of [gevent](http://www.gevent.org/)

> gevent is a [coroutine](https://en.wikipedia.org/wiki/Coroutine) -based [Python](http://python.org/) networking library that uses [greenlet](https://greenlet.readthedocs.io/) to provide a high-level synchronous API on top of the [libev](http://software.schmorp.de/pkg/libev.html) or [libuv](http://libuv.org/) event loop.
>
> gevent is [inspired by eventlet](http://blog.gevent.org/2010/02/27/why-gevent/) but features a more consistent API, simpler implementation and better performance. 

The differences

> 1. gevent is built on top of libevent(since 1.0, gevent uses libev and c-ares.)
>    * Signal handling is integrated with the event loop.
>    * Other libevent-based libraries can integrate with your app through single event loop.
>    * DNS requests are resolved asynchronously rather than via a threadpool of blocking calls.
>    * WSGI server is based on the libevent’s built-in HTTP server, making it [super fast](http://nichol.as/benchmark-of-python-web-servers).
> 2. gevent’s interface follows the conventions set by the standard library
> 3. gevent does not have all the features that Eventlet has.

If you had another library (written in C) that used libevent’s event loop and want to integrate them together in a single process, gevent support while eventlet does not

Let's go back to `gunicorn`

`GeventWorker` inherit from `AsyncWorker`, it also override the `init_process` method and `run` method

```python3
def patch(self):
    monkey.patch_all()

def init_process(self):
    self.patch()
    hub.reinit()
    super().init_process()
```

After `fork` from the master process, the `init_process` calls `gevent.monkey()`  , which replace the following modules by the corresponding `gevent` support module

```python3
def patch_all(socket=True, dns=True, time=True, select=True, thread=True, os=True, ssl=True,
              subprocess=True, sys=False, aggressive=True, Event=True,
              builtins=True, signal=True,
              queue=True, contextvars=True,
              **kwargs):
              pass

```

The pattern is similar to [eventlet](#Eventlet), the interface is different, so the actual function being called in `run` is slightly different

```python3
# gunicorn/workers/ggevent.py
from gevent.pool import Pool
from gevent.server import StreamServer

def run(self):
	# ...
	pool = Pool(self.worker_connections)
	# ...
	server = StreamServer(s, handle=hfun, spawn=pool, **ssl_args)
	# ...
	server.start()
```

If you run

```bash
gunicorn --workers 2 --worker-class eventlet mysite.wsgi
```

![gevent](./gevent.png)

The pros and cons of using `gevent` is the same as `eventlet`, we are not repeating it again

If you focus more on performance, or you've  C lib that use libevent’s(or libev) event loop that want to integrate into Python in a single process, consider using `gevent`

If you rely on some specific features on `eventlet` such as `eventlet.db_pool` or `eventlet.processes`, you probably should keep using `eventlet`

## thread

By default `gunicorn` use the `sync` [mode](https://github.com/zpoint/Blog/blob/master/Python/django/django.md), It prefork `workers` number of process and each worker is able to handle one request at a time

`ThreadWorker` inherit from `Worker`, it also override the `init_process` method and `run` method

```python3
def init_process(self):
    self.tpool = self.get_thread_pool()
    self.poller = selectors.DefaultSelector()
    self._lock = RLock()
    super().init_process()

def enqueue_req(self, conn):
    conn.init()
    # submit the connection to a worker
    fs = self.tpool.submit(self.handle, conn)
    self._wrap_future(fs, conn)

def accept(self, server, listener):
    try:
        sock, client = listener.accept()
        # initialize the connection object
        conn = TConn(self.cfg, sock, client, server)
        self.nr_conns += 1
        # enqueue the job
        self.enqueue_req(conn)
    except EnvironmentError as e:
        if e.errno not in (errno.EAGAIN, errno.ECONNABORTED,
                           errno.EWOULDBLOCK):
            raise
            
def run(self):
    # ....
```

We can see that `init_process` create a thread pool, and `accept` just push the established connection to the `queue` inside the `ThreadPool` object

> 1. If there is a concern about the application [memory footprint](https://en.wikipedia.org/wiki/Memory_footprint), using `threads` and its corresponding **gthread worker class** in favor of `workers` yields better performance because the application is loaded once per worker and every thread running on the worker shares some memory, this comes to the expense of some additional CPU consumption.

Let's see an example

```bash
gunicorn --workers 1 --worker-class gthread --threads 2 mysite.wsgi
```

The `--threads` will only affect `gthread` worker class, other worker class will not be affected by `--threads` parameter

![gthread](./gthread.png)

Each worker initialize a `ThreadPool` with size `--threads` threads, whenever the main thread accept a socket object, the object is pushed into the `queue`, and the working thread in `ThreadPool` will pop it from the `queue` and delegate the actual request to django application

## tornado

The last worker class is `tornado`, the code is pretty simple

```python3
# gunicorn/gunicorn/workers/gtornado.py
def init_process(self):
    # IOLoop cannot survive a fork or be shared across processes
    # in any way. When multiple processes are being used, each process
    # should create its own IOLoop. We should clear current IOLoop
    # if exists before os.fork.
    IOLoop.clear_current()
    super().init_process()
    
def run(self):
    # ...
```

The `run` method initlaize monitor utility in `gunicorn` , start a tornado server instance, bind the listening sockets to the tornado server, and finally runs the `IOLoop`

## read more

* [what are you using gevent for?](#https://groups.google.com/g/gevent/c/TelwPl3KgnE)
* [Comparing gevent to eventlet](https://blog.gevent.org/2010/02/27/why-gevent/)
* [Better performance by optimizing Gunicorn config](https://medium.com/building-the-system/gunicorn-3-means-of-concurrency-efbb547674b7)

