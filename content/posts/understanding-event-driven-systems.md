+++ 
date = 2019-06-28T21:11:46+02:00
title = "Understanding Event Driven Systems"
slug = "understanding-event-driven-systems" 
tags = []
categories = []
description = "Learn how event driven asynchronous non blocking i/o works with the help of simple python code snippets."
+++

(Cross posted, originally on medium (https://medium.com/@saswatachakravarty/6bc167f59d40)

Event driven system architectures have become become really popular in the last decade. It is at the heart of high performance web servers such as NGINX. The good performance of Node.js can be credited to its asynchronous event driven runtime. There are reactive programming frameworks such as RxJava available today which helps developers build event driven systems. So what is this event driven paradigm and how does it help bring efficiency? This article will help you understand everything about the terms event driven, asynchronous and non blocking I/O using simple python code snippets.

Consider a typical scenario that is encountered in a micro-services environment— you have some function which makes multiple API requests to downstream services, collects the results and does some manipulation before returning the result to the client. Here is a toy model —

The downstream service provides an API /get_resource , which has a latency of 1 seconds which we model by letting it sleep for 1 seconds.

```python
from flask import Flask
import time

downstream_service = Flask(__name__)


@downstream_service.route('/get_resource')
def get_resource():
    time.sleep(1)
    return 'Hello World!'

if __name__ == '__main__':
    downstream_service.run(port=8080)
```

The application code needs to make two requests to this API.

## Serial Calls

The naive approach is to make two requests one after the other, which costs approximately 2 seconds to execute.

```python
import urllib.request
import time


def call_downstream():
    return urllib.request.urlopen('http://localhost:8080/get_resource').read()


start = time.time()
call_downstream()
call_downstream()
end = time.time()
print("Time taken for serial requests:", end - start)
```

Time taken for serial requests: 2.011678457260132

## Using ThreadPools

The two calls to the downstream service are independent of one another. So instead of waiting 1 seconds for the first request to complete, we can make the requests in parallel. This is typically accomplished using thread pools. We submit the work of calling the downstream service to a background thread managed by a thread pool executor. The main thread does not get blocked, and goes on to submit the next call to the downstream service into the thread pool. It then waits for the result. This approach takes approximately 1 second to execute, as expected.

```python
import time
import urllib.request
from concurrent.futures import ThreadPoolExecutor


def call_downstream():
    return urllib.request.urlopen('http://localhost:8080/get_resource').read()


executor = ThreadPoolExecutor()

start = time.time()
future_1 = executor.submit(call_downstream)
future_2 = executor.submit(call_downstream)
future_1.result()
future_2.result()
end = time.time()
print("Time taken using thread pool:", end - start)
```

Time taken using thread pool: 1.0046939849853516

So far, all this is familiar. Now let us ask ourselves the question- can we achieve the same execution time of approximately 1 seconds by making parallel calls using just a single thread?

## Using Event Driven Model

It seems counter-intuitive at first — how can you make parallel calls using just a single thread? Event driven models helps us achieve this, as the following sections will make clear. In order to understand this, we must first take a quick peek into how communication takes place over HTTP to the downstream service.

At a high level, when our application sends a request downstream it first creates an HTTP connection, which is backed by a socket. It then sends the request and then keeps listening on the socket for the response to arrive. Once the response has arrived, it will read it off the socket. All this is encapsulated in the urlopen function call.

Lets start with implementing the downstream call with the low level http APIs of python.

```python
import http.client

def call_downstream_low_level():
    conn = http.client.HTTPConnection("localhost:8080")
    conn.request("GET", "/get_resource")
    res = conn.getresponse()
    return res.read()
```

In the function above, the main bottleneck comes from the conn.getresponse() call , which blocks for 1 second waiting for the reply to come from the downstream service. Wouldn’t it be nice if somehow the function gets paused, other useful tasks are performed and then function is resumed only when the response is ready? This is exactly what we are going to achieve. Before making the blocking conn.getresponse() call, we will voluntarily pause and give back the control of execution to the caller of the function, and pass it back the connection socket on which we are waiting to read. This is done with the help of the yield statement.

```python
import http.client
def call_downstream_async():
    conn = http.client.HTTPConnection("localhost:8080")
    conn.request("GET", "/get_resource")
    yield conn.sock
    res = conn.getresponse()
    return res.read()
```

Next we will define an event loop which will process the tasks (which in our case is the task of calling downstream) present in a queue, and execute them when they are ready to.

```python
def run(tasks):
    # List of sockets which are waiting for data to arrive
    # along with their corresponding task
    waiting_to_read = {}
    # Main event loop, continue to process events while they exist.
    while any([tasks, waiting_to_read]):
        # If there are no tasks in the queue, check for sockets
        # on which data has arrived.
        while not tasks:
            ready_to_read, _, _ = select.select(
                waiting_to_read, [], [])
            # For the sockets on which data is available,
            # remove them from the waiting queue
            # and add the task back to the list.
            for socket in ready_to_read:
                tasks.append(waiting_to_read.pop(socket))
        task = tasks.popleft()
        try:
            # The socket on which the task is waiting
            sock = next(task)
            # Add the socket to waiting queue
            waiting_to_read[sock] = task
        except StopIteration as e:
            print("Finished task: ", task.__name__)
            print("Returned value: ", e.value)
```

Here is how it works. It maintains an internal queue of sockets which are waiting for data to arrive. As long as there are items present in the task queue or the waiting queue, it will continue to process them.

Our tasks are calls to the function **call_downstream_async()** , which returns a generator. The generator returns a socket on the first next call and exits on the subsequent call. The event loop gets the socket and places them in a wait queue. It checks whether it is ready for read with the help of the select function call, which is delegated to this unix system call. Once it is ready, the task is placed back on the main tasks queue, so that call_downstream_async can resume execution, now that the data is available in the socket to read.

Finally, in our application code, we create the task queue and call the run function to execute them.

```python
start = time.time()
tasks = deque()
tasks.append(call_downstream_async())
tasks.append(call_downstream_async())
run(tasks)
end = time.time()
print("Time taken using event loop: ", end - start)
```

Time taken using event loop: 1.0052828788757324

Thus we see that we are able to achieve execution of 1 seconds using a single thread, with the event driven paradigm.

## Event Driven Vs Threads

You may ask, what is the benefit of using the event driven model, with its added complexity, over just using a thread pool, given that we achieve the same execution time of 1 seconds?

Using multiple threads is more expensive that using a single thread. The costs involved in the context switching between the threads adds up. The degree of parallelism depends on the number of threads, which in turn has to be chosen according to the number of available cpu cores.

Let us compare the timings with 100 requests -

```python
executor = ThreadPoolExecutor()
start = time.time()
futures = [executor.submit(call_downstream) for _ in range(100)]
results = [future.result() for future in futures]
end = time.time()
print("Time taken using thread pool:", end-start)
start = time.time()
tasks = deque()
for _ in range(100):
    tasks.append(call_downstream_async())
run(tasks)
end = time.time()
print("Time taken using event loop: ", end - start)
```

Time taken using thread pool: 3.0826776027679443

Time taken using event loop: 1.0724318027496338

In this case, using an event loop turns out to be more efficient. The result should be taken with a grain of salt though — the task in these example is completely I/O bound, involving minimum cpu work. However, in real life scenarios the workload will typically have some more compute component. Computational tasks can be parallelized only using multiple threads or processes, so most real life systems will have both event driven asynchronous I/O as well have thread pools or process pools to take advantage of multiple cores of the machine.

## Conclusion

In the above examples, we helped develop intuition for how event driven systems work and achieve parallelism using a single thread by using constructs like non-blocking calls, voluntarily yielding control (also known as co-operative multitasking) and using a event loop.

The examples have a lot of scope for improvement — it is not clear how to collect the return values of the asynchronous functions, what happens if there dependencies among the calls to the downstream service, the event loop implementation is task specific; does this mean we need to write our own event loops for our applications? Luckily there are frameworks available which help address these needs. Exploring some of the frameworks will be a topic for a future post.

------------------------------------------

## Acknowledgement

[1] David Beazley — Python Concurrency From the Ground Up: LIVE! — PyCon 2015
