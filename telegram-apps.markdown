---
layout: page
title: Telegram Applications
permalink: /telegram-apps/
youtubeId: hgE77lXTJjs
youtubeTitle: Programmable Banking Community Community Demos | 28 Sep 2023
youtubeWidth: 701
youtubeHeight: 394
youtubeStartSec: 9
---
This page lists the various applications I've built that make use of the [Telegram][telegram-url] messenger [Bot API][telegram-bot-api-url]. I've found this to be useful for any application that produces [temporal events][google-temporal-events] for human consumption that may require some asynchronous command-and-control by the operator. Telegram [bot commands][telegram-bot-cmds-url] provides a capable and "batteries-included" interface for this purpose.

This is *not* an endorsement for any [one messenger app over another][im-userbase-url], but rather my lived experience using Telegram as a human-computer interface (HCI).

## Spend Summaries on Investec Bank

When I heard about Investec Bank's [programmable banking][investec-url] community, I was interested to build a Telegram Bot that would show me various categorizations of my personal spending habits. To achieve this, I built [this project][investec-app-url] and took a small side-quest to create a [Python client][investec-client-url] given that the OpenAPI client generation logic did not work as expected for me at the time.

You can see it all in action here, in my community demo:

{% include youtube.html id=page.youtubeId title=page.youtubeTitle width=page.youtubeWidth height=page.youtubeHeight start=page.youtubeStartSec %}

## Basic Command-and-Control

One of my earliest [automation projects][event-processor-url] consisted of a monolithic Python application which had multiple responsibilities: A [Flask][flask-url] frontend for persisted configuration management in a simple [SQLite][sqlite-url] database, event processing logic (since ported to [Java][jevent-processor-url] while learning about [Java Virtual Threads][java-virtual-threads-url]). This project predates [Home Assistant][home-assistant-url] which would be a more sensible platform to build upon if starting again from scratch. Later in the project's lifecycle, I used Telegram as a convenient method to send automation events to myself and bot commands were a very useful addition as runtime modifiers of behaviour.

The logic that may be useful to others in this project is [here][event-processor-bot-url] which reconciled a few technical challenges:

1) Telegram client's move to Python [asyncio][python-asyncio-url] for more efficient asynchronous I/O management.

2) My choice to run the Telegram client in its own thread.

