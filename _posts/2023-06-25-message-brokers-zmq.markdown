---
layout: post
title:  "Building with Message Passing"
date:   2023-06-25 05:30:52 +0200
categories: update zmq
---
Today I am going to talk about how and why I used message passing in my [automation projects][home-automation-blog-url] along with a demonstration of the implementation in my [Python library][pylib-url] project.

:incoming_envelope: While this project is an evaluation of my own experience with specific message passing, I would recommend also reading more broadly on [the topic](https://duckduckgo.com/?va=v&t=ha&q=message+passing&ia=web) to get a better idea of best fit for your ideas.

:computer: My first experience with message passing was for a university project on [parallel computing](https://duckduckgo.com/?va=v&t=ha&q=parallel+computing&ia=web), also known as High Performance Computing (HPC). While this project was built on a cluster of CPUs, the concept of [general purpose graphics processing (GPGPU)](https://duckduckgo.com/?q=gpgpu&va=v&t=ha&ia=web) had just been established. At any rate, given the academic focus of my project, [LAM/MPI](https://duckduckgo.com/?q=lam-mpi&va=v&t=ha&ia=web) was the chosen message passing framework. This exists as [OpenMPI][openmpi-url] today.

Years later, when I decided to build my automation projects, I had a few real-world constraints to work within. Devices in my project needed to be deployed close to the sensors and outputs they interact with, i.e. the physical electronics. My initial implementation had only a few, isolated [Raspberry Pi][rpi-url] devices, and were few enough that configuration of individual network addresses was still practical. Based on my prior interest and experience with message passing, I wanted to find out what was on offer many years after my HPC project, and obviously not on HPC clusters :nerd_face:.

### ZeroMQ

After a few web searches, I found [ZeroMQ][zmq-url]. From their product page, I highlighted the features that attracted me to it.

> ZeroMQ (also known as ØMQ, 0MQ, or zmq) looks like an **embeddable** networking library but acts like a **concurrency framework**. It gives you sockets that carry **atomic messages** across various transports like **in-process, inter-process, TCP**, and multicast. You can connect sockets **N-to-N with patterns like fan-out, pub-sub, task distribution, and request-reply**. It's fast enough to be the fabric for clustered products. Its **asynchronous I/O** model gives you scalable multicore applications, built as asynchronous message-processing tasks. It has a **score of language APIs** and runs on **most operating systems**.

For my needs, it was perfect. At the time I was running on various ARM builds of [Raspberry Pi OS](https://en.wikipedia.org/wiki/Raspberry_Pi_OS) and although Python [wheels](https://www.geeksforgeeks.org/what-is-a-python-wheel/) for ZeroMQ weren't always available, building them through [pip](https://pypi.org/project/pip/) installs usually worked fine. Without going into too much detail about dependency management, I've greatly simplified my Python development setup using [poetry](https://python-poetry.org/) :bulb:.

Once I was comfortable with the ZeroMQ [socket API paradigm][zmq-socket-api-url], their helpful boilerplate examples enabled me to add robust communications between my projects over the network. Since ZeroMQ supports blocking and non-blocking behavior for socket-read operations, it is easy for application threads to plan their work around I/O. I have not yet made use of the Python [library](https://aiozmq.readthedocs.io/) that adds `asyncio` support to the ZeroMQ interface.

Since my prior HPC work and general interest in interoperability, I did not want to send [Python pickles](https://docs.python.org/3/library/pickle.html) across the network. I chose [JSON](https://www.json.org/json-en.html) as the message format along with helpers like [simplejson](https://pypi.org/project/simplejson/). This maps JSON off the wire to primitive dictionary (map) and list (array) types. Though marshalling outgoing data into JSON, I also used [Message Pack][msg-pack-url] for the bytes that go on the wire.

Although I didn't need to think about reliably sending messages between my applications, I needed to add a bit of scaffolding to deal with the real-world, particularly around network, application and device disruptions. Relevant to my projects, ZeroMQ has a few rules:

1. ZeroMQ sockets must *not* be shared between threads. Sharing data *between* threads can be achieving using special *in-process* socket types, which can be addressed trivially using unique labels like `inproc://my-socket`. This enables trivial coordination between threads by employing the same blocking and non-blocking semantics of the underlying sockets. The result is that data is shared between threads of an application using exactly the same semantics as if the data was also being sent over the wire. The ZeroMQ Python API provides a method `recv_pyobj` and `send_pyobj` which supports sending Python objects if you want to use this for `inproc` socket types, which I do for these socket types, but usually for built-in or primitive types (strings, integers, tuples).
2. *All* ZeroMQ sockets created across all threads must be closed in order to cleanly exit the application via `zmq.Context().term()` which blocks for every open socket. A thread must manage the socket life-cycle even if it is a daemon thread. I found that this rule also applies if sockets are set with a short "[linger time](http://api.zeromq.org/2-1%3azmq-setsockopt)". Given that I have a soft-spot for orderly shutdown behaviour, I put a lot of work into being able to make this easy to get right *every time*. More on this below.

#### Patterns

Here are the components of my [library][pylib-url] and the Python helpers I created for my applications that use this code and the underlying message passing libraries. When my application threads need a ZeroMQ socket, there are a few patterns that can be used.

**Option 1**: Create and use a socket directly.

This example is pretty straight-forward. Import a `zmq_socket` function from `pylib.zmq` and call it to get a ZeroMQ socket of the desired type.

:outbox_tray: Connect a `PUSH` socket.
{% highlight python %}
from pylib.zmq import zmq_socket

my_pusher = zmq_socket(zmq.PUSH)
my_pusher.connect('inproc://the-best-socket')
my_pusher.send_pyobj((my_tuple_a, my_tuple_b))
my_pusher.close()
{% endhighlight %}

:inbox_tray: Bind a `PULL` socket.
{% highlight python %}
from pylib.zmq import zmq_socket

my_puller = zmq_socket(zmq.PULL)
my_puller.bind('inproc://the-best-socket')
data = my_puller.recv_pyobj()
my_puller.close()
{% endhighlight %}

Let's take a look at what the function `pylib.zmq.zmq_socket` actually does. Using the Python `inspect` module, the `FrameInfo` object for the calling code is retrieved and stored with a weak reference to the ZeroMQ socket in a Python `WeakKeyDictionary`. This means that the line of code for each socket creation is tracked.

:shrug:	Why is this needed? In my experience, when ZeroMQ applications move from prototype to non-trivial, it becomes harder to work out where improper socket lifecycle management is holding up application shutdown. I've wasted enough time on this to want *all the visibility*.

[pylib.zmq][pylib-zmq-url]
{% highlight python %}
import inspect
import zmq
from weakref import WeakKeyDictionary

zmq_sockets = WeakKeyDictionary()
zmq_context = zmq.Context()
zmq_context.setsockopt(zmq.LINGER, 0)

def zmq_socket(socket_type):
    fi = inspect.stack()[-1]
    location = f'{fi.function} in {fi.filename} @ line {fi.lineno}'
    socket = zmq_context.socket(socket_type)
    zmq_sockets[socket] = location
    return socket

def try_close(socket):
    if socket is None:
        return
    try:
        try:
            location = zmq_sockets[socket]
            if location:
                log.info(f'Closing socket created at {location}...')
        except KeyError:
            pass
        socket.close()
    except ZMQError:
        log.warning(f'Ignoring socket error when closing socket.', exc_info=True)
{% endhighlight %}

Later when the application attempts to shutdown, the `thread_nanny` thread method uses this information to report on any sockets that do not close within some grace period (typically 30 seconds). This is incredibly helpful to diagnose issues where a change has inadvertently left a socket open. I needed to know about all ZeroMQ sockets created by the application and so with a *slight* ~~ab~~use of Python's encapsulation permissiveness, I tap into `zmq_context._sockets` to know this. Now it is possible to map the internal sockets to the ones created by the application. Horrible? Maybe :see_no_evil:.

[pylib.threads][pylib-threads-url]
{% highlight python %}
def thread_nanny(signal_handler):
    ...
    try:
        for s in zmq_context._sockets: # type: ignore
            try:
                if s and not s.closed:
                    log.warning(f'Closing lingering socket type {s.TYPE} (push is {zmq.PUSH}, pull is {zmq.PULL}) for endpoint {s.LAST_ENDPOINT}.')
                    try_close(s)
            except ZMQError:
                # not interesting in this context
                continue
    except RuntimeError:
        # protect against "Set changed size during iteration", try again later
        pass
{% endhighlight %}

:writing_hand: In a future post, I'll be talking about the resilience features at the application level and will come back to the rest of the behaviour in [pylib.threads][pylib-threads-url].


**Option 2** Extend `AppThread` from [pylib.app](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/app.py#L15C25-L15C25) and `Closable` from [pylib.zmq](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/zmq.py#L46) and call `get_socket`. Notice the use of the `exception_handler` Python [context manager](https://docs.python.org/3/library/contextlib.html).

`exception_handler` deserves a bit of attention because it pulls a lot of functionality together even though it's actually rather compact. It takes the following arguments:
* `connect_url`: the ZeroMQ URL to *bind* or *connect* depending on `socket_type` set to `PULL` or `PUSH` respectively.
* On `__exit__`:
* `closable` closes the `Closable` or a socket created for the lifespan of the context manager.
* `and_raise`: re-raise any exception.
* `close_on_exit`: close the underlying socket.
* `shutdown_on_error`: shutdown the application on error.
* there are two exceptions that trigger specific behaviour. `zmq.error.ContextTerminated` is thrown if a socket operation is attempted after `zmq.Context().term()` is called. If caught, a socket-close operation is attempted and the context manager will return without error because handing it is pointless in that scope if the application is already shutting down. I've appropriated Python's `ResourceWarning` to trigger a situational issue that may provoke a shutdown but without error.

Using the pattern below, the only other code needed is the thread-specific processing of data received by the thread.
{% highlight python %}
from pylib import threads
from pylib.app import AppThread
from pylib.zmq import Closable
from pylib.handler import exception_handler

class MyThread(AppThread, Closable):

    def __init__(self):
        AppThread.__init__(self, name=self.__class__.__name__)
        Closable.__init__(self, connect_url='inproc://my-socket', socket_type=zmq.PULL)

    def run(self):
        with exception_handler(closable=self, and_raise=False, shutdown_on_error=True):
            while not threads.shutting_down:
                data_for_me = self.socket.recv_pyobj()
                ...
                # processing happens here
                ...
{% endhighlight %}

**Option 3**: If using the Pipeline Pattern by chaining PUSH/PULL socket pairs across threads, extend `ZmqRelay` in [pylib.app](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/app.py#L26) and implement the `process_message` function. Notice that `ZmqRelay` also [uses](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/app.py#L48C14-L48C31) the `exception_handler` and so no special exception handling is needed in `process_message`. Any exceptions will be handled for the thread.

{% highlight python %}
class MyPipelineThread(ZmqRelay):

    def __init__(self):
        ZmqRelay.__init__(self,
            name=self.__class__.__name__,
            source_zmq_url='inproc://my-source',
            source_socket_type=zmq.PULL,
            sink_zmq_url='inproc://my-sink',
            sink_socket_type=zmq.PUSH)

    def startup(self):
        pass

    def process_message(self, zmq_socket):
        (my_tuple_a, my_tuple_b) = zmq_socket.recv_pyobj()
        ...
        # processing happens here
        ...
        self.socket.send_pyobj((my_result_a, my_result_b))
{% endhighlight %}

### Message Brokers

ZeroMQ adds value without a broker server to coordinate activities, leaving addressing of nodes to the application author. Given how my early infrastructure was built on ZeroMQ, I understand how to use its strengths and I continue to use it today, though it is limited to inter-thread communication. In part 2, I'll talk more about the other half of my message passing mechanisms that require a broker server, in particular, [RabbitMQ][rabbit-url] and [MQTT][mqtt-url].

[esp-url]:                  https://www.espressif.com/
[home-automation-blog-url]: https://tailucas.github.io/update/2023/06/18/home-automation.html
[mosquitto-url]:            https://mosquitto.org/
[mqtt-url]:                 https://mqtt.org/
[mqttx-url]:                https://mqttx.app/
[msg-pack-url]:             https://msgpack.org/
[openmpi-url]:              https://www.open-mpi.org/
[pylib-threads-url]:        https://github.com/tailucas/pylib/blob/master/pylib/threads.py
[pylib-url]:                https://github.com/tailucas/pylib
[pylib-zmq-url]:            https://github.com/tailucas/pylib/blob/master/pylib/zmq.py
[python-url]:               https://www.python.org/
[rabbit-gsg-url]:           https://www.rabbitmq.com/getstarted.html
[rabbit-url]:               https://www.rabbitmq.com/
[rpi-url]:                  https://www.raspberrypi.com/
[zmq-socket-api-url]:       https://zeromq.org/socket-api/
[zmq-url]:                  https://zeromq.org/
