---
layout: post
title:  "IoT with Mongoose OS"
date:   2023-06-07 21:30:52 +0200
categories: update
---
While [IoT](https://en.wikipedia.org/wiki/Internet_of_things) needs little introduction, the choice of platform for your own IoT project carries far-reaching implications for how you spend your energy deploying more than a single instance of your project to the real-world. This is the first of two posts about the types of IoT frameworks that I have used that represent fairly different approaches. This is my experience with [Mongoose OS][mongoose-url].

I was put onto the idea of Mongoose OS after enjoying some fun possibilities with the [AWS IoT Button](https://aws.amazon.com/iotbutton/). Since general availability of the IoT button was not great, I naturally wanted to compose something with more readily available hardware. I stumbled upon mention of the [ESP32][esp-url] running Mongoose OS featured in the [AWS Partner Device Catalog](https://devices.amazonaws.com/detail/a3G0L00000AANshUAH/ESP32-PICO-KIT-V4-with-Mongoose-OS) and so I thought I'd go learn more. While Mongoose OS enables rather trivial registration of your IoT device with major cloud services, I noticed that Cesanta had its own [IoT Cloud][mdash-url] product which was really all I needed.

As a brief aside, anecdotally, I found my Mongoose-enabled devices would be much more reliable staying connected to WiFi than when building an application using the ESP libraries included with the Arduino-IDE development environment. This likely boils down to more sophisticated handling of connectivity done out the box by the Mongoose firmware.

With Mongoose OS, you can choose to integrate with their API in either [C](https://mongoose-os.com/docs/mongoose-os/quickstart/develop-in-c.md)-style language or in [mJS](https://mongoose-os.com/docs/mongoose-os/quickstart/develop-in-js.md).

By [installing][mos-install-url] the `mos` [tool][mos-tool-url], you can quickly register your IoT device, get WiFi configured and then build and flash your project firmware to your device in minutes. Using [mDash][mdash-url], you can register a few devices in their free-tier to interact with the [device shadow](https://mongoose-os.com/docs/mdash/shadow.md). Here you can get details about the device's network connectivity, and interact with RPC methods to further configure and manage the device, including OTA updates which can be separately staged and committed. This is useful when testing some new functionality where a panic-reboot would restore the previously committed firmware. All these are fairly standard device management features in IoT.

My projects make use of an [ESP32][esp-url] on a [NodeMCU dev-kit board](https://www.nodemcu.com/index_en.html), a common form-factor for use by hobbyists. Mongoose projects are rather straight-forward but have a specific layout, which you can find by following their [quick start guide][mongoose-gsg-url]. The `mos.yml` file contains application specific configuration variables and also library definitions to include as part of the firmware build. It is useful to know that the variables in the YAML configuration are also made available to the device shadow.  The API documentation will indicate which libraries are needed for the desired capabilities.

Using mJS as an example, the `fs/init.js` script contains your application code which can then access the deployed values using `Cfg.get`. Here are examples of my Mongoose OS projects:

* [Magnetic Contact Sensor][adc-app-url]
* [Meter Counter with programmable register][meter-app-url]
* [Almost entirely uninteresting GPIO channel control for electronic relays using MQTT][switch-app-url]

In each of these projects, you'll notice heartbeats sent using [MQTT][mqtt-url] messages. This pairs well with visibility tools like [Uptime Kuma][uptime-kuma-url] which I will talk about in a future post.

I use [mDash][mdash-url] not only for occasional device management on their dashboard, but I also use their `REST` API for [device discovery][mdash-api-url]. I have an example of this in [one of my larger projects][event-processor-mdash-url] which I will open-source and discuss in a future post.

In the next post, I will discuss my experience with the [Balena Cloud](https://www.balena.io/cloud) IoT platform.

[adc-app-url]: https://github.com/tailucas/adc-app
[meter-app-url]: https://github.com/tailucas/meter-app
[switch-app-url]: https://github.com/tailucas/switch-app

[uptime-kuma-url]: https://uptime.kuma.pet/
[esp-url]: https://www.espressif.com/en/products/socs/esp32
[event-processor-mdash-url]: https://github.com/tailucas/event-processor/blob/fa94393b835efa5de312ec182ba7dbae73bd60a3/app/__main__.py#L1783-L1849
[mdash-url]: https://mdash.net/home/
[mdash-api-url]: https://mdash.net/docs/#management-rest-api
[mongoose-url]: https://mongoose-os.com/
[mongoose-gsg-url]: https://mongoose-os.com/docs/mongoose-os/quickstart/setup.md#4-create-new-app
[mos-tool-url]: https://mongoose-os.com/docs/mongoose-os/userguide/mos-tool.md
[mos-install-url]: https://mongoose-os.com/docs/mongoose-os/quickstart/setup.md
[mqtt-url]: https://mqtt.org/
