---
layout: post
title: Coroutine学习笔记
categories: Tornado
---
### Coroutine如何调用
协程不会像正常函数那样抛出异常，因此正确的调用很重要。比如下面的错误示例：  

`````python  
@gen.coroutine  
def divide(x, y):  
    return x / y  
def bad_call():  
    # This should raise a ZeroDivisionError, but it won't because  
    # the coroutine is called incorrectly.  
    divide(1, 0)  
`````  
coroutine必须被coroutine本身调用，并且用yield。如：  

```python
@gen.coroutine
def good_call():
    # yield will unwrap the Future returned by divide() and raise
    # the exception.
    yield divide(1, 0)
```  
如果你想使用`fire and forget`模式，则可以考虑`IOLoop.spawn_callback_`,`IOLoop`会捕获异常。  

```
IOLoop.current().spawn_callback(divide, 1, 0)
```
如果，你想在脚本等程序中调用`coroutine`，则可以考虑`IOLoop.run_sync`方法  

```
IOLoop.current().run_sync(lambda: divide(1, 0))
```

### Coroutine patterns  

**与`callback`交互**  
用`Task`来包裹回调函数，并传参，如：  

```python
@gen.coroutine
def call_task():
    # Note that there are no parens on some_function.
    # This will be translated by Task into
    # some_function(other_args, callback=callback)
    yield gen.Task(some_function, other_args)
```  

**调用阻塞函数**  
通常在tornado异步中调用阻塞函数是影响性能的，这时可以使用`ThreadPoolExecutor`  

```python
thread_pool = ThreadPoolExecutor(4)
@gen.coroutine
def call_blocking():
    yield thread_pool.submit(blocking_func, args)
```

**并行交互**  
`coroutine`会识别future中所有的dict和list，并等待  

```python
@gen.coroutine
def parallel_fetch(url1, url2):
    resp1, resp2 = yield [http_client.fetch(url1),http_client.fetch(url2)]
@gen.coroutine
def parallel_fetch_many(urls):
    responses = yield [http_client.fetch(url) for url in urls]
    # responses is a list of HTTPResponses in the same order
@gen.coroutine
def parallel_fetch_dict(urls):
    responses = yield {url: http_client.fetch(url)
                         for url in urls}
    # responses is a dict {url: HTTPResponse}
```

**Intervaling**  
confusing....  

**循环**  
Coroutine中做循环是很微妙的，常用的比如`Motor`的`db.collections.find()`操作  

```python
import motor
db = motor.MotorClient().test
@gen.coroutine
def loop_example(collection):
    cursor = db.collection.find()
    while (yield cursor.fetch_next):
         doc = cursor.next_object()
```

**后台运行**  
PeriodicCallback类用来周期的调用一个callback函数，但在coroutine中貌似有问题.  
> （_PeriodicCallback_ is not normally used with coroutines）  
  
  这时可以使用while true和gen.sleep来实现需求。  

```python
@gen.coroutine
def minute_loop():
    while True:
        yield do_something()
        yield gen.sleep(60)
# Coroutines that loop forever are generally started with
# spawn_callback().
IOLoop.current().spawn_callback(minute_loop)
```

**`couroutine`中延迟任务**  
有时需求在某段时间后执行某任务，可以考虑`add_timeout`  

```python
class MyHandler(RequestHandler):
    @asynchronous
    @gen.coroutine
    def get(self):
        self.write("sleeping .... ")
        self.flush()
        # Do nothing for 5 sec
        yield gen.Task(IOLoop.instance().add_timeout, time.time() + 5)
        self.write("I'm awake!")
        self.finish()
```

#### Reference
> Tornado手册