3) My [design choice](https://tailucas.github.io/update/mqtt/2023/06/18/home-automation.html) of using [ZeroMQ][zmq-url] to pass messages between application components.

## Pocket Lint (AKA pick from my Pocket)

> **📢 2025 Announcement:** Unfortunately, on 8 July 2025 Mozilla decided to [shutdown Pocket](https://getpocket.com/home) APIs, rendering [my project](https://github.com/tailucas/pocket-lint) defunct (on those APIs). However, there is a wealth of useful implementation in my project that would be applicable for other, non-trivial Telegram-enabled projects. Or perhaps even an interface to similar link-archival services since the majority of the project lends itself well to any similar backend.
{: .alert .alert-info}

---

Welcome to the Pocket Lint Telegram Bot documentation. Though most bot functions are self-explanatory, certain background behaviours are documented here for your convenience.
![Pocket][pocket-logo-url]{:height="64" align="right"}
{:style="clear: right"}

### Frequently Asked Questions

#### Why use Pocket Lint Telegram Bot?

Pocket has a web site and an mobile apps so another way to interact with Pocket seems redundant at first. I have been using Pocket for many years, but mostly to save thinks that I rarely come back to, if at all. I built this bot because I wanted to have a *single link* served to me by the bot and if I haven't tagged or archived the item, I wanted the opportunity to do this after actually reading the item. Since I already use Telegram, its built-in ability to perform link previews is also rather handy from a usability perspective; nothing new is needed to interact with my Pocket. If this presentation method works for you, then you should try this bot.

#### What information does the bot store about me?

At a minimum, the bot needs to store an authentication token that allows it to interact with your Pocket items. This token is generated as part of the `/start` command workflow and stored in a local database with the bot and is encrypted at rest using a key that is never persisted with the bot. Pick positions are stored and associated with your Telegram user ID. If tags are included, only cryptographic digests of the tags are stored with your pick positions meaning that there is no way to map the digest back to the actual tag. For more details about the implementation, see the [project documentation](https://github.com/tailucas/pocket-lint#readme-top). Application logs *do not* contain any user-content such as item links or any metadata fetched from Pocket, in the default logging mode. Logs include Telegram user ID and the action taken so that general activity is visible.

#### Can Pocket Lint serve me a random item from my Pocket?

The Application Programming Interface (API) provided by Pocket for item retrieval is constrained in that it will only allow retrieval for a limited number of items and an offset is needed to select which ones (if not the newest/oldest). There is no way to determine how many items there are in total without a brute-force method of fetching them all and then request rate limits become a factor at scale. So, the simplest design choice is to fetch a single item and store a "pick position" relative to the type of search (like unread, favourites, etc.). This pick position can be reset at any time with the `/settings` command.

#### Can the bot show me all my tags?

Unfortunately Pocket does not make this API available to 3rd party developer applications even though it is used internally by the Pocket web site.

#### Can the bot delete my items for me?

By design, no code exists in the bot to delete user content, even though modify-actions can be taken like archiving items. To delete your items, use the official Pocket app or web site.

#### Why do my commands fail with a permissions problem?

Either the bot is not yet registered with your account, or a problem in the bot means that a new authentication token is needed. It is always safe to use the `/start` command to generate a fresh authentication token.

#### How to I remove the bot's access to my Pocket?

1) Navigate to [getpocket.com/account](https://getpocket.com/account/)

2) Scroll to the *Third Party Applications* section.

3) Look for **Pocket Lint Telegram**

4) Click on the *Revoke Access* button.

The bot will now no longer have permissions to access your pocket. You can use the `/start` command to re-register the bot with your account.

#### Can the bot speak my language?

While Telegram allows the bot to know the user's language, the project currently supports only English feedback.

![Telegram][telegram-logo-url]{:width="32" align="center"}
[Take me back to the Bot][bot-url]

[bot-url]:                http://t.me/PocketLintBot
[event-processor-bot-url]: https://github.com/tailucas/event-processor/blob/e54f0d448264169b013cc5ed4ff766ef6240c327/app/__main__.py#L1826
[event-processor-url]:    https://github.com/tailucas/event-processor/tree/master
[flask-url]:              https://flask.palletsprojects.com/en/stable/
[google-temporal-events]: https://share.google/aimode/j1TmnojQK5QNcLY4D
[home-assistant-url]:     https://www.home-assistant.io/
[im-userbase-url]:        https://en.wikipedia.org/wiki/Instant_messaging#Current_user_base
[investec-app-url]:       https://github.com/tailucas/investec-my-charges
[investec-client-url]:    https://github.com/tailucas/investec-api-python
[investec-url]:           https://www.investec.com/en_za/banking/tech-professionals/programmable-banking.html
[java-virtual-threads-url]: https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html
[jevent-processor-url]:   https://github.com/tailucas/event-processor/blob/master/src/main/java/tailucas/app/EventProcessor.java
[pocket-lint-url]:        /assets/pocket-lint/Pocket_lint.JPG
[pocket-logo-url]:        https://upload.wikimedia.org/wikipedia/commons/thumb/2/2e/Pocket_App_Logo.png/320px-Pocket_App_Logo.png
[python-asyncio-url]:     https://docs.python.org/3/library/asyncio.html
[sqlite-url]:             https://sqlite.org/index.html
[telegram-bot-api-url]:   https://core.telegram.org/api#bot-api
[telegram-bot-cmds-url]:  https://core.telegram.org/api/bots/commands
[telegram-logo-url]:      https://upload.wikimedia.org/wikipedia/commons/thumb/8/82/Telegram_logo.svg/240px-Telegram_logo.svg.png
[telegram-url]:           https://telegram.org/
[zmq-url]:                https://zeromq.org/
