---
layout: page
title: Pocket Lint
permalink: /pocket-lint/
---
Welcome to the Pocket Lint Telegram Bot documentation. Though most bot functions are self-explanatory, certain background behaviours are documented here for your convenience.
![Pocket][pocket-logo-url]{:height="64" align="right"}
{:style="clear: right"}
![Pocket Lint][pocket-lint-url]{:width="256" align="right"}
{:style="clear: right"}

### Frequently Asked Questions

#### What information does the bot store about me?

At a minimum, the bot needs to store an authentication token that allows it to interact with your Pocket items. This token is generated as part of the `/start` command workflow and stored in a local database with the bot and is encrypted at rest using a key that is never persisted with the bot. Pick positions are stored and associated with your Telegram user ID. If tags are included, only cryptographic digests of the tags are stored with your pick positions. For more details about the implementation, see the [project documentation](https://github.com/tailucas/pocket-lint#readme-top). Application logs *do not* contain any user-content such as item links or any metadata fetched from Pocket, in the default logging mode. Logs include Telegram user ID and the action taken so that general activity is visible.

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

[bot-url]: http://t.me/PocketLintBot
[pocket-logo-url]: https://upload.wikimedia.org/wikipedia/commons/thumb/2/2e/Pocket_App_Logo.png/320px-Pocket_App_Logo.png
[pocket-lint-url]: /assets/pocket-lint/Pocket_lint.JPG
[telegram-logo-url]: https://upload.wikimedia.org/wikipedia/commons/thumb/8/82/Telegram_logo.svg/240px-Telegram_logo.svg.png