---
layout: post
title:  "Building for Resilience"
date:   2023-07-01 05:30:52 +0200
categories: update availability
---
Today I'm going to talk about how I build resilience into my personal projects.

:thinking: This is **not** a post about testing or [writing testable code](https://duckduckgo.com/?q=writing+testable+code&va=v&t=ha&ia=web). I'll claim that my strategy of having "fun" with my personal projects limited my investment in the area of unit and integration testing at home. I may write about this topic in future but meanwhile you can find a significant wealth of information [online](https://duckduckgo.com/?va=v&t=ha&q=software+testing&ia=web) or specifically on [Python](https://duckduckgo.com/?q=python+testing&va=v&t=ha&ia=web).

### Concepts

[Resilience Engineering](https://duckduckgo.com/?q=resilience+engineering&va=v&t=ha&ia=web) is a big topic. In my projects I've defined resilience as having these properties:
1. Takes predictable action for either application or dependency failures, and recovers to steady-state when the disruption is resolved.
2. Provides adequate visibility for new scenarios that are not well handled by existing logic, and without noise when dependencies are disrupted.

#### Constraints and Assumptions

I also apply an important [simplifying assumption](https://duckduckgo.com/?va=v&t=ha&q=simplifying+assuptions&ia=web) in my project design: disruptions (usually outside of main event loops and in daemon threads) trigger an application shutdown [with an expectation of a supervised restart]. This dramatically reduces the number of recovery scenarios that I need to specifically handle in code if I want the application to fully return to an unimpaired state. An example of an impaired state would be thread death.

{:refdef: style="text-align: center;"}
![Switch it off and on again](https://i.imgflip.com/1mnwik.jpg)
{: refdef}

:warning: There are many real-world scenarios where this is almost certainly undesirable behaviour from automation because it explicitly introduces correlated disruption either across a horizontally scaled application or across applications sharing a common strategy. It also assumes that application restart carries neither impact to the end-user experience nor system resource cost at startup, which is a bad assumption in real-world systems at scale. Substantial, real-time systems tend to have more exhaustive exception handling mechanisms to obviate the need for restart-on-failure and usually also employ some kind of traffic throttling mechanisms to protect available resources. Exception handling is made robust by unit and regression testing to explicitly exercise recovery code paths. It is because exceptions are exceptional by definition, this should be the absolute minimum due diligence when building resilience off the common-path.

:robot:	There are legitimate examples of forced-restart in the form of [watchdog timers](https://duckduckgo.com/?q=watchdog+timer&va=v&t=ha&ia=web) (WDT) typically used in low-level applications like micro-controllers. In these environments it is more useful to trigger a reboot of the processor with the expectation of post-reboot success than to leave a device in a stuck or impaired state. It also means that all setup and initialization code can be run during startup paths, keeping exception paths simple.

Since my personal projects carry none of these constraints and incidentally benefit from [non-redundant] [message brokers](https://tailucas.github.io/update/2023/06/30/message-brokers-rabbitmq.html), short disruptions for restarts should not actually drop any un-fetched work and may only delay event processing for a few seconds during application restart.

#### Desired Traits

With that, let's discuss a few of the resilience features I wanted in all my applications.

##### Steady State

When an application is running in the steady state, it has:

1. A uniform interface for logging activity.
2. Unhandled exception capturing with the ability to define additional [integrations](https://docs.sentry.io/platforms/python/guides/flask/configuration/integrations/) as needed.
3. Automatic detection of thread death with a means of responding appropriately.

##### Shutdown

When an application is to shutdown or in the process of shutting down:

1. Unless busy in application logic or I/O, a thread must shutdown *immediately* when it is signaled by an application to do so. *All* blocking behavior, including sleeps, must allow interruption. As far as possible, this behavior is also followed by daemon threads to support clean shutdown of ZeroMQ which requires that all sockets be closed to allow the main thread to terminate.
2. The application logger switches to debug logging automatically if shutdown takes longer than 30 seconds. This also includes a listing of all remaining ZeroMQ sockets along with their code instantiation locations. This gives the application a chance to report on what is delaying the shutdown before the process manager later follows `TERM` with `KILL`.
3. If an unhandled exception is responsible for application shutdown, the process exit code is set to non-zero (typically `1`). This seems obvious, but isn't necessarily the behaviour for non-trivial applications.
4. A process supervisor must restart the application for exits where the code is not `0`. It is of course possible to make the process manager restart the child process under all conditions but I've found it useful for environment debugging to be able to send a `TERM` signal to the app and to keep the supervisor from bringing back the application in the container.

### Monitoring and Observability

This is a big topic and worth spending time doing [research](https://duckduckgo.com/?va=v&t=ha&q=observability&ia=web) online. The IBM cloud blog has a useful [distinction](https://www.ibm.com/cloud/blog/observability-vs-monitoring) between the concepts of monitoring and observability.

> Monitoring tells you when something is wrong, while observability can tell you what’s happening, why it’s happening and how to fix it.

Here is a brief list of the mechanisms I use in my projects, both self-made and borrowed from helpful tools online.

:telephone: It's useful to develop an early opinion about which tools you need local and/or network-isolated vs tools that can be reached on the Internet because it will determine your monitoring strategy and almost certainly also the setup and operating cost.

I'll first introduce the functional properties of these mechanisms and then in the later section I'll show how this is used in code.

* [cronitor.io][cronitor-url]: Using [cronitor-python][cronitor-python-url] for in-process monitoring, I post periodic metrics from my thread monitor [thread_nanny](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/threads.py#L100-L111) and report on thread counts and missing threads as part of the metadata. This allows me to explicitly monitor cases where threads have died unexpectedly.
* [healthchecks.io][healthchecks-url]: All my container projects include a cron job to run a shell script to invoke `curl` to call project-specific URLs in the Healthchecks web service. Calls to the URL are tracked by healthchecks and overdue calls are alerted via your chosen set of many [integrations](https://healthchecks.io/docs/). While Healthchecks do support in-process probes, I explicitly want mine sent via `cron` to verify that my container application has a properly functioning `cron` instance for other jobs like cleanup or backups.
* [InfluxDB][influxdb-url]: I post a variety of time-series data from my various projects. Influx provide both a containerized project for network-local instances as well as alerting capabilities based on Influx QL queries. The main purpose of this is to visualize my data in the form of metrics. Excellent alternatives include [Grafana](https://grafana.com/) or [Prometheus](https://prometheus.io/docs/introduction/overview/).
* [PagerDuty][pagerduty-url]: While I don't interact with this service directly from code, the monitoring services above do include PagerDuty as an integration option and so this provides resilience in *communication* of the issue. Good paging tools provide both communication (push notification, phone call) and team escalations (another human).
* [sentry.io][sentry-url]: This service is the single reason that I am able to achieve resilience in my personal projects without bothering with automated testing or tailing logs for every single unexpected issue. One of the many features I use with Sentry is an in-process mechanism for capturing unhandled exceptions. When triggered, Sentry will automatically create a unique ticket for the issue, including a variety of metadata such as local context and call-tracing breadcrumbs. With Sentry, I've been able to rapidly identify unintuitive failure modes and add robustness to my implementation where an application restart (to fix) is either inappropriate or unnecessary.
* [Telegram][telegram-url]: While not a monitoring tool as such, Telegram provides a bot interface to easily communicate rich context about actions taken by the application or other discretionary information. The monitoring tools above also have the ability to send notifications to a Telegram group.

:bulb: **Top Tip**: Host reboots and network disruptions of your service flush out hard-to-find issues because it includes testing your code and all the library, system and network dependencies that you [can and should] take for granted. Test both controlled and uncontrolled shutdown of containers and host systems. It will teach you some valuable lessons, I guarantee it.

### Putting it Together

In the same way that my [pylib][pylib-url] project contains a variety of code factored out of my projects over time, I discovered that it was useful to do the same with the project structure of my container applications. You can find an example in my [base-app][base-app-url] project which uses `pyblib` as a package dependency (installed as a git submodule). I designed this project to also be stand-alone to test basic functionality of a working application. This gives me confidence that Docker projects that extend this project have inherited functionality that is already tested. The examples below take from both of these projects.

#### Logging

With `APP_NAME` being defined in `__init__.py` (for example [here](https://github.com/tailucas/base-app/blob/43838b1e34beaabeb36cc14964b24b563d9d0c7f/app/__init__.py#L2)), a Python [StreamHandler](https://docs.python.org/3/library/logging.handlers.html#streamhandler) is created in [pylib](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/__init__.py#L27-L44) `__init__.py` to include both the application name and thread name. By default, the system log is used but otherwise the console is used which is useful when running the application interactively.

{% highlight python %}
log = logging.getLogger(APP_NAME)
log.propagate = False
log.setLevel(logging.DEBUG)
formatter = logging.Formatter('%(name)s %(threadName)s [%(levelname)s] %(message)s')
log_handler = None
if os.path.exists('/dev/log'):
    log_handler = logging.handlers.SysLogHandler(address='/dev/log')
    log_handler.setFormatter(formatter)
    log.addHandler(log_handler)
if sys.stdout.isatty() or ('SUPERVISOR_ENABLED' in os.environ and log_handler is None):
    log.warning("Using console logging because there is a tty or under supervisord.")
    log_handler = logging.StreamHandler(stream=sys.stdout)
    log_handler.setFormatter(formatter)
    log.addHandler(log_handler)
{% endhighlight %}

It's typically convenient to have all container logs sent to a central remote logging service. On Linux, these logs can be easily forwarded to the host system `rsyslog` instance by using the [logging driver](https://github.com/tailucas/base-app/blob/43838b1e34beaabeb36cc14964b24b563d9d0c7f/docker-compose.template#L6-L7) in your `docker-compose.yml` template.

```yaml
version: "3.8"
services:
  app:
    logging:
      driver: syslog
```

I happen to use [solarwinds papertrail](https://www.papertrail.com/solution/tips/how-to-configure-remote-syslog/) for off-box log persistence but the free tier does not tolerate logs that are too chatty.

#### Unhandled Exceptions

Sentry makes this so easy that there's very little to say about it in terms of code. If you want to explicitly forward an exception as a ticket to Sentry, you can use the following pattern.

{% highlight python %}
from sentry_sdk import capture_exception

def some_function():
    try:
        # ...
    except NetworkError:
        # ... handling specific error
    except Exception:
        # ... catch-all
        capture_exception()
{% endhighlight %}

It is important to note that Sentry will still detect unhandled exceptions via your logger without having to always use the call to `capture_exception` as above. You can also install a Sentry filter for a logger namespace in order to prevent triggering tickets for cases where there is application-level handling. I found that I needed this to filter some exception noise in RabbitMQ. Of course, filtering should be used with care to prevent masking real issues.

{% highlight python %}
from sentry_sdk.integrations.logging import ignore_logger
ignore_logger('pika.adapters.utils.io_services_utils')
{% endhighlight %}

Here is an example of using Sentry with integrations. In this example, the HTTP 500 handler for Python [Flask](https://flask.palletsprojects.com/) is updated with information to enable a [feedback form](https://github.com/tailucas/event-processor/blob/master/templates/error.html) to post to Sentry. A user-friendly way to admit failure.

{% highlight python %}
import sentry_sdk
from sentry_sdk import last_event_id
from sentry_sdk.integrations.flask import FlaskIntegration

sentry_sdk.init(
    dsn=creds.sentry_dsn,
    integrations=[FlaskIntegration()]
)

@flask_app.errorhandler(500)
def internal_server_error(e):
    return render_template('error.html',
                           sentry_event_id=last_event_id(),
                           sentry_dsn=creds.sentry_dsn
                           ), 500
{% endhighlight %}

#### Process Management

Some kind of process manager is needed to control and monitor execution of your application. I've had prior success with [systemd](https://systemd.io/) but for my container applications I currently use [supervisord](http://supervisord.org/) which is loaded as part of my container entrypoint. By using this syntax below, I replace the execution context of the Docker entrypoint with supervisord as the root process.

{% highlight ruby %}
exec env supervisord -n -c /opt/app/supervisord.conf
{% endhighlight %}

Supervisord has some helpful configuration templates and good documentation on default values and possible overrides. Since all my applications use a common pattern, they all use [this stanza](https://github.com/tailucas/base-app/blob/43838b1e34beaabeb36cc14964b24b563d9d0c7f/config/supervisord.conf#L53-L57):

```
[program:app]
command=poetry run python -m app
directory=/opt/app/
user=app
autorestart=unexpected
```

The [program stanza](http://supervisord.org/configuration.html#program-x-section-values) has this to say about `autorestart` which I've set to `unexpected`.

> If unexpected, the process will be restarted when the program exits with an exit code that is not one of the exit codes associated with this process’ configuration...

If, for whatever reason, the root process fails or the container exits unexpectedly due to an environment issue, Docker can also be configured with a rule regarding what to do with the container. In [my example](https://github.com/tailucas/event-processor/blob/bcca7e27c238cb783abf2102a339e2efcc11a7c8/docker-compose.template#L6), I use `unless-stopped` which will restart the container on any condition other than an explicit stop, including starting the container at host boot.

{% highlight yaml %}
version: "3.8"
services:
  app:
    restart: unless-stopped
{% endhighlight %}

#### Helpers

I have also built a few patterns in [pylib][pylib-url] for common error handling. Python's [context manager](https://docs.python.org/3/library/contextlib.html#examples-and-recipes) allows for a convenient way to support sophisticated but scoped activity life cycle management. I've applied this pattern in my [exception_handler](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/handler.py#L16C12-L16C12) which takes appropriate action based on the activity block.

These work in tandem with another module [pylib.threads](https://github.com/tailucas/pylib/blob/master/pylib/threads.py) which contains the thread nanny (aptly named `thread_nanny`) mentioned earlier as well as a few [thread trackers](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/threads.py#L20-L27) with some ~~ab~~use of a handful of globals that rely on Python's [threading.Event](https://docs.python.org/3/library/threading.html#event-objects). Any instantiation of [AppThread](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/app.py#L15-L23) automatically registers as threads to track by the nanny.

Let's take an end-to-end example from the [base-app](https://github.com/tailucas/base-app/blob/43838b1e34beaabeb36cc14964b24b563d9d0c7f/app/__main__.py#L78) entrypoint which also installs a [signal handler](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/process.py#L23-L41). This lays down all the code necessary to start the application and worker threads which continue work until the application signals a shutdown.

{% highlight python %}
from pylib.process import SignalHandler
from pylib.threads import thread_nanny, die, bye
from pylib.app import AppThread
from pylib.zmq import zmq_term, Closable
from pylib.handler import exception_handler

class EventProcessor(AppThread, Closable):

    def __init__(self):
        AppThread.__init__(self, name=self.__class__.__name__)
        Closable.__init__(self, connect_url='inproc://my-zeromq-in-process-socket')

    def run(self):
        with exception_handler(closable=self, and_raise=False, shutdown_on_error=True):
            while not threads.shutting_down:
                event = self.socket.recv_pyobj()
                log.debug(event)
                # other processing


def main():
    # only log at INFO level
    log.setLevel(logging.INFO)
    # ensure proper signal handling; must be main thread
    signal_handler = SignalHandler()
    # create the application worker thread
    event_processor = EventProcessor()
    # start the thread nanny with signal handler
    nanny = threading.Thread(
        name='nanny',
        target=thread_nanny,
        args=(signal_handler,),
        daemon=True)
    try:
        event_processor.start()
        # start thread nanny
        nanny.start()
        # main thread now waits on the shutdown latch
        threads.interruptable_sleep.wait()
        raise RuntimeWarning()
    except(KeyboardInterrupt, RuntimeWarning, ContextTerminated) as e:
        log.warning(str(e))
        threads.shutting_down = True
        # ensure the latch is set if we arrive here due to another issue
        threads.interruptable_sleep.set()
    finally:
        # tell ZeroMQ to shutdown (blocks on any remaining open sockets)
        zmq_term()
    # exists the Python process with an exit code dependent on exceptions thrown
    bye()


if __name__ == "__main__":
    main()
{% endhighlight %}

Here's a little more detail about [how](https://github.com/tailucas/pylib/blob/master/pylib/threads.py) `exception_handler` does its job, particularly around `__exit__` behaviour:

{% highlight python %}
from sentry_sdk import capture_exception
from zmq.error import ContextTerminated
from . import threads
from .threads import die
from .zmq import Closable, try_close

class exception_handler(object):

    def __init__(self, closable: Closable = None, connect_url=None, socket_type=None, and_raise=True, close_on_exit=True, shutdown_on_error=False):
        self._closable = closable
        self._zmq_socket = None
        self._zmq_url = connect_url
        self._socket_type = socket_type
        self._and_raise = and_raise
        self._close_on_exit = close_on_exit
        self._shutdown_on_error = shutdown_on_error

    def __enter__(self):
        # ...

    def __exit__(self, exc_type, exc_val, tb):
        if self._close_on_exit or (exc_type and issubclass(exc_type, ContextTerminated)):
            if self._closable:
                self._closable.close()
            elif self._zmq_socket:
                try_close(self._zmq_socket)
        if exc_type is None:
            return True
        if issubclass(exc_type, ContextTerminated):
            # treat as non-critical
            return True
        elif issubclass(exc_type, ResourceWarning):
            # raised to indicate a fatal dependency error that
            # does not fill Sentry with exception regressions
            # or unhandled exceptions; used typically at startup
            log.warning(self.__class__.__name__, exc_info=True)
            if self._shutdown_on_error:
                die(exception=exc_type)
        elif issubclass(exc_type, Exception):
            if not threads.shutting_down:
                log.exception(self.__class__.__name__)
                capture_exception(error=(exc_type, exc_val, tb))
                if self._shutdown_on_error:
                    die(exception=exc_type)
            else:
                # log the exception as informational if in debug mode
                log.debug(self.__class__.__name__, exc_info=True)
        return not self._and_raise
{% endhighlight %}

When the context manager closes, the `__exit__` method is called by the Python runtime. If there is a ZeroMQ socket or `Closable` associated with the context manager, an attempt is made to close it. If the context manager has no exception context, denoted by the `exc_type` parameter, then the context manager is exited with a return (True indicates that it will not be re-raised to the calling code). If there is an exception on exit:
1. a ZeroMQ `ContextTerminated` exception, which happens when a ZeroMQ socket operation is attempted after calling `zmq.Context().term()`, then this is treated as non-critical; handling it is pointless because the application is shutting down.
2. I ~~ab~~use Python's built-in `ResourceWarning` as a placeholder for an unrecoverable error (like dependency failure) that should trigger an application shutdown but *without* capturing an error in Sentry because there is nothing to debug in the application code. Of course, the dependency needs its own monitoring. :point_right: I've found [Uptime Kuma](https://github.com/louislam/uptime-kuma) a good option for this.
3. For any (unhandled) `Exception` type, capture the error in Sentry if the application isn't already shutting down. The `die()` method captures this to use in the exit code for the process.
4. Re-raise the exception if the context manager is used with the parameter `and_raise` is set to `True`.

[base-app-url]:        https://github.com/tailucas/base-app
[blog-balena-url]:     https://tailucas.github.io/update/2023/06/11/iot-with-balena-cloud.html
[blog-rabbit-url]:     https://tailucas.github.io/update/2023/06/30/message-brokers-rabbitmq.html
[cronitor-python-url]: https://pypi.org/project/cronitor/
[cronitor-url]:        https://cronitor.io/
[docker-url]:          https://www.docker.com/
[healthchecks-url]:    https://healthchecks.io/
[influxdb-url]:        https://www.influxdata.com/
[pagerduty-url]:       https://www.pagerduty.com/pricing/incident-response/
[pylib-url]:           https://github.com/tailucas/pylib
[sentry-url]:          https://sentry.io/for/python/
[telegram-url]:        https://telegram.org/
