---
layout: post
title:  "Electricity Inverter Monitoring"
date:   2023-06-04 08:30:52 +0200
categories: update
---
This post is a companion to my [inverter monitoring][inverter-monitor-url] project.

Electricity inverter products typically come with a mobile app to visualize your production and consumption data. I wanted to be able to retrieve this telemetry directly from my [inverter](https://www.deyeinverter.com/) which is shipped with a separate [stick logger](https://www.deyeinverter.com/product/accessory-monitoring-1/stick-logger.html). When configured for the local WiFi network, the logger is responsible for sending telemetry to a proprietary web service. If the stick logger is not supplied, there are options to interface directly with the inverter hardware like [this product](https://solar-assistant.io/explore/deye#hero). Since my inverter came with the logger device and because I do occasionally use the mobile app, I wanted to be able to tap additional data off the logger on my local network to get more fields than I am able to see on either the app or the proprietary web interface.

Having found a [binary protocol translator][deye-project-url] for my stick logger, I went to work building an application around this for the purposes of [visualizing the data][influxdata-url]. I also wanted to send some MQTT messages for control of switches for which I created a [companion project][switch-app-url] to control some [ESP32][esp-url] devices. I wanted to be able to create a crude measure of solar production capability by fetching cloud cover data for my location, and in addition to this I created a synthetic metric that indicates the proximity to noon for the given time of the year. I haven't found this particularly indicative of solar production capacity however.

The template for this dashboard is located [here](/assets/blog/inverter/influxdb_dashboard_sample.json).

![Dashboard Left](/assets/blog/inverter/inverter_dashboard_a.png)

![Dashboard Right](/assets/blog/inverter/inverter_dashboard_b.png)

[inverter-monitor-url]: https://github.com/tailucas/inverter-monitor
[deye-project-url]: https://github.com/jlopez77/DeyeInverter
[influxdata-url]: https://www.influxdata.com/
[switch-app-url]: https://github.com/tailucas/switch-app
[esp-url]: https://www.espressif.com/en/products/socs/esp32