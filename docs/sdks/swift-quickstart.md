---
sidebar_label: Swift
sidebar_position: 2
toc_max_heading_level: 4
description: 'xmtp-ios provides a Swift implementation of an XMTP message API client for use with iOS apps.'
---

# Quickstart for the Swift XMTP client SDK

![Status](https://img.shields.io/badge/Project_Status-Production-31CA54) ![Lint](https://github.com/xmtp/xmtp-ios/actions/workflows/lint.yml/badge.svg) 

The [Swift XMTP client SDK](https://github.com/xmtp/xmtp-ios) (`xmtp-ios`) provides a Swift implementation of an XMTP message API client for use with iOS apps.

Build with this SDK to provide messaging between blockchain wallet addresses, including DMs, notifications, announcements, and more.

To keep up with the latest SDK developments, see the [Issues tab](https://github.com/xmtp/xmtp-ios/issues) in the `xmtp-ios` repo.

To learn more about XMTP and get answers to frequently asked questions, see [FAQ about XMTP](/docs/concepts/faq).

## Reference docs

:::tip View the reference

Access the [Swift client SDK reference documentation](https://xmtp.github.io/xmtp-ios/documentation/xmtp/).

:::

## Example app

For a basic demonstration of the core concepts and capabilities of the `xmtp-ios` client SDK, see the [Example app project](https://github.com/xmtp/xmtp-ios/tree/main/XMTPiOSExample/XMTPiOSExample).

## Install with Swift Package Manager

Use Xcode to add to the project (**File** > **Add Packages…**) or add this to your `Package.swift` file:

```swift
.package(url: "https://github.com/xmtp/xmtp-ios", branch: "main")
```

## Usage overview

The XMTP message API revolves around a message API client (client) that allows retrieving and sending messages to other XMTP network participants. A client must connect to a wallet app on startup. If this is the very first time the client is created, the client will generate a key bundle that is used to encrypt and authenticate messages. The key bundle persists encrypted in the network using an account signature. The public side of the key bundle is also regularly advertised on the network to allow parties to establish shared encryption keys. All of this happens transparently, without requiring any additional code.

```swift
import XMTP

// You'll want to replace this with a wallet from your application.
let account = try PrivateKey.generate()

// Create the client with your wallet. This will connect to the XMTP `dev` network by default.
// The account is anything that conforms to the `XMTP.SigningKey` protocol.
let client = try await Client.create(account: account)

// Start a conversation with XMTP
let conversation = try await client.conversations.newConversation(with: "0x3F11b27F323b62B159D2642964fa27C46C841897")

// Load all messages in the conversation
let messages = try await conversation.messages()
// Send a message
try await conversation.send(content: "gm")
// Listen for new messages in the conversation
for try await message in conversation.streamMessages() {
  print("\(message.senderAddress): \(message.body)")
}
```

## Configure content types

You can use custom content types by calling `Client.register`. The SDK comes with two commonly used content type codecs, `AttachmentCodec` and `RemoteAttachmentCodec`:

```swift
Client.register(AttachmentCodec())
Client.register(RemoteAttachmentCodec())
```

To learn more about using `AttachmentCodec` and `RemoteAttachmentCodec`, see [Handle different content types](#handle-different-content-types).

## Handle conversations

Most of the time, when interacting with the network, you'll want to do it through `conversations`. Conversations are between two accounts.

```swift
import XMTP
// Create the client with a wallet from your app
let client = try await Client.create(account: account)
let conversations = try await client.conversations.list()
```

### List existing conversations

You can get a list of all conversations that have one or more messages.

```swift
let allConversations = try await client.conversations.list()

for conversation in allConversations {
  print("Saying GM to \(conversation.peerAddress)")
  try await conversation.send(content: "gm")
}
```

These conversations include all conversations for a user **regardless of which app created the conversation.** This functionality provides the concept of an [interoperable inbox](/docs/concepts/interoperable-inbox), which enables a user to access all of their conversations in any app built with XMTP.


### Listen for new conversations

You can also listen for new conversations being started in real-time. This will allow apps to display incoming messages from new contacts.

:::caution

This stream will continue infinitely. To end the stream, break from the loop.

:::

```swift
for try await conversation in client.conversations.stream() {
  print("New conversation started with \(conversation.peerAddress)")

  // Say hello to your new friend
  try await conversation.send(content: "Hi there!")

  // Break from the loop to stop listening
  break
}
```

### Start a new conversation

You can create a new conversation with any Ethereum address on the XMTP network.

```swift
let newConversation = try await client.conversations.newConversation(with: "0x3F11b27F323b62B159D2642964fa27C46C841897")
```

### Send messages

To be able to send a message, the recipient must have already created a client at least once and consequently advertised their key bundle on the network. Messages are addressed using account addresses. By default, the message payload supports plain strings.

To learn about support for other content types, see [Handle different content types](#handle-different-content-types).

```swift
let conversation = try await client.conversations.newConversation(with: "0x3F11b27F323b62B159D2642964fa27C46C841897")
try await conversation.send(content: "Hello world")
```

### List messages in a conversation

You can receive the complete message history in a conversation by calling `conversation.messages()`.

```swift
for conversation in client.conversations.list() {
  let messagesInConversation = try await conversation.messages()
}
```

### List messages in a conversation with pagination

It may be helpful to retrieve and process the messages in a conversation page by page. You can do this by calling `conversation.messages(limit: Int, before: Date)` which will return the specified number of messages sent before that time.

```swift
let conversation = try await client.conversations.newConversation(with: "0x3F11b27F323b62B159D2642964fa27C46C841897")

let messages = try await conversation.messages(limit: 25)
let nextPage = try await conversation.messages(limit: 25, before: messages[0].sent)
```

### Listen for new messages in a conversation

You can listen for any new messages (incoming or outgoing) in a conversation by calling `conversation.streamMessages()`.

A successfully received message (that makes it through the decoding and decryption without throwing) can be trusted to be authentic. Authentic means that it was sent by the owner of the `message.senderAddress` account and that it wasn't modified in transit. The `message.sent` timestamp can be trusted to have been set by the sender.

The stream returned by the `stream` methods is an asynchronous iterator and as such is usable by a for-await-of loop. Note however that it is by its nature infinite, so any looping construct used with it will not terminate, unless the termination is explicitly initiated (by breaking the loop).

```swift
let conversation = try await client.conversations.newConversation(with: "0x3F11b27F323b62B159D2642964fa27C46C841897")

for try await message in conversation.streamMessages() {
  if message.senderAddress == client.address {
    // This message was sent from me
    continue
  }

  print("New message from \(message.senderAddress): \(message.body)")
}
```


### Decode a single message

You can decode a single `Envelope` from XMTP using the `decode` method:

```swift
let conversation = try await client.conversations.newConversation(with: "0x3F11b27F323b62B159D2642964fa27C46C841897")

// Assume this function returns an Envelope that contains a message for the above conversation
let envelope = getEnvelopeFromXMTP()

let decodedMessage = try conversation.decode(envelope)
```

### Serialize/Deserialize conversations

You can save a conversation object locally using its `encodedContainer` property. This returns a `ConversationContainer` object which conforms to `Codable`.

```swift
// Get a conversation
let conversation = try await client.conversations.newConversation(with: "0x3F11b27F323b62B159D2642964fa27C46C841897")

// Get a container.
let container = conversation.encodedContainer

// Dump it to JSON
let encoder = JSONEncoder()
let data = try encoder.encode(container)

// Get it back from JSON
let decoder = JSONDecoder()
let containerAgain = try decoder.decode(ConversationContainer.self, from: data)

// Get an actual Conversation object like we had above
let decodedConversation = containerAgain.decode(with: client)
try await decodedConversation.send(text: "hi")
```

## Handle different content types

All of the send functions support `SendOptions` as an optional parameter. The `contentType` option allows specifying different types of content other than the default simple string standard content type, which is identified with content type identifier `ContentTypeText`.

To learn more about content types, see [Content types with XMTP](/docs/concepts/content-types).

Support for other content types can be added by registering additional `ContentCodec`s with the client. Every codec is associated with a content type identifier, `ContentTypeID`, which is used to signal to the client which codec should be used to process the content that is being sent or received.

For example, see the [Codecs](https://github.com/xmtp/xmtp-ios/tree/main/Sources/XMTP/Codecs) available in `xmtp-ios`.

### Send a remote attachment

Use the [RemoteAttachmentCodec](https://github.com/xmtp/xmtp-ios/blob/main/Sources/XMTP/Codecs/RemoteAttachmentCodec.swift) package to enable your app to send and receive message attachments.

Message attachments are files. More specifically, attachments are objects that have:

- `filename` Most files have names, at least the most common file types.
- `mimeType` What kind of file is it? You can often assume this from the file extension, but it's nice to have a specific field for it. [Here's a list of common mime types.](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Common_types)
- `data` What is this file's data? Most files have data. If the file doesn't have data then it's probably not the most interesting thing to send.

Because XMTP messages can only be up to 1MB in size, we need to store the attachment somewhere other than the XMTP network. In other words, we need to store it in a remote location.

End-to-end encryption must apply not only to XMTP messages, but to message attachments as well. For this reason, we need to encrypt the attachment before we store it.

#### Create an attachment object

```swift
let attachment = Attachment(
  filename: "screenshot.png",
  mimeType: "image/png",
  data: Data(somePNGData)
)
```

#### Encrypt the attachment

Use the `RemoteAttachmentCodec.encodeEncrypted` to encrypt the attachment:

```swift
// Encode the attachment and encrypt that encoded content
const encryptedAttachment = try RemoteAttachment.encodeEncrypted(
	content: attachment,
	codec: AttachmentCodec()
)
```

#### Upload the encrypted attachment

Upload the encrypted attachment anywhere where it will be accessible via an HTTPS GET request. For example, you can use web3.storage:

```swift
func upload(data: Data, token: String): String {
  let url = URL(string: "https://api.web3.storage/upload")!
  var request = URLRequest(url: url)
  request.addValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
  request.addValue("XMTP", forHTTPHeaderField: "X-NAME")
  request.httpMethod = "POST"

  let responseData = try await URLSession.shared.upload(for: request, from: data).0
  let response = try JSONDecoder().decode(Web3Storage.Response.self, from: responseData)

  return "https://\(response.cid).ipfs.w3s.link"
}

let url = upload(data: encryptedAttachment.payload, token: YOUR_WEB3_STORAGE_TOKEN)
```

#### Create a remote attachment

Now that you have a `url`, you can create a `RemoteAttachment`.

```swift
let remoteAttachment = try RemoteAttachment(
  url: url,
  encryptedEncodedContent: encryptedEncodedContent
)
```

#### Send a remote attachment

Now that you have a remote attachment, you can send it:

```swift
try await conversation.send(
	content: remoteAttachment,
	options: .init(
		contentType: ContentTypeRemoteAttachment,
		contentFallback: "a description of the image"
	)
)
```

Note that we’re using `contentFallback` to enable clients that don't support these content types to still display something. For cases where clients _do_ support these types, they can use the content fallback as alt text for accessibility purposes.

#### Receive a remote attachment

Now that you can send a remote attachment, you need a way to receive a remote attachment. For example:

```swift
let messages = try await conversation.messages()
let message = messages[0]

guard message.encodedContent.contentType == ContentTypeRemoteAttachment else {
	return
}

const remoteAttachment: RemoteAttachment = try message.content()
```

#### Download, decrypt, and decode the attachment

Now that you can receive a remote attachment, you need to download, decrypt, and decode it so your app can display it. For example:

```swift
let attachment: Attachment = try await remoteAttachment.content()
```

You now have the original attachment:

```swift
attachment.filename // => "screenshot.png"
attachment.mimeType // => "image/png",
attachment.data // => [the PNG data]
```

#### Display the attachment

Display the attachment in your app as you please. For example, you can display it as an image:

```swift
import UIKIt
import SwiftUI

struct ContentView: View {
	var body: some View {
		Image(uiImage: UIImage(data: attachment.data))
	}
}

```

#### Handle custom content types

Beyond this, custom codecs and content types may be proposed as interoperable standards through XRCs. To learn more about the custom content type proposal process, see [XIP-5](https://github.com/xmtp/XIPs/blob/main/XIPs/xip-5-message-content-types.md).

## Compression

Message content can be optionally compressed using the compression option. The value of the option is the name of the compression algorithm to use. Currently supported are gzip and deflate. Compression is applied to the bytes produced by the content codec.

Content will be decompressed transparently on the receiving end. Note that Client enforces maximum content size. The default limit can be overridden through the ClientOptions. Consequently, a message that would expand beyond that limit on the receiving end will fail to decode.

```swift
try await conversation.send(text: '#'.repeat(1000), options: .init(compression: .gzip))
```

## Breaking revisions

Because `xmtp-ios` is in active development, you should expect breaking revisions that might require you to adopt the latest SDK release to enable your app to continue working as expected.

XMTP communicates about breaking revisions in the [XMTP Discord community](https://discord.gg/xmtp), providing as much advance notice as possible. Additionally, breaking revisions in an `xmtp-ios` release are described on the [Releases page](https://github.com/xmtp/xmtp-ios/releases).

## Deprecation

Older versions of the SDK will eventually be deprecated, which means:

1. The network will not support and eventually actively reject connections from clients using deprecated versions.
2. Bugs will not be fixed in deprecated versions.

The following table provides the deprecation schedule.

| Announced                                                        | Effective | Minimum Version | Rationale |
| ---------------------------------------------------------------- | --------- | --------------- | --------- |
| There are no deprecations scheduled for `xmtp-ios` at this time. |           |                 |           |

Bug reports, feature requests, and PRs are welcome in accordance with these [contribution guidelines](https://github.com/xmtp/xmtp-ios/blob/main/CONTRIBUTING.md).
