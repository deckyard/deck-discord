# @deckyard/discord

[![test](https://github.com/deckyard/deck-discord/actions/workflows/test.yml/badge.svg)](https://github.com/deckyard/deck-discord/actions/workflows/test.yml)
[![Documentation](https://img.shields.io/badge/docs-github%20pages-blue)](https://deckyard.github.io/deck-discord/)
[![release](https://img.shields.io/github/v/release/deckyard/deck-discord)](https://github.com/deckyard/deck-discord/releases/latest)

Posts messages to a [Discord™](https://discord.com) channel through a webhook
from a [Jennifer](https://jennifer-lang.dev/) program, from plain text to
embeds with fields, footer and author. Replaces the `discord` module of
Jennifer's standard library (shipped since 0.18.0, deprecated from 0.26.0,
removed in 0.28.0); the API is unchanged, only the import differs.

Up to 0.1.0 this deck was `@jennifer/discord`. To switch, change the `[decks]`
entry to `"@deckyard/discord" = "^0.2.0"` and the imports to `"@deckyard/discord/"`.

**[Documentation](https://deckyard.github.io/deck-discord/)**

## Install

```sh
jvc add @deckyard/discord
```

This vendors the deck to `vendor/deckyard/discord/`. Without `jvc`, unpack the
`src/` subtree of a release archive (`deckyard-discord-<version>.tar.gz`) there.

## Usage

```jennifer
import "@deckyard/discord/" as discord;
import "http.j" as http;

def r as http.Response init discord.send(os.getEnv("DISCORD_WEBHOOK_URL"), "deploy finished");
io.printf("%d\n", $r.status);   # 204; a deleted webhook is 404, not an error
```

`examples/send.j` builds a message with an embed, prints its payload and
posts it when `DISCORD_WEBHOOK_URL` is set.

## Requirements

Jennifer 0.27.0 or newer and network access to Discord. Sending needs the
default `jennifer` binary; the stock `jennifer-tiny` has no network driver.

## Development

```sh
tools/check-all.sh                   # the full gate, as CI runs it
jennifer test src/discord_test.j     # the test overlay
tools/check-docs.sh                  # docs against the code
tools/pack.sh dist                   # build the release archive
```

## Licence

LGPL-3.0-only, see [LICENSE](LICENSE). Copyright (C) 2026 mplx &lt;mplx@jennifer-lang.dev&gt;.

`tools/check-headers.sh` checks every source file's SPDX header against
`deck.toml`.

## Trademarks

Discord™ is a trademark of Discord Inc. Other product and company names may
be trademarks of their respective owners. They are used only to identify the
products this deck works with. This deck is an independent project, not
affiliated with, sponsored or endorsed by Discord Inc.
