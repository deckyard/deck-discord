<!-- SPDX-License-Identifier: LGPL-3.0-only -->
<!-- SPDX-FileCopyrightText: Copyright (C) 2026 mplx <mplx@jennifer-lang.dev> -->

# Design

## Scope

The deck posts messages to a Discord channel through a channel webhook: plain
text with `send`, and rich messages (content, embeds with fields, footer and
author, per-message username and avatar) built with `message` and the builder
steps and posted with `sendMessage`. `render` returns the JSON payload without
sending it.

It is deliberately not a Discord client:

- **No bot API.** No gateway, no reading channels, no reactions, no slash
  commands. A webhook is the smallest thing that lets a program notify a
  channel, and it needs no bot token or OAuth application.
- **No attachments or file uploads.** A webhook can take multipart uploads;
  the module this deck came from never did, and no user has asked.
- **No editing or deleting** sent messages (that needs `?wait=true` and the
  message id).
- **No client-side enforcement of Discord's size limits** (content length,
  number of embeds and fields). Discord answers an oversized payload with a
  `400`, which reaches the caller as the response.

## Decisions

- **A deck, not a bundled module.** A client for one vendor's service belongs
  in the deck ecosystem, so Discord's API changes never touch the
  interpreter's release cycle. The deck is one of the bundled modules that
  `jennifer` moves out (M30.1 in its milestones, formerly M28.32).
- **Same surface as the module.** The types and functions are those of the
  standard library's `discord` module, so migrating changes only the import.
  Three additions differ: `noMentions` with the `Message` field
  `allowedMentions` behind it, the colour range check in `embed`, and the
  webhook URL check.
- **The response is returned, not judged.** `send` and `sendMessage` return
  the `http.Response`. Discord answers `204` on success; a deleted webhook
  (`404`), a malformed payload (`400`) or rate limiting (`429`) is a response
  the caller reads, not an error. A notifier usually logs a failure and
  carries on, and a status is easier to log than a caught error.
- **Immutable builder.** Every step takes a `Message` and returns a new one,
  so a partly built message can be reused as a template. The field, footer
  and author steps modify the latest embed.
- **Hand-rendered JSON.** `render` writes the payload itself, with every
  string passed through `json.encode`, so the key order is fixed and empty
  parts (no username, no avatar, no content, no embeds) are left out instead
  of being sent as empty strings, which Discord would reject or show.
- **Mentions are opt-out per message.** Discord acts on `@everyone`, `@here`,
  `<@id>` and `<@&role>` wherever the text came from. `noMentions` adds
  `"allowed_mentions": {"parse": []}`; it is a builder step rather than a
  default so that a message meant to ping someone keeps working as it did in
  the module. `send` has no such option; untrusted text goes through
  `sendMessage`.

## Errors

The deck throws `Error{kind: "discord"}` for its own checks only:

- a webhook URL that is not `https://`, or `http://` to a loopback host
  (thrown before anything is sent; the message leaves the URL out);
- an embed colour outside 0 to 16777215 (`0xFFFFFF`), which Discord refuses;
- `embedField`, `embedFooter` or `embedAuthor` on a message with no embed.

Discord's answers are responses, not errors (see above), so the deck has no
error struct (as recorded for the `@jennifer` decks in M28.57). Transport
failures come from `http` unchanged: `"timeout"`, `"closed"`, `"limit"`, and
on interpreters with transport error kinds (M28.65) a `net.NetError` of kind
`"net"` for a refused connection, reset, failed lookup or TLS failure (older
ones throw those as `"runtime"`).

## Security and limits

- **The webhook URL is the credential.** It carries the webhook's token, so
  anyone holding it can post to the channel. The docs tell callers to read it
  from the environment or a config file.
- **No cleartext sends.** `http` sends a plain `http://` URL to any host in
  the clear, so the deck refuses anything but `https://`. `http://` is allowed
  only to `127.0.0.1`, `localhost` and `[::1]`, for a local test server. The
  host is taken from the authority after any `user@` part, so
  `http://127.0.0.1@evil.example/` is refused.
- **Untrusted text** belongs in a message sent with `noMentions`; the deck
  escapes it as JSON, so it cannot change the payload's structure.
- Response size, deadlines and redirects are `http`'s defaults; the deck adds
  no caps of its own and no retries.

## Testing and validation

`src/discord_test.j` tests the rendered payloads directly (escaping, field,
footer and author order, identity overrides, mentions, colour range) and
sends through an `httpd` server on a loopback port that answers like a
webhook: `204` on the expected path, Discord's unknown-webhook `404`
otherwise. It also checks that every malformed or plain-`http://` URL throws
before a request goes out.

The tests never reach Discord. `examples/send.j` prints the payload it would
send and posts it when `DISCORD_WEBHOOK_URL` is set; without the variable it
posts nothing, so CI runs it offline.

## Known limitations and open questions

- Rate limits (`429` with `retry_after`) are left to the caller; the deck
  does not retry.
- Embed images, thumbnails, URLs and timestamps, and components, are not in
  the builder. They can be added when someone needs them.
- Threads and forum channels (`thread_id`, `thread_name`) are not supported.

## Dependencies on the interpreter

- The floor is `>=0.27.0`, for the UTF-8 codec of `encoding.decode` and
  `testing.assertThrowsArgs`. 0.26.0 deprecates the bundled module (M30.1)
  and its `http` refuses a URL without a valid scheme.
- `http` needs the `net` capability, so sending needs the default `jennifer`
  binary; building and rendering messages also work on `jennifer-tiny`.
- The tests use `httpd` and `task` for the loopback server.
