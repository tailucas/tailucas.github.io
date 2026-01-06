---
layout: post
title:  "Automation with Message Brokers"
date:   2023-06-18 08:30:52 +0200
categories: update
---
Today I am going to talk about how I built my home automation projects.

:bulb: I recommend first reading both parts of my posts on [ZeroMQ][zmq-blog-url] and [RabbitMQ][rabbitmq-blog-url] message passing to introduce concepts and core implementation in my projects. It will make this post much easier to digest.

### Foreword

:point_right: "Automation" in this context is my specific use of the technologies discussed here to solve my own automation challenges. At present, there is no integration with sensible frameworks like [openHAB][oh-url] or [Home Assistant][ha-url] in my projects. The goal behind these projects was a learning opportunity by employing some specific technologies and my opinion on design. The parts you *may* find useful are touch-points with third-party libraries like [Flask][flask-url], [ZeroMQ][zmq-url], [RabbitMQ][rabbit-url], [SQLAlchemy][sqlalchemy-url], [Telegram's][telegram-url] (well documented) [Python Bot][telegram-bot-url], Python libraries like `asyncio`, and Docker containerization because seamless behavior comes after *much* trial and error.

:incoming_envelope:	While writing this post, it became apparent that I need to actually discuss more about my *choice* of the ZeroMQ and RabbitMQ message broker frameworks. MQTT is much less controversial because it is better known and effectively ubiquitous for IoT information exchange. In a future post, I'll discuss how I came to use ZeroMQ and why I augmented it with RabbitMQ when ZeroMQ has a perfectly robust protocol for exchange over the wire.

:nut_and_bolt: This project really just focuses on functional relationships between the components of this system, and at a relatively high level. While this is a useful complement to the projects, there is little detail about how robustness is achieved in the implementation. I will write a separate post about this in future.

:rotating_light: While also technically an implementation detail, visibility and monitoring deserves its own discussion since they build on the messy realities of imperfect dependencies (software *and* hardware). This topic will definitely have a dedicated post in the future.

:computer: If you want to skip straight to the implementation, you can visit each of the projects in Github with fairly comprehensive README which includes some additional detail about the class relationships.

* [Event Hub][event-processor-url]
* [IP Camera snapshots][snapshot-processor-url]
* [ADC and I/O expander on Raspberry Pi][remote-monitor-url]

### The Problem

The problem being solved was to use off-the-self software and hardware to build a home security appliance. As a hobbyist, I wanted to spend the least amount of money to get something functional. I had some [Raspberry Pi][rpi-url], [Arduino][arduino-url], [ESP8266 and ESP32][esp-url] boards in my electronics parts and so the rest was "just" putting it together with some inputs, such as IP Camera snapshots, passive infrared (PIR) sensors and standard magnetic reed switches. In terms of user experience, I wanted to have a web dashboard that would work on either mobile or desktop. I haven't yet had a need to develop mobile applications for this purpose but I was aware that Telegram supported a bot API which provided a good basis for basic control as well was a notification mechanism richer than simple (and expensive) SMS.

{:refdef: style="text-align: center;"}
![Overview](/assets/blog/automation/automation_overview.png)
{: refdef}

### User Experience

To reach the dashboard, I needed some kind of reverse proxy. I quickly discovered the [ngrok][ngrok-url] free-tier. The main requirement is that I did not want to expose an Internet-facing virtual-server port on my router and with my ISP moving to [carrier-grade NAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT) a direct route to my router IP was no longer possible. Using the recently published [ngrok client for Python][ngrok-python-url], the ngrok tunnel is easy to enable for your favourite embedded server, which I have started to incorporate into another project. The ngrok free-tier generates a new, unpredictable tunnel DNS record but the local ngrok management port provides an easy way to determine this for which I have a [thread](https://github.com/tailucas/event-processor/blob/bcca7e27c238cb783abf2102a339e2efcc11a7c8/app/__main__.py#L1868-L1882) called `CallbackUrlDiscovery`. The discovered URL is then forwarded to me over a notification message. I've noticed that ngrok now offer a single, static DNS name for free which makes it better for use with a DNS alias.

Incidentally, [Tailscale][tailscale-url] solves this problem in a way that also covers other common networking challenges but I'll leave a deep-dive into Tailscale for another time. There are many [examples](https://duckduckgo.com/?va=v&t=ha&q=tailscale&ia=images) online about how it can be used. Suffice it to say that I'm a huge fan of Tailscale.

The dashboard itself makes use of a combination of Python [Flask][flask-url] to serve up pages, with [Bootstrap][bootstrap-url] and [Font Awesome][fontawesome-url] for styling with relatively little effort.

### Persistence

Beyond the statically deployed configuration, the project needs to be able to store some basic configuration for user preference, such as which devices were enabled, i.e. could generate trigger events. I had previously used a combination of [SQLite][sqlite-url] and [AWS DynamoDB][aws-url] for configuration management but I've settled on [SQLAlchemy][sqlalchemy-url] on SQLite for configuration with a cron job to create periodic backups of the database to [AWS S3][aws-url].

### Architecture

The diagram below is a bit of a mashup of architecture and sequence diagram. My aim is to introduce the most important classes responsible for message handling and event processing in my [Event Hub][event-processor-url] project. The project includes another useful diagram in the README to distinguish between classes for this project versus base classes in my [library][pylib-url]. Other classes and methods not shown here are mostly self-explanatory with a code inspection. An instance of `ZMQListener` (extends `MQConnection`) is responsible for handling incoming RabbitMQ messages and an instance of `MqttSubscriber` (extends `AppThread`) does the same for MQTT messages. All the devices in my project are configured to send both trigger messages and heartbeat messages on some fixed interval. When either of these instances receive a message, they pass through an instance of `HeartbeatFilter` which extends a delegate I created called `ZMQRelay`. This uses [ZeroMQ][zmq-url] for sharing data between application threads. Once the `HeartbeatFilter` has associated the message with an existing device to keep track of devices last seen, the message is then forwarded to the main application thread in an instance of `EventProcessor` which extends `MQConnection` to publish trigger decisions back to the devices that host outputs like controlling security appliances.

{:refdef: style="text-align: center;"}
![Event Processor](/assets/blog/automation/automation_event-processor.png)
{: refdef}

My choice of ZeroMQ dates back to some of the earliest work one on this project and deserves its own future post. For the purposes of this discussion, I want to show briefly how the message passing is actually done between threads of the above sequence diagram. Consider the diagram below which focuses on just the job of processing MQTT events.

{:refdef: style="text-align: center;"}
![Event Processor MQ](/assets/event-processor/event-processor_zmq_sockets.png)
{: refdef}

These application threads use two types of ZMQ sockets for inter-thread communication. To receive a message from another thread, the `EventProcessor` instance creates and *binds* a `PULL` socket type with a special in-process address of `inproc://app-worker`, a label that uniquely identifies the message sink to other threads. Anything sent to this address will be forwarded to this socket, using the semantics defined a the `PULL` socket type. For the purposes of this post, we'll treat a `PUSH` socket as a *sender* and a `PULL` socket as a *receiver*. All messages sent to the main application loop in `EventProcessor` arrive through this `PULL` socket. In this MQTT example, the `EventProcessor` also creates and *connects* a `PUSH` socket with an `inproc` address of `inproc://mqtt-publish`. Can you guess where messages sent on this socket go?

{:refdef: style="text-align: center;"}
![Square Hole](https://i.ytimg.com/vi/Yh6TFAwX-QA/hqdefault.jpg)
{: refdef}

That's right. Any thread that pushes messages to this address will arrive at a corresponding `PULL` socket created and bound in the `MqttSubscriber` instance which contains the concrete MQTT Python client implementation. MQTT messages arriving to the client from the broker are forwarded to another `PUSH` socket addressing `inproc://heartbeat-nanny` :thinking:. As described above, this gives the `HeartbeatFilter` the chance to update the sender's heartbeat and then forward it on to `EventProcessor` addressed at `inproc://app-worker` completing the event life-cycle :relieved:.

One of the features of ZeroMQ is that you don't need more code to sequence messages between threads, this is done for you. It sounds complicated but when an application becomes non-trivial, I found that it takes a whole class of problems off your attention. More on ZeroMQ in a future post.

Next, we move onto the next project for [IP Camera snapshots][snapshot-processor-url] with another mashup diagram. This project contains an embedded [FTP server](https://github.com/tailucas/snapshot-processor/blob/84394fbbcdb9402696720b1c6bf67586d77dcdd1/app/__main__.py#L839) in order to host snapshot pushes from an IP camera. If camera-sourced motion detection is enabled, each camera is configured to push updates to a path on the FTP server that uniquely identifies the input. Alternatively, a snapshot can be fetched from the IP camera on demand, if this project receives a RabbitMQ control message. In either case, image data is stored on a local data volume for archival to [Google Drive](https://github.com/tailucas/snapshot-processor/blob/84394fbbcdb9402696720b1c6bf67586d77dcdd1/app/__main__.py#L402). For either retrieval method, image data can be sent for object detection. I've experimented with a variety of options, including inference on the [Jetson Nano][jetson-url] development kit. The current implementation makes use of [AWS Rekognition][rekognition-url] for object detection. More details can be found in the [project readme][snapshot-processor-url].

Similar to the Event Hub project above, the message processing chain is illustrated in the sequence diagram. This shows how a trigger message that arrives on an instance of `ZMQListener` (extends `MQConnection`) is forwarded to a chain of `ZMQRelay` instances, one for creating the snapshot (`Snapshot`) and one for object detection (`ObjectDectector`). The benefit of using the `ZMQRelay` [pattern](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/app.py#L26) is that the application code need only deal with the task at hand with message passing details and error handling is hidden. After object detection is done, the output message is forwarded to a `RabbitMQRelay` (extends `AppThread`) to send the RabbitMQ message on the network.

{:refdef: style="text-align: center;"}
![Snapshot Processor](/assets/blog/automation/automation_snapshot-processor.png)
{: refdef}

The next diagram again illustrates the message flow between the application threads using ZeroMQ. A RabbitMQ message arrives from the network to and instance of [`ZMQListener`](https://github.com/tailucas/snapshot-processor/blob/84394fbbcdb9402696720b1c6bf67586d77dcdd1/app/__main__.py#L896-L901) which is rather confusingly named because it actually receives RabbitMQ messages and relays them to the application code via a ZeroMQ `PUSH` socket to the IPC address `inproc://app-worker`. The implementation for this can be found [here](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/rabbit.py#L140). The event data then goes through the ZeroMQ chains [`Snapshot`](https://github.com/tailucas/snapshot-processor/blob/84394fbbcdb9402696720b1c6bf67586d77dcdd1/app/__main__.py#L230), [`ObjectDetector`](https://github.com/tailucas/snapshot-processor/blob/84394fbbcdb9402696720b1c6bf67586d77dcdd1/app/__main__.py#L752) and finally [`RabbitMQRelay`](https://github.com/tailucas/snapshot-processor/blob/84394fbbcdb9402696720b1c6bf67586d77dcdd1/app/__main__.py#L904-L909) which then dispatches the output message to the RabbitMQ exchange.

{:refdef: style="text-align: center;"}
![Snapshot Processor MQ](/assets/snapshot-processor/snapshot-processor_zmq-sockets.png)
{: refdef}

Another piece of this system is the [ADC and I/O expander on Raspberry Pi][remote-monitor-url]. Apart from all the software and deployment scaffold, this is a relatively simple electronics project. You can find some additional information in my [previous post][balena-blog-url] about additional use of Balena Cloud for deployment and the hardware choice.

{:refdef: style="text-align: center;"}
![Remote Monitor](/assets/blog/automation/automation_remote-monitor.png)
{: refdef}

A few other [ESP-based][esp-url] projects not shown here can be found in my [previous post][mongoose-blog-url] which discuss the [MQTT][mqtt-url] inputs and outputs.

[1p-url]:               https://developer.1password.com/docs/connect/
[arduino-url]:          https://docs.arduino.cc/hardware/uno-rev3
[aws-url]:              https://aws.amazon.com/
[balena-blog-url]:      https://tailucas.github.io/update/2023/06/11/iot-with-balena-cloud.html
[bootstrap-url]:        https://getbootstrap.com/
[cronitor-url]:         https://cronitor.io/
[docker-url]:           https://www.docker.com/
[esp-url]:              https://www.espressif.com/
[event-processor-url]:  https://github.com/tailucas/event-processor
[flask-url]:            https://flask.palletsprojects.com/
[fontawesome-url]:      https://fontawesome.com/
[ha-url]:               https://www.home-assistant.io/
[healthchecks-url]:     https://healthchecks.io/
[influxdb-url]:         https://www.influxdata.com/
[jetson-url]:           https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/jetson-nano/
[mdash-url]:            https://mdash.net/home/
[mongoose-blog-url]:    https://tailucas.github.io/update/2023/06/07/iot-with-mongoose-os.html
[mqtt-url]:             https://mqtt.org/
[ngrok-python-url]:     https://github.com/ngrok/ngrok-python/tree/main
[ngrok-url]:            https://ngrok.com/
[oh-url]:               https://www.openhab.org/docs/
[poetry-url]:           https://python-poetry.org/
[pylib-url]:            https://github.com/tailucas/pylib
[python-url]:           https://www.python.org/
[rabbit-url]:           https://www.rabbitmq.com/
[rabbitmq-blog-url]:    https://tailucas.github.io/update/2023/06/30/message-brokers-rabbitmq.html
[rekognition-url]:      https://aws.amazon.com/rekognition/
[remote-monitor-url]:   https://github.com/tailucas/remote-monitor
[rpi-url]:              https://www.raspberrypi.org/
[sentry-url]:           https://sentry.io/for/python/
[snapshot-processor-url]: https://github.com/tailucas/snapshot-processor
[sqlalchemy-url]:       https://www.sqlalchemy.org/
[sqlite-url]:           https://www.sqlite.org/
[tailscale-url]:        https://tailscale.com/
[telegram-bot-url]:     https://docs.python-telegram-bot.org/en/stable/
[telegram-url]:         https://telegram.org/
[zmq-blog-url]:         https://tailucas.github.io/update/2023/06/25/message-brokers-zmq.html
[zmq-url]:              https://zeromq.org/
