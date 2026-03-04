# Session Chat Architecture (Open Collaboration Tools)

This document explains how **session chat** is implemented end-to-end in Open Collaboration Tools, with concrete function-level references.

## Scope

This covers the chat path across:

- `packages/open-collaboration-vscode` (chat webview UI + extension bridge)
- `packages/open-collaboration-protocol` (chat API, message types, encryption-aware connection)
- `packages/open-collaboration-server` (generic relay/routing across peers)

It focuses on:

- normal room messages (broadcast)
- direct messages (peer-targeted notification)
- typing indicator events
- in-memory chat history behavior

---

## 1) High-level architecture

### VS Code extension layer

- `chat-webview/src/webview.tsx` renders the **Session Chat** panel and receives pushed messages.
- `chat-webview/src/message-input.tsx` captures user input, recipient selection, and typing notifications.
- `chat-webview/chat-webview.ts` bridges webview messages to `CollaborationInstance.Current.connection.chat.*` APIs and pushes incoming chat events back into the webview.

### Protocol layer

- `open-collaboration-protocol/src/connection.ts` exposes `ChatHandler` and maps chat calls to protocol message channels.
- `open-collaboration-protocol/src/messages.ts` defines chat methods:
  - `chat/message` (broadcast)
  - `chat/directMessage` (notification to specific peer)
  - `chat/writing` (broadcast typing indicator)
- `open-collaboration-protocol/src/messaging/abstract-connection.ts` performs send/receive, handler dispatch, and encryption/decryption.

### Server layer

- `open-collaboration-server/src/peer.ts` receives inbound protocol messages and routes by message kind.
- `open-collaboration-server/src/message-relay.ts` forwards notifications and broadcasts to target peers.
- `open-collaboration-server/src/channel.ts` is the per-peer transport wrapper (with reconnect buffering).

---

## 2) Chat initialization and wiring

### Extension activation

Entry point: `open-collaboration-vscode/src/extension.ts`

- `activate(...)` creates DI container and calls:
  - `container.get(ChatWebview).register()`
- This is what enables the chat webview provider (`oct.chatView`) for the extension runtime.

### Chat webview registration

File: `open-collaboration-vscode/src/chat-webview/chat-webview.ts`

- `ChatWebview.register()`:
  - registers the view provider via `vscode.window.registerWebviewViewProvider`
  - creates `Messenger`
  - subscribes to room lifecycle through `roomService.onDidJoinRoom(...)`

When a room is active, it attaches handlers on the collaboration connection:

- `collabInstance.connection.chat.onMessage(...)`
- `collabInstance.connection.chat.onIsWriting(...)`
- `collabInstance.onDidUsersChange(...)`
- `collabInstance.onDidDispose(...)`

This is the core bridge from protocol events to webview updates.

---

## 3) Send flow: UI -> extension -> protocol -> server -> peers

## 3.1 Webview UI sends message intent

File: `open-collaboration-vscode/src/chat-webview/src/message-input.tsx`

- `MessageInput` defines `sendChatMessage(target?)`.
- On Enter/click send, it calls:
  - `messenger.sendNotification(sendMessage, { type: 'extension' }, { message, target })`
- It also updates local UI state immediately with a synthetic local message:
  - `{ user: 'me', message, isDirect: !!target, timestamp: Date.now() }`

### 3.2 Extension receives webview send notification

File: `open-collaboration-vscode/src/chat-webview/chat-webview.ts`

- In `registerMessengerHandlers(...)`, this handler is installed:
  - `this.messenger.onNotification(sendMessage, ...)`
- Behavior:
  - appends `{ user: 'me', message, isDirect }` to `chatHistory`
  - if `target` exists: `connection.chat.sendDirectMessage(target, message)`
  - else: `connection.chat.sendMessage(message)`

### 3.3 Protocol converts API call to message kinds

File: `open-collaboration-protocol/src/connection.ts`

`chat: ChatHandler` maps methods as follows:

- `sendMessage(message)` -> `sendBroadcast(Messages.Chat.ChatMessage, message)`
- `sendDirectMessage(target, message)` -> `sendNotification(Messages.Chat.DirectChatMessage, target, message)`

File: `open-collaboration-protocol/src/messages.ts`

- `Messages.Chat.ChatMessage = BroadcastType('chat/message')`
- `Messages.Chat.DirectChatMessage = NotificationType('chat/directMessage')`

### 3.4 Protocol transport and encryption

File: `open-collaboration-protocol/src/messaging/abstract-connection.ts`

- `sendBroadcast(...)`:
  - builds a broadcast message
  - encrypts for all known peer public keys
  - writes through transport
- `sendNotification(...)`:
  - builds notification with explicit `target`
  - encrypts for the target peer public key
  - writes through transport

### 3.5 Server relays to room peers

Files:

- `open-collaboration-server/src/peer.ts`
- `open-collaboration-server/src/message-relay.ts`

Flow on server:

1. `PeerImpl.receiveMessage(...)` identifies message kind.
2. For `NotificationMessage` (direct chat), it calls `messageRelay.sendNotification(targetPeer, message)`.
3. For `BroadcastMessage` (room chat), it calls `messageRelay.sendBroadcast(originPeer, message)`.

`MessageRelay.sendBroadcast(...)` iterates `room.peers` except sender and forwards one encrypted key payload per recipient (when encrypted).

### 3.6 Recipient side receives and pushes to chat UI

