---
sidebar_label: Dart
sidebar_position: 3
toc_max_heading_level: 4
description: 'xmtp-flutter provides a Dart implementation of an XMTP message API client for use with Flutter and mobile apps.'
---

# Quickstart for the Dart XMTP client SDK

![Status](https://img.shields.io/badge/Project_Status-Production-31CA54) ![Test](https://github.com/xmtp/xmtp-flutter/actions/workflows/test.yml/badge.svg)

The [Dart XMTP client SDK](https://github.com/xmtp/xmtp-flutter) (`xmtp-flutter`) provides a Dart implementation of an XMTP message API client for use with Flutter and mobile apps.

Build with this SDK to provide messaging between blockchain wallet addresses, including DMs, notifications, announcements, and more.

To keep up with the latest SDK developments, see the [Issues tab](https://github.com/xmtp/xmtp-flutter/issues) in the `xmtp-flutter` repo.

To learn more about XMTP and get answers to frequently asked questions, see [FAQ about XMTP](/docs/concepts/faq).

## Example app

For a basic demonstration of the core concepts and capabilities of the `xmtp-flutter` client SDK, see the [Example app project](https://github.com/xmtp/xmtp-flutter/tree/main/example).

## Reference docs

:::tip View the reference

Access the [Dart client SDK reference documentation](https://pub.dev/documentation/xmtp/latest/xmtp/xmtp-library.html) on pub.dev.

:::

## Install with Dart Package Manager

```bash
flutter pub add xmtp
```

To see more options, check out the [verified XMTP Dart package](https://pub.dev/packages/xmtp/install).

## Usage overview

The XMTP message API revolves around a message API client (client) that allows retrieving and sending messages to other XMTP network participants. A client must connect to a wallet app on startup. If this is the very first time the client is created, the client will generate a key bundle that is used to encrypt and authenticate messages. The key bundle persists encrypted in the network using an account signature. The public side of the key bundle is also regularly advertised on the network to allow parties to establish shared encryption keys. All of this happens transparently, without requiring any additional code.

```dart
import 'package:xmtp/xmtp.dart' as xmtp;
import 'package:web3dart/credentials.dart';
import 'dart:math';

var wallet = EthPrivateKey.createRandom(Random.secure());
var api = xmtp.Api.create();
var client = await xmtp.Client.createFromWallet(api, wallet);
```

### List existing conversations

You can list existing conversations and send them messages.

```dart
var conversations = await client.listConversations();
for (var convo in conversations) {
  debugPrint('Saying GM to ${convo.peer}');
  await client.sendMessage(convo, 'gm');
}
```

These conversations include all conversations for a user **regardless of which app created the conversation.** This functionality provides the concept of an interoperable inbox, which enables a user to access all of their conversations in any app built with XMTP.


### Listen for new conversations

You can also listen for new conversations being started in real-time.
This will allow apps to display incoming messages from new contacts.

```dart
var listening = client.streamConversations().listen((convo) {
  debugPrint('Got a new conversation with ${convo.peer}');
});
// When you want to stop listening:
await listening.cancel();
```

### Start a new conversation

You can create a new conversation with any Ethereum address on the XMTP network.

```dart
var convo = await client.newConversation("0x...");
```

### Send messages

To be able to send a message, the recipient must have already created a client at least once and
consequently advertised their key bundle on the network.

Messages are addressed using account addresses.

The message content can be a plain text string. Or you can configure custom content types.
See [Handle different types of content](#handle-different-types-of-content).

```dart
var convo = await client.newConversation("0x...");
await client.sendMessage(convo, 'gm');
```

### List messages in a conversation

You can receive the complete message history in a conversation.

```dart
// Only show messages from the last 24 hours.
var messages = await alice.listMessages(convo,
    start: DateTime.now().subtract(const Duration(hours: 24)));
```

### List messages in a conversation with pagination

It may be helpful to retrieve and process the messages in a conversation page by page.
You can do this by specifying `limit` and `end`, which will return the specified number
of messages sent before that time.

```dart
var messages = await alice.listMessages(convo, limit: 10);
var nextPage = await alice.listMessages(
    convo, limit: 10, end: messages.last.sentAt);
```

### Listen for new messages in a conversation

You can listen for any new messages (incoming or outgoing) in a conversation by calling
`client.streamMessages(convo)`.

A successfully received message (that makes it through the decoding and decryption) can be trusted
to be authentic. Authentic means that it was sent by the owner of the `message.sender` account and
that it wasn't modified in transit. The `message.sentAt` time can be trusted to have been set by
the sender.

```dart
var listening = client.streamMessages(convo).listen((message) {
  debugPrint('${message.sender}> ${message.content}');
});
// When you want to stop listening:
await listening.cancel();
```

:::note

This package does not currently include the `streamAllMessages()` functionality from the [XMTP client SDK for JavaScript](https://github.com/xmtp/xmtp-js) (xmtp-js).

:::

## Handle different types of content

When sending a message, you can specify the type of content. This allows you to specify different types of content than the default (a simple string, `ContentTypeText`).

To learn more about content types, see [Content types with XMTP](/docs/concepts/content-types).

Support for other types of content can be added during client construction by registering additional `Codec`s, including a `customCodecs` parameter. Every codec declares a specific content type identifier, `ContentTypeId`, which is used to signal to the client which codec should be used to process the content that is being sent or received.

```dart
/// Example [Codec] for sending [int] values around.
final contentTypeInteger = xmtp.ContentTypeId(
  authorityId: "com.example",
  typeId: "integer",
  versionMajor: 0,
  versionMinor: 1,
);
class IntegerCodec extends Codec<int> {
  @override
  xmtp.ContentTypeId get contentType => contentTypeInteger;

  @override
  Future<int> decode(xmtp.EncodedContent encoded) async =>
      Uint8List.fromList(encoded.content).buffer.asByteData().getInt64(0);

  @override
  Future<xmtp.EncodedContent> encode(int decoded) async => xmtp.EncodedContent(
    type: contentType,
    content: Uint8List(8)..buffer.asByteData().setInt64(0, decoded),
    fallback: decoded.toString(),
  );
}

// Using the custom codec to send around an integer.
var client = await Client.createFromWallet(api, wallet, customCodecs:[IntegerCodec()]);
var convo = await client.newConversation("0x...");
await client.sendMessage(convo, "Hey here comes my favorite number:");
await client.sendMessage(convo, 42, contentType: contentTypeInteger);
```

If there is a concern that the recipient may not be able to handle a non-standard content type, the sender can use the `contentFallback` option to provide a string that describes the content being sent. If the recipient fails to decode the original content, the fallback will replace it and can be used to inform the recipient what the original content was.

Codecs and content types may be proposed as interoperable standards through XRCs. To learn about the custom content type proposal process, see [XIP-5](https://github.com/xmtp/XIPs/blob/main/XIPs/xip-5-message-content-types.md).

## Compression

This package currently does not support message content compression.

## Breaking revisions

Because `xmtp-flutter` is in active development, you should expect breaking revisions that might require you to adopt the latest SDK release to enable your app to continue working as expected.

XMTP communicates about breaking revisions in the [XMTP Discord community](https://discord.gg/xmtp), providing as much advance notice as possible. Additionally, breaking revisions in an `xmtp-flutter` release will be described on the [Releases page](https://github.com/xmtp/xmtp-flutter/releases).

## Deprecation

Older versions of the SDK will eventually be deprecated, which means:

1. The network will not support and eventually actively reject connections from clients using deprecated versions.
2. Bugs will not be fixed in deprecated versions.

The following table provides the deprecation schedule.

| Announced                                                            | Effective | Minimum Version | Rationale |
| -------------------------------------------------------------------- | --------- | --------------- | --------- |
| There are no deprecations scheduled for `xmtp-flutter` at this time. |           |                 |           |

Bug reports, feature requests, and PRs are welcome in accordance with these [contribution guidelines](https://github.com/xmtp/xmtp-flutter/blob/main/CONTRIBUTING.md).
