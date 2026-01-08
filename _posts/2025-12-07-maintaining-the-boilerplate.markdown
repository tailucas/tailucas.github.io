---
layout: post
title:  "Maintaining the Boilerplate"
date:   2025-12-07 06:30:52 +0200
categories: update
---
Today I'm going to talk about my [code boilerplate](https://share.google/aimode/SFNxRaAG9MDgteB6x), created and maintained across the lifespan of my projects. Given that my projects are open-sourced under the MIT License, I hope that these serve as a useful framework for others to use. While I am of course opinionated about the specific package dependencies, these can be swapped out as needed. I also welcome any specific feedback about taking a better approach.

## So what is it?

From the moment that I needed to create another distinct project with the same scaffold as before, I created these:

1) my top-level boilerplate package [base-app][base-app-url] whose multi-architecture images are maintained by me in the [GitHub Container Registry](https://github.com/tailucas/base-app/pkgs/container/base-app) and [Docker Hub](https://hub.docker.com/repository/docker/tailucas/base-app/tags). Until very recently I was building for only `x86`, but I'm slowly preparing for the possibility of building my base container for both `x86` and `ARM` for future compute flexibility.

2) my common Python library package [pylib][pylib-url].

3) my companion "reference implementation" [simple-app][simple-app-url] which serves as the template for any new application I want to create based on the boilerplate I've created.

## base-app

This Docker application was created by factoring out many reusable code artifacts from my [various projects][tailucas-url] over a number of years. Since this work was not a part of a group effort, the test coverage is predictably abysmal :raised_eyebrow: and Python documentation notably absent :expressionless:. This package takes a dependency on another one of my [common packages][pylib-url]. While this application is almost entirely boilerplate, it can run as a stand-alone application and serves the basis for any well-behaved Python application. The design is opinionated with the use of [ZeroMQ][zmq-url] but this is not a strict requirement. This project has a required dependency on [1Password][1p-url] for both build and run time (explained later). If this is unacceptable, you'll need to fork this project, send a pull request with an appropriate shim for a substitute like Bitwarden or equivalent.

Enough talk! What do I get?

* A Docker application that is specifically designed to act as a base image for derived applications, or can be forked to run as is (mostly for testing purposes). This includes a variety of default setup and entrypoint scripts that can be optionally overridden in derived containers.
* Powerful threading and inter-thread data handling functions with significant resilience to unchecked thread death. This relies heavily on functions provided by my [pylib package][pylib-url] published also to the Python Package Index (PyPi).
* Sample [Healthchecks][healthchecks-url] cron job with built-in container setup (this was much less straight-forward and well documented than I expected).
* Pre-configured process control using [supervisor](http://supervisord.org/).
* Automatic syslog configuration to log to the Docker host rsyslog.
* Support for AWS-CLI if appropriate AWS environment variables are present, like `AWS_DEFAULT_REGION`.
* Python dependency management using [uv][uv-url].
* A sample Python application.
* A sample Java application.
* A sample Rust application.

### Notable Changes
Over time, I've made some specific, non-trivial changes to the project:

* Switched from [Poetry][poetry-url] to [uv][uv-url] for python package dependency management.
* Use [SDKMAN!][sdkman-url] for Java environment management and runtime selection.
* Switched to Ubuntu as the base image, previously using Alpine for size, but I found that older SBC libraries would not consistently have required binary dependencies for that platform. This aligns with my personal development environment and is usually well supported by the Linux tooling ecosystem. My default preference would be Debian.
* In the Python sample application, more embracing of `asyncio` by default.

General simplification of the environment setup:
* Removed magic constants in Python `__init__.py`; it's now blank.

### Lessons Learned
Over the time I've maintained this boilerplate, I've learned a few things:

#### Delegate undifferentiated work to tools

While it's tempting to stitch together project components using shell scripting, there's often a project that does it better in a more versatile manner. A good example of this [SDKMAN!][sdkman-url] for Java environment setup.

#### Establish separation of concerns early

Adding [dev container][dev-container-url] support for my project was a major productivity win, and the fact that this is supported by VSCode makes this a "no-brainer". This enabled a much closer parity between my development and runtime environments. As the project evolved, I still faced a leaky abstraction between the setup of the development environment and the final project runtime, particularly at the interaction points of Docker builds. This became clear when I added support for container publication via GitHub actions.

When I added [Taskfile][taskfile-url] support to my project, I inadvertently had build steps in my Taskfile definition that should have actually been better contained in the closure of the `docker-compose.yml` and `Dockerfile`. My more recent project refactors make a more clear separation: Taskfile is responsible for simply driving build actions and everything to do with the actual runtime environment is entirely self-contained in the Docker builder ecosystem.

## pylib
Python-specific boilerplate lives in my (creatively named) [pylib][pylib-url] and published to PyPi as [tailucas-pylib][pylib-pypi-url].

### Notable Changes
This has had a number of changes over time as I've extended functionality in my derived projects.

* `__init__.py`: Much of the early application bootstrap happens here and is now more tolerant to cases where certain things are not set up. For example when importing this as part of short-lived tool invocations.
* `aws/__init__.py`: generally better boto client session setup in using role assumption.
* `creds`: A [fully abstracted 1Password client][1p-secrets-automation] supporting *both* Connect Server and Service Account modes depending on configuration, that properly loads client keys using [Docker secrets][docker-secrets-url]. This removes all 1Password-specific logic from client applications.
* `data`: More consistent timestamp handling and creation.
* `datetime`: Cleaned-up timestamp generation and parsing logic.
* `device`: Introduction of a generic "Device" [Pydantic][pydantic-url] `BaseModel`.
* `flags`: Better abstraction of feature flag configuration.
* `threads`: Improved application fatal exception handling and client library shutdown logic.

### Lessons Learned
The most obvious lesson here is test coverage. While it's tempting to take shortcuts on personal projects, I've found investment in test coverage for reusable code to be a time saver in the long run. It takes time to roll the changes out to each project and broken shared code puts you back at square-one.

## simple-app
This serves as a [reference implementation][simple-app-url] project that derives from the [base application][base-app-url]. As far as possible, I keep this project up to date so that if I need a new application project later, I can hard-fork this to create a new one.

[1p-secrets-automation]: https://developer.1password.com/docs/secrets-automation/
[1p-url]:               https://developer.1password.com/docs/connect/
[aws-url]:              https://aws.amazon.com/
[base-app-url]:         https://github.com/tailucas/base-app
[botoflow-url]:         https://github.com/boto/botoflow
[cronitor-url]:         https://cronitor.io/
[dev-container-url]:    https://containers.dev/implementors/features/
[docker-msb-url]:       https://docs.docker.com/build/building/multi-stage/
[docker-secrets-url]:   https://docs.docker.com/engine/swarm/secrets/
[healthchecks-url]:     https://healthchecks.io/
[msgpack-url]:          https://msgpack.org/
[poetry-url]:           https://python-poetry.org/
[pydantic-url]:         https://pydantic.dev/
[pylib-pypi-url]:       https://pypi.org/project/tailucas-pylib/
[pylib-url]:            https://github.com/tailucas/pylib
[python-url]:           https://www.python.org/
[rabbit-url]:           https://www.rabbitmq.com/
[sdkman-url]:           https://sdkman.io/
[sentry-url]:           https://sentry.io/
[simple-app-url]:       https://github.com/tailucas/simple-app
[tailucas-url]:         https://github.com/tailucas
[taskfile-url]:         https://taskfile.dev/
[uv-url]:               https://docs.astral.sh/uv/
[zmq-url]:              https://zeromq.org/