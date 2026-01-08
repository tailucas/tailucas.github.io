---
layout: page
title: Ping Tool
permalink: /ping-tool/
---
[This project](https://github.com/tailucas/ping-tool) belongs to my open-source [collection][tailucas-url], borrowing elements from the more sophisticated [base project][baseapp-url] from which I have created a [reference implementation][simple-app-url] to generally bootstrap new projects for myself. This project has been deliberately simplified to run a Python application natively on the still-relevant [Raspberry Pi][pi-url] single-board computer (SBC). With the few remaining Model B+ devices that I have, I run a [Python][python-url] application on the Debian-based [Raspbery Pi OS][pi-os-url] with successful Python dependency compilation on ARMv6. Although this project is configured for development using VSCode [development containers][dev-container-url], I chose to install and run the application natively without running a built container due to the constrained environment of the Model B+ SBC.

![classes](/assets/ping-tool/ping-tool.drawio.png)

You can read more about the problem solved by this project in my post [here](https://tailucas.github.io/update/2024/11/10/network-monitoring-with-tailscale.html).

[baseapp-url]: https://github.com/tailucas/base-app
[dev-container-url]: https://code.visualstudio.com/docs/devcontainers/containers
[pi-os-url]: https://www.raspberrypi.com/software/
[pi-url]: https://en.wikipedia.org/wiki/Raspberry_Pi
[python-url]: https://www.python.org/
[simple-app-url]: https://github.com/tailucas/simple-app
[tailucas-url]: https://github.com/tailucas