File: `open-collaboration-protocol/src/connection.ts`

- `chat.onMessage(handler)` subscribes to:
  - broadcast `Messages.Chat.ChatMessage` -> callback `(origin, msg, false)`
  - notification `Messages.Chat.DirectChatMessage` -> callback `(origin, msg, true)`

File: `open-collaboration-vscode/src/chat-webview/chat-webview.ts`

- In `connection.chat.onMessage(...)` callback:
  - finds user metadata via `CollaborationInstance.Current?.connectedUsers`
  - creates `ChatMessage` object `{ message, user, color, isDirect }`
  - pushes into `chatHistory`
  - sends to active webview via `messenger.sendNotification(messageReceived, webviewId, messageObj)`

File: `open-collaboration-vscode/src/chat-webview/src/webview.tsx`

- `App` subscribes via `messenger.onNotification(messageReceived, ...)`
- appends message to UI state and renders bubble in Session Chat.

---

## 4) Typing indicator flow

### Send typing signal

File: `open-collaboration-vscode/src/chat-webview/src/message-input.tsx`

- On textarea change, `sendWritingNotification()` is triggered.
- It is throttled (`lodash/throttle`) so signals are not sent excessively.
- It emits webview notification `isWriting` to extension.

File: `open-collaboration-vscode/src/chat-webview/chat-webview.ts`

- `onNotification(isWriting, ...)` calls `connection.chat.isWriting()`.

File: `open-collaboration-protocol/src/connection.ts`

- `chat.isWriting()` -> `sendBroadcast(Messages.Chat.IsWriting)`

### Receive typing signal

File: `open-collaboration-protocol/src/connection.ts`

- `chat.onIsWriting(handler)` subscribes to `Messages.Chat.IsWriting` broadcast.

File: `open-collaboration-vscode/src/chat-webview/chat-webview.ts`

- On protocol `onIsWriting(userId)` it forwards to webview notification `isWriting`.

File: `open-collaboration-vscode/src/chat-webview/src/message-input.tsx`

- Webview tracks currently-writing users in `usersWriting` with timeout-based auto-clear.

---

## 5) Chat history and state behavior

### Where history is stored

File: `open-collaboration-vscode/src/chat-webview/chat-webview.ts`

- `private chatHistory: ChatMessage[] = []` stores chat history in extension memory.
- History is returned to webview via `onRequest(getHistory, ...)`.
- On room disposal (`collabInstance.onDidDispose`), history is cleared.

### Persistence model

- Chat history is **not persisted server-side** by these modules.
- Server acts as relay/router, not long-term chat storage.
- If extension host reloads, in-memory history is lost unless another layer persists it externally.

---

## 6) Direct vs broadcast semantics

### Broadcast (`sendMessage`)

- Delivered to all peers in room except sender.
- Marked in UI with `isDirect = false`.

### Direct (`sendDirectMessage`)

- Delivered only to selected target peer.
- Marked in UI with `isDirect = true`.
- In UI, direct messages are shown with a `*` marker next to sender name.

---

## 7) Transport/reconnect characteristics impacting chat

### Protocol transport reconnect

File: `open-collaboration-protocol/src/transport/socket-io-transport.ts`

- Tracks socket reconnect/disconnect.
- Delays hard disconnect event by 30s for transient network drops.

### Server channel buffering

File: `open-collaboration-server/src/channel.ts`

- `Channel.sendMessage(...)` buffers outbound messages while transport is absent.
- On transport restoration, buffered messages flush.

This improves resilience for short disconnect windows, though no durable persistence is guaranteed.

---

## 8) Session lifecycle context for chat

Chat is available only after the collaboration room is connected and `CollaborationInstance.Current` exists.

Relevant room/session orchestration:

- `open-collaboration-vscode/src/collaboration-room-service.ts`
  - room creation/joining (`createRoom`, `joinRoom`, `tryConnect`)
  - emits `onDidJoinRoom` used by `ChatWebview`
- `open-collaboration-vscode/src/collaboration-instance.ts`
  - keeps current connection and user identity data used by chat sender/recipient metadata

---

## 9) Function index (quick lookup)

### VS Code chat integration

- `ChatWebview.register`
- `ChatWebview.resolveWebviewView`
- `ChatWebview.registerMessengerHandlers`
- `ChatWebview.getOtherUsers`
- `MessageInput` (component)
- `sendChatMessage`
- `sendWritingNotification`
- `App` (webview root)

### Protocol chat API

- `createConnection`
- `ProtocolBroadcastConnectionImpl.chat.sendMessage`
- `ProtocolBroadcastConnectionImpl.chat.sendDirectMessage`
- `ProtocolBroadcastConnectionImpl.chat.onMessage`
- `ProtocolBroadcastConnectionImpl.chat.isWriting`
- `ProtocolBroadcastConnectionImpl.chat.onIsWriting`

### Protocol transport core

- `AbstractBroadcastConnection.sendBroadcast`
- `AbstractBroadcastConnection.sendNotification`
- `AbstractBroadcastConnection.handleMessage`

### Server relay core

- `PeerImpl.receiveMessage`
- `MessageRelay.sendBroadcast`
- `MessageRelay.sendNotification`
- `Channel.sendMessage`

---

## 10) Key takeaway

Session chat behavior is implemented primarily in the **VS Code extension + protocol connection layer**. The server is intentionally chat-agnostic and forwards encrypted protocol messages based on type (broadcast/notification) and target peer/room context.

