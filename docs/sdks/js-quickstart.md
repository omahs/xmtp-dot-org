---
sidebar_label: JavaScript
sidebar_position: 1
toc_max_heading_level: 4
description: "The XMTP client SDK for JavaScript (xmtp-js) provides a TypeScript implementation of an XMTP message API client (client) for use with JavaScript and React applications."
---

# Quickstart for the JavaScript XMTP client SDK

![Status](https://img.shields.io/badge/Project_Status-Production-brightgreen)
![Test](https://github.com/xmtp/xmtp-js/actions/workflows/test.yml/badge.svg)
![Lint](https://github.com/xmtp/xmtp-js/actions/workflows/lint.yml/badge.svg)
![Build](https://github.com/xmtp/xmtp-js/actions/workflows/build.yml/badge.svg)

The [JavaScript XMTP client SDK](https://github.com/xmtp/xmtp-js) (`xmtp-js`) provides a TypeScript implementation of an XMTP message API client (client) for use with JavaScript and React applications.

Build with this SDK to provide messaging between blockchain wallet addresses, including DMs, notifications, announcements, and more.

`xmtp-js` was included in a [security assessment](https://xmtp.org/assets/files/REP-final-20230207T000355Z-3825cbc68c115f4ec81f3b1d53a24fce.pdf) prepared by [Certik](https://www.certik.com/company/about).

To learn more about XMTP and get answers to frequently asked questions, see [FAQ about XMTP](/docs/concepts/faq).

## Example apps built with `xmtp-js`

- **XMTP Quickstart React app**: Provides a basic messaging app demonstrating core capabilities of the SDK. The app is intentionally built with lightweight code to help make it easier to parse and start learning to build with XMTP.

  - [Try the app](https://xmtp-quickstart-react.vercel.app/)
  - [View the repo](https://github.com/xmtp/xmtp-quickstart-react)

- **XMTP Inbox app**: Provides a messaging app demonstrating core and advanced capabilities of the SDK. The app aims to showcase innovative ways of building with XMTP.
  - [Try the app](https://xmtp.chat/inbox)
  - [View the repo](https://github.com/xmtp-labs/xmtp-inbox-web)

## Reference docs

:::tip View the reference

Access the `xmtp-js` client SDK [reference documentation](https://xmtp-js.pages.dev/modules).

:::

## Install

```bash
npm install @xmtp/xmtp-js
```

Additional configuration is required in React environments due to the removal of polyfills from Webpack 5.

### Create React App

Use `react-scripts` prior to version `5.0.0`. For example:

```bash
npx create-react-app my-app --scripts-version 4.0.2
```

Or downgrade after creating your app.

### Next.js

In `next.config.js`:

```js
webpack: (config, { isServer }) => {
  if (!isServer) {
    config.resolve.fallback.fs = false;
  }
  return config;
};
```

## Usage

The [XMTP message API](/docs/concepts/architectural-overview#network-layer) revolves around a network client that allows retrieving and sending messages to other network participants. A client must be connected to a wallet on startup. If this is the very first time the client is created, the client will generate a [key bundle](/docs/concepts/key-generation-and-usage) that is used to [encrypt and authenticate messages](/docs/concepts/invitation-and-message-encryption). The key bundle persists encrypted in the network using a [wallet signature](/docs/concepts/account-signatures). The public side of the key bundle is also regularly advertised on the network to allow parties to establish shared encryption keys. All this happens transparently, without requiring any additional code.

```ts
import { Client } from "@xmtp/xmtp-js";
import { Wallet } from "ethers";

// You'll want to replace this with a wallet from your application
const wallet = Wallet.createRandom();
// Create the client with your wallet. This will connect to the XMTP development network by default
const xmtp = await Client.create(wallet);
// Start a conversation with XMTP
const conversation = await xmtp.conversations.newConversation(
  "0x3F11b27F323b62B159D2642964fa27C46C841897"
);
// Load all messages in the conversation
const messages = await conversation.messages();
// Send a message
await conversation.send("gm");
// Listen for new messages in the conversation
for await (const message of await conversation.streamMessages()) {
  console.log(`[${message.senderAddress}]: ${message.content}`);
}
```

Currently, network nodes are configured to rate limit high-volume publishing from clients. A rate-limited client can expect to receive a 429 status code response from a node. Rate limits can change at any time in the interest of maintaining network health.

### Conversations

Most of the time, when interacting with the network, you'll want to do it through `conversations`. Conversations are between two wallets.

```ts
import { Client } from "@xmtp/xmtp-js";
// Create the client with a `Signer` from your application
const xmtp = await Client.create(wallet);
const conversations = xmtp.conversations;
```

#### List existing conversations

You can get a list of all conversations that have one or more messages.

```ts
const allConversations = await xmtp.conversations.list();
// Say gm to everyone you've been chatting with
for (const conversation of allConversations) {
  console.log(`Saying GM to ${conversation.peerAddress}`);
  await conversation.send("gm");
}
```

These conversations include all conversations for a user **regardless of which app created the conversation.** This functionality provides the concept of an [interoperable inbox](/docs/concepts/interoperable-inbox), which enables a user to access all of their conversations in any app built with XMTP.

#### Listen for new conversations

You can also listen for new conversations being started in real-time. This will allow applications to display incoming messages from new contacts.

:::caution

This stream will continue infinitely. To end the stream, you can either break from the loop or call `await stream.return()`.

:::

```ts
const stream = await xmtp.conversations.stream();
for await (const conversation of stream) {
  console.log(`New conversation started with ${conversation.peerAddress}`);
  // Say hello to your new friend
  await conversation.send("Hi there!");
  // Break from the loop to stop listening
  break;
}
```

#### Start a new conversation

You can create a new conversation with any Ethereum address on the XMTP network.

```ts
const newConversation = await xmtp.conversations.newConversation(
  "0x3F11b27F323b62B159D2642964fa27C46C841897"
);
```

#### Send messages

To be able to send a message, the recipient must have already started their client at least once and consequently advertised their key bundle on the network. Messages are addressed using wallet addresses. The message payload can be a plain string, but other types of content can be supported through the use of `SendOptions`. To learn more, see [Handle different types of content](#handle-different-types-of-content).

```ts
const conversation = await xmtp.conversations.newConversation(
  "0x3F11b27F323b62B159D2642964fa27C46C841897"
);
await conversation.send("Hello world");
```

#### List messages in a conversation

You can receive the complete message history in a conversation by calling `conversation.messages()`.

```ts
for (const conversation of await xmtp.conversations.list()) {
  // All parameters are optional and can be omitted
  const opts = {
    // Only show messages from last 24 hours
    startTime: new Date(new Date().setDate(new Date().getDate() - 1)),
    endTime: new Date(),
  };
  const messagesInConversation = await conversation.messages(opts);
}
```

#### List messages in a conversation with pagination

It may be helpful to retrieve and process the messages in a conversation page by page. You can do this by calling `conversation.messagesPaginated()`, which will return an [AsyncGenerator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/AsyncGenerator) yielding one page of results at a time. `conversation.messages()` uses this under the hood internally to gather all messages.

```ts
const conversation = await xmtp.conversations.newConversation(
  "0x3F11b27F323b62B159D2642964fa27C46C841897"
);

for await (const page of conversation.messagesPaginated({ pageSize: 25 })) {
  for (const msg of page) {
    // Breaking from the outer loop will stop the client from requesting any further pages
    if (msg.content === "gm") {
      return;
    }
    console.log(msg.content);
  }
}
```

#### Listen for new messages in a conversation

You can listen for any new messages (incoming or outgoing) in a conversation by calling `conversation.streamMessages()`.

A successfully received message (that makes it through the decoding and decryption without throwing) can be trusted to be authentic, i.e. that it was sent by the owner of the `message.senderAddress` wallet and that it wasn't modified in transit. The `message.sent` timestamp can be trusted to have been set by the sender.

The Stream returned by the `stream` methods is an asynchronous iterator and as such usable by a for-await-of loop. Note however that it is by its nature infinite, so any looping construct used with it will not terminate, unless the termination is explicitly initiated (by breaking the loop or by an external call to `Stream.return()`).

```ts
const conversation = await xmtp.conversations.newConversation(
  "0x3F11b27F323b62B159D2642964fa27C46C841897"
);
for await (const message of await conversation.streamMessages()) {
  if (message.senderAddress === xmtp.address) {
    // This message was sent from me
    continue;
  }
  console.log(`New message from ${message.senderAddress}: ${message.content}`);
}
```

#### Listen for new messages in all conversations

To listen for any new messages from all conversations, use `conversations.streamAllMessages()`.

:::info

There is a chance this stream can miss messages if multiple new conversations are received in the time it takes to update the stream to include a new conversation.

:::

```ts
for await (const message of await xmtp.conversations.streamAllMessages()) {
  if (message.senderAddress === xmtp.address) {
    // This message was sent from me
    continue;
  }
  console.log(`New message from ${message.senderAddress}: ${message.content}`);
}
```

#### Handle different types of content

All send functions support `SendOptions` as an optional parameter. The `contentType` option allows specifying different types of content than the default simple string standard content type, which is identified with content type identifier `ContentTypeText`.

To learn more about content types, see [Content types with XMTP](/docs/concepts/content-types).

Support for other types of content can be added by registering additional `ContentCodecs` with the `Client`. Every codec is associated with a content type identifier, `ContentTypeId`, which is used to signal to the client which codec should be used to process the content that is being sent or received.

For example, see the [Codecs](https://github.com/xmtp/xmtp-js/tree/main/src/codecs) available in `xmtp-js`.

If there is a concern that the recipient may not be able to handle a non-standard content type, the sender can use the `contentFallback` option to provide a string that describes the content being sent. If the recipient fails to decode the original content, the fallback will replace it and can be used to inform the recipient what the original content was.

```ts
// Assuming we've loaded a fictional NumberCodec that can be used to encode numbers,
// and is identified with ContentTypeNumber, we can use it as follows.
xmtp.registerCodec:(new NumberCodec())
conversation.send(3.14, {
  contentType: ContentTypeNumber,
  contentFallback: 'sending you a pie'
})
```

Additional codecs can be configured through the `ClientOptions` parameter of `Client.create`. The `codecs` option is a list of codec instances that should be added to the default set of codecs (currently only the `TextCodec`). If a codec is added for a content type that is already in the default set, it will replace the original codec.

```ts
// Adding support for `xmtp.org/composite` content type
import { CompositeCodec } from "@xmtp/xmtp-js";
const xmtp = Client.create(wallet, { codecs: [new CompositeCodec()] });
```

To learn more about how to build a custom content type, see [Build a custom content type](/docs/concepts/content-types#build-a-custom-content-type).

Custom codecs and content types may be proposed as interoperable standards through XRCs. To learn about the custom content type proposal process, see [XIP-5](https://github.com/xmtp/XIPs/blob/main/XIPs/xip-5-message-content-types.md).

#### Compression

Message content can be optionally compressed using the `compression` option. The value of the option is the name of the compression algorithm to use. Currently supported are `gzip` and `deflate`. Compression is applied to the bytes produced by the content codec.

Content will be decompressed transparently on the receiving end. Note that `Client` enforces maximum content size. The default limit can be overridden through the `ClientOptions`. Consequently a message that would expand beyond that limit on the receiving end will fail to decode.

```ts
import { Compression } from "@xmtp/xmtp-js";

conversation.send("#".repeat(1000), {
  compression: Compression.COMPRESSION_DEFLATE,
});
```

#### Cache conversations

When running in a browser, conversations are cached in `LocalStorage` by default. Running `client.conversations.list()` will update that cache and persist the results to the browser's `LocalStorage`. The data stored in `LocalStorage` is encrypted and signed using the Keystore's identity key so that attackers cannot read the sensitive contents or tamper with them.

To disable this behavior, set the `persistConversations` client option to `false`.

```ts
const clientWithNoCache = await Client.create(wallet, {
  persistConversations: false,
});
```

## Breaking revisions

Because `xmtp-js` is in active development, you should expect breaking revisions that might require you to adopt the latest SDK release to enable your app to continue working as expected.

XMTP communicates about breaking revisions in the [XMTP Discord community](https://discord.gg/xmtp), providing as much advance notice as possible. Additionally, breaking revisions in an `xmtp-js` release are described on the [Releases page](https://github.com/xmtp/xmtp-js/releases).

### Deprecation schedule

Older versions of the SDK will eventually be deprecated, which means:

1. The network will not support and will eventually actively reject connections from clients using deprecated versions.
2. Bugs will not be fixed in deprecated versions.

The following table provides the deprecation schedule:

| Announced &nbsp; | Effective &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; | Minimum version | Rationale                                                                                                              |
| ---------------- | ------------------------------------------------------------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 2022-08-18       | 2022-11-08                                                                      | v6.0.0          | The XMTP network will stop supporting the Waku/libp2p-based client interface in favor of the new GRPC-based interface. |

Issues and PRs are welcome in accordance with XMTP [contribution guidelines](https://github.com/xmtp/xmtp-js/blob/main/CONTRIBUTING.md).

## XMTP `production` and `dev` network environments

XMTP provides both `production` and `dev` network environments to support the development phases of your project.

The `production` and `dev` networks are completely separate and not interchangeable.
For example, for a given blockchain account address, its XMTP identity on `dev` network is completely distinct from its XMTP identity on the `production` network, as are the messages associated with these identities. In addition, XMTP identities and messages created on the `dev` network can't be accessed from or moved to the `production` network, and vice versa.

:::important

When you [create a client](#create-a-client), it connects to the XMTP `dev` environment by default. To learn how to use the `env` parameter to set your client's network environment, see [Configure the client](#configure-the-client).

:::

The `env` parameter accepts one of three valid values: `dev`, `production`, or `local`. Here are some best practices for when to use each environment:

- `dev`: Use to have a client communicate with the `dev` network. As a best practice, set `env` to `dev` while developing and testing your app. Follow this best practice to isolate test messages to `dev` inboxes.

- `production`: Use to have a client communicate with the `production` network. As a best practice, set `env` to `production` when your app is serving real users. Follow this best practice to isolate messages between real-world users to `production` inboxes.

- `local`: Use to have a client communicate with an XMTP node you are running locally. For example, an XMTP node developer can set `env` to `local` to generate client traffic to test a node running locally.

The `production` network is configured to store messages indefinitely. XMTP may occasionally delete messages and keys from the `dev` network, and will provide advance notice in the [XMTP Discord community](https://discord.gg/xmtp).
