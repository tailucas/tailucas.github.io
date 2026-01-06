---
layout: post
title:  "Building with Message Brokers"
date:   2023-06-30 05:30:52 +0200
categories: update
---
Today I am going to talk about how and why I used message brokers in my [automation projects][home-automation-blog-url] along with a demonstration of the implementation in my [Python library][pylib-url] project.

:writing_hand: This post follows my [previous][zmq-blog-url] on message passing, which includes introduction of some of my common library code [pylib][pylib-url].

:incoming_envelope: While this project is an evaluation of my own experience with specific message brokers, I would recommend also reading more broadly on [the topic](https://duckduckgo.com/?va=v&t=ha&q=programming+with+message+brokers&ia=web) to get a better idea of best fit for your ideas.

### MQTT

If you are familiar with IoT, the [MQTT][mqtt-url] protocol needs little introduction. There is so much posted online that any introduction here would just be an inferior duplication of effort. In order to coordinate message transmission between publish and subscribe topics, you need to run a broker to which IoT devices connect. There are cloud-based brokers like [EMQX][emqx-url] (see also [MQTTX][mqttx-url]) but you can host your own with various levels of authentication and authorization. For my projects, I chose [Eclipse Mosquitto][mosquitto-url] which is also supports [docker container][mosquitto-docker-url] deployments and can be run with almost no configuration. For my application client, I chose [Eclipse Paho][paho-python-url] for Python.

Referring briefly [back][home-automation-blog-url] to my previous post, you can see how I build my MQTT client into my [ZeroMQ][zmq-url] pipeline:

{:refdef: style="text-align: center;"}
![Event Processor MQ](/assets/event-processor/event-processor_zmq_sockets.png)
{: refdef}

The `MqttSubscriber` MQTT client wrapper can be found [here](https://github.com/tailucas/event-processor/blob/bcca7e27c238cb783abf2102a339e2efcc11a7c8/app/__main__.py#L1471-L1615) on GitHub. Here is a summarized form:

{% highlight python %}
import paho.mqtt.client as mqtt
from paho.mqtt.client import MQTT_ERR_SUCCESS, MQTT_ERR_NO_CONN

from pylib.app import AppThread
from pylib.zmq import Closable
from pylib.handler import exception_handler


class MqttSubscriber(AppThread, Closable):

    def __init__(self):
        AppThread.__init__(self, name=self.__class__.__name__)
        Closable.__init__(self, connect_url='inproc://mqtt-publisher')
        # push socket to forward received MQTT messages to the event loop
        self.processor = self.get_socket(zmq.PUSH)
        # Paho client
        self._mqtt_client = None

    def close(self):
        Closable.close(self)
        try:
            self._mqtt_client.disconnect()
        except Exception:
            log.warning('Ignoring error closing MQTT socket.', exc_info=True)

    def on_connect(self, client, userdata, flags, rc):
        self._mqtt_client.subscribe(self._mqtt_subscribe_topics)

    def on_disconnect(self, client, userdata, rc):
        self._disconnected = True

    def on_message(self, client, userdata, msg):
        msg_data = None
        try:
            msg_data = json.loads(msg.payload)
        except JSONDecodeError:
            log.exception(f'Unstructured message: {msg.payload}')
            return
        # check assumptions on topic structure
        topic_base = '/'.join(msg.topic.split('/')[0:2])
        # unpack the message
        try:
            # ...
            # process msg_data
            # ...
            # forward message data to application event loop
            self.processor.send_pyobj({
                topic_base: {
                    'data': {
                        'device_info': {'inputs': device_inputs},
                        'active_devices': active_devices
                    },
                }
            })
        except ContextTerminated:
            self.close()

    # noinspection PyBroadException
    def run(self):
        # forward messages to the application event loop
        # via the device heartbeat nanny
        self.processor.connect('inproc://heartbeat-nanny')
        self._mqtt_client = mqtt.Client()
        self._mqtt_client.on_connect = self.on_connect
        self._mqtt_client.on_disconnect = self.on_disconnect
        self._mqtt_client.on_message = self.on_message
        self._mqtt_client.connect(self._mqtt_server_address)
        # Python context manager to support basic connection and exception handling
        with exception_handler(closable=self, and_raise=False, shutdown_on_error=True):
            while not threads.shutting_down:
                # blocking happens here
                rc = self._mqtt_client.loop()
                if rc == MQTT_ERR_NO_CONN or self._disconnected:
                    # this terminates the application but without sending exceptions to Sentry.io
                    raise ResourceWarning(f'No connection to MQTT broker at {self._mqtt_server_address} (disconnected? {self._disconnected})')
                # check for messages to publish back to MQTT
                try:
                    # non-blocking ZMQ read
                    mqtt_pub_topic, message_data = self.socket.recv_pyobj(flags=zmq.NOBLOCK)
                    self._mqtt_client.publish(topic=mqtt_pub_topic, payload=message_data)
                except ZMQError:
                    # ignore, no data
                    pass
{% endhighlight %}

### RabbitMQ

For communication between my automation applications I chose [RabbitMQ][rabbit-url] to explore another type of broker. The diagram below includes here [pylib][pylib-url] modules are used. This illustrates the use of a ZeroMQ Pipeline Pattern for inter-thread communication.

{:refdef: style="text-align: center;"}
![Overview](/assets/blog/messages/messages.png)
{: refdef}

For any of my applications, some instance of `ZMQListener` (found [here](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/rabbit.py#L140-L197)) is used to receive RabbitMQ messages from the network.

`ZMQListener` extends a class called `MQConnection` (found [here](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/rabbit.py#L41-L137)) in order to support RabbitMQ callbacks (i.e. receiving messages). By extending `AppThread` and `Closable`, `MQConnection` functions as an entirely self-sustaining application thread which contains the implementation to set up a RabbitMQ connection, publication of messages to the configured RabbitMQ exchange and topics, and connection teardown at application shutdown. 

From the diagram above, applications that host purely input or output functions also make use of `RabbitMQRelay` in the `pylib.rabbit` module (found [here](https://github.com/tailucas/pylib/blob/ac05d39592c2264143ec4a37fe76b7e0369515bd/pylib/rabbit.py#L200-L242)) for outbound communications. These are typically instantiated in the [main application thread](https://github.com/tailucas/snapshot-processor/blob/84394fbbcdb9402696720b1c6bf67586d77dcdd1/app/__main__.py#L896-L909) and require little more thought after this point. Here is an excerpt illustrating this and how they build up the ZeroMQ pipeline for inter-thread communication.

{% highlight python %}
from pika.exceptions import AMQPConnectionError, StreamLostError, ConnectionClosedByBroker

from pylib import app_config
from pylib.rabbit import ZMQListener, RabbitMQRelay
from pylib import threads
from pylib.threads import die, bye
from pylib.zmq import zmq_term


def main():
    mq_server_address=app_config.get('rabbitmq', 'server_address')
    mq_exchange_name=app_config.get('rabbitmq', 'mq_exchange')
    mq_device_topic=app_config.get('rabbitmq', 'device_topic')
    # receives RabbitMQ messages
    mq_control_listener = ZMQListener(
        zmq_url='inproc://app-thread',
        mq_server_address=mq_server_address,
        mq_exchange_name=f'{mq_exchange_name}_control',
        mq_topic_filter=f'event.control.{mq_device_topic}',
        mq_exchange_type='direct')
    # sends RabbitMQ messages for any ZeroMQ inter-thread messages sent to zmq_url
    try:
        mq_relay = RabbitMQRelay(
            zmq_url='inproc://rabbit-mq-publisher',
            mq_server_address=mq_server_address,
            mq_exchange_name=mq_exchange_name,
            mq_topic_filter=mq_device_topic,
            mq_exchange_type='topic')
    except AMQPConnectionError as e:
        die(exception=e)
        bye()
    # ...
    mq_control_listener.start()
    mq_relay.start()
    # ...
    try:
        # ...
        # start thread nanny
        nanny = threading.Thread(name='nanny', target=thread_nanny, args=(signal_handler,))
        nanny.setDaemon(True)
        nanny.start()
        # start heartbeat loop
        publisher_socket.connect('inproc://rabbit-mq-publisher')
        while not threads.shutting_down:
            heartbeat_payload = {
                'device_info': device_info
            }
            publisher_socket.send_pyobj((f'event.heartbeat.{mq_relay.device_topic}', heartbeat_payload))
            threads.interruptable_sleep.wait(HEARTBEAT_INTERVAL_SECONDS)
        raise RuntimeWarning()
    except(KeyboardInterrupt, RuntimeWarning, ContextTerminated) as e:
        die()
        mq_control_listener.stop()
        try:
            mq_relay.close()
        except (AMQPConnectionError, ConnectionClosedByBroker, StreamLostError) as e:
            log.warning(f'When closing: {e!s}')
    finally:
        zmq_term()
    bye()


if __name__ == "__main__":
    main()
{% endhighlight %}

My application [event loop](https://github.com/tailucas/event-processor/blob/bcca7e27c238cb783abf2102a339e2efcc11a7c8/app/__main__.py#L746-L757) also makes use of this pattern in order to send RabbitMQ messages to trigger actions. This is illustrated here with this code excerpt:

{% highlight python %}
class EventProcessor(MQConnection, Closable):

    def __init__(self, mq_server_address, mq_exchange_name):
        MQConnection.__init__(
            self,
            mq_server_address=mq_server_address,
            mq_exchange_name=mq_exchange_name,
            # direct routing
            mq_exchange_type='direct',
            # no control message should live longer than 90s
            mq_arguments={'x-message-ttl': 90*1000})
        Closable.__init__(self, connect_url=URL_WORKER_APP)
{% endhighlight %}

Publication to RabbitMQ happens [in this event loop](https://github.com/tailucas/event-processor/blob/bcca7e27c238cb783abf2102a339e2efcc11a7c8/app/__main__.py#L1301C32-L1301C32) by invoking `MQConnection._basic_publish`.

{% highlight python %}
    self._basic_publish(
        routing_key=f'event.control.{output_type}',
        event_payload=event_payload)
{% endhighlight %}

[emqx-url]:                 https://www.emqx.com/
[esp-url]:                  https://www.espressif.com/
[home-automation-blog-url]: https://tailucas.github.io/update/2023/06/18/home-automation.html
[mosquitto-docker-url]:     https://hub.docker.com/_/eclipse-mosquitto/
[mosquitto-url]:            https://mosquitto.org/
[mqtt-url]:                 https://mqtt.org/
[mqttx-url]:                https://mqttx.app/
[msg-pack-url]:             https://msgpack.org/
[paho-python-url]:          https://pypi.org/project/paho-mqtt/
[pylib-threads-url]:        https://github.com/tailucas/pylib/blob/master/pylib/threads.py
[pylib-url]:                https://github.com/tailucas/pylib
[pylib-zmq-url]:            https://github.com/tailucas/pylib/blob/master/pylib/zmq.py
[python-url]:               https://www.python.org/
[rabbit-gsg-url]:           https://www.rabbitmq.com/getstarted.html
[rabbit-url]:               https://www.rabbitmq.com/
[rpi-url]:                  https://www.raspberrypi.com/
[zmq-blog-url]:             https://tailucas.github.io/update/2023/06/25/message-brokers-zmq.html
[zmq-socket-api-url]:       https://zeromq.org/socket-api/
[zmq-url]:                  https://zeromq.org/
