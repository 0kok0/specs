# Status Group Chat Specification

> Version: 0.1 (Draft)
>
> Authors: Andrea Maria Piana <andreap@status.im>
>


## Table of Contents

- [Abstract](#abstract)
- [Membership updates](#membership-updates)
  - [Chat ID](#chat-id)
  - [Signature](#signature)
  - [Group membership event](#group-membership-event)
    - [chat-created](#chat-created)
    - [name-changed](#name-changed)
    - [members-added](#members-added)
    - [members-joined](#members-joined)
    - [admins-added](#admins-added)
    - [members-removed](#members-removed)
    - [admin-removed](#admin-removed)


## Abstract

This documents describes the group chat protocol used by the status application. Pairwise encryption is used among member so a message is exchanged between each participants, similarly to a one-to-one message.

## Membership updates

Membership updates messages are used to propagate group chat membership changes. The protobuf format is described in the [Status Payload Specs](status-payload-specs.md). Here we will be describing each specific field.

The protobuf messages are:

```protobuf
// MembershipUpdateMessage is a message used to propagate information
// about group membership changes.
message MembershipUpdateMessage {
  // The chat id of the private group chat
  string chat_id = 1;
  // A list of events for this group chat, first 65 bytes are the signature, then is a 
  // protobuf encoded MembershipUpdateEvent
  repeated bytes events = 2;
  // An optional chat message
  ChatMessage message = 3;
}

message MembershipUpdateEvent {
  // Lamport timestamp of the event as described in [Status Payload Specs](status-payload-specs.md#clock-vs-timestamp-and-message-ordering)
  uint64 clock = 1;
  // List of public keys of the targets of the action
  repeated string members = 2;
  // Name of the chat for the CHAT_CREATED/NAME_CHANGED event types
  string name = 3;
  // The type of the event
  EventType type = 4;

  enum EventType {
    UNKNOWN = 0;
    CHAT_CREATED = 1; // See [CHAT_CREATED](#chat-created)
    NAME_CHANGED = 2; // See [NAME_CHANGED](#name-changed)
    MEMBERS_ADDED = 3; // See [MEMBERS_ADDED](#members-added)
    MEMBER_JOINED = 4; // See [MEMBER_JOINED](#member-joined)
    MEMBER_REMOVED = 5; // See [MEMBER_REMOVED](#member-removed)
    ADMINS_ADDED = 6; // See [ADMINS_ADDED](#admins-added)
    ADMIN_REMOVED = 7; // See [ADMIN_REMOVED](#admin-removed)
  }
}
```

### Payload

`MembershipUpdateMessage`:

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | chat-id | `string` | The chat id of the chat where the change is to take place |
| 2 | events | See details | A list of events that describe the membership changes, in their encoded protobuf form |
| 3 | message | `ChatMessage` | An optional message, described in [Message](#message) |

`MembershipUpdateEvent`:

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | The clock value of the event |
| 2 | members | `[]string` | An optional list of hex encoded (prefixed with `0x`) public keys, the targets of the action |
| 3 | name | `name` | An optional name, for those events that make use of it |
| 4 | type | `EventType` | The type of event sent, described below |


### Chat ID

Each membership update MUST be sent with a corresponding `chatId`. 
The format of this chat ID MUST be a string of [UUID](https://tools.ietf.org/html/rfc4122 ), concatenated with the hex-encoded public key of the creator of the chat, joined by `-`. This chatId MUST be validated by all clients, and MUST be discarded if it does not follow these rules.

### Signature

The signature for each event is calculated by encoding each `MembershipUpdateEvent` in its protobuf representation and prepending the bytes of the chatID, lastly the `Keccak256` of the bytes is signed using the private key by the author and added to the `events` field of MembershipUpdateMessage.
      
### Group membership event

Any group membership event received MUST be verified by calculating the signature as per the method described above. 
The author MUST be extracted from it, if the verification fails the event MUST be discarded.

#### CHAT_CREATED

Chat created event is the first event that needs to be sent. Any event with a clock value lower then this MUST be discarded.
Upon receiving this event a client MUST validate the `chatId` provided with the updates and create a chat with identified by `chatId` and named `name`.

#### NAME_CHANGED

A name changed event is used by admins to change the name of the group chat.
Upon receiving this event a client MUST validate the `chatId` provided with the updates and MUST ensure the author of the event is an admin of the chat, otherwise the event MUST be ignored.
If the event is valid the chat name SHOULD be changed to `name`.

#### MEMBERS_ADDED

A members added event is used by admins to add members to the chat.
Upon receiving this event a client MUST validate the `chatId` provided with the updates and MUST ensure the author of the event is an admin of the chat, otherwise the event MUST be ignored.
If the event is valid a client MUST update the list of members of the chat who have not joined, adding the `members` received.
`members` is an array of hex encoded public keys.

#### MEMBER_JOINED

A members joined event is used by a member of the chat to signal that they want to start receiving messages from this chat.
Upon receiving this event a client MUST validate the `chatId` provided with the updates.
If the event is valid a client MUST update the list of members of the chat who joined, adding the signer. Any `message` sent to the group chat should now include the newly joined member.

#### ADMINS_ADDED

An admins added event is used by admins to add make other admins in the chat.
Upon receiving this event a client MUST validate the `chatId` provided with the updates, MUST ensure the author of the event is an admin of the chat and MUST ensure all `members` are already `members` of the chat, otherwise the event MUST be ignored.
If the event is valid a client MUST update the list of admins of the chat, adding the `members` received.
`members` is an array of hex encoded public keys.

#### MEMBER_REMOVED

A member-removed event is used to leave or kick members of the chat.
Upon receiving this event a client MUST validate the `chatId` provided with the updates, MUST ensure that:
- If the author of the event is an admin, target can only be themselves or a non-admin member.
- If the author of the event is not an admin, the target of the event can only be themselves.
-
If the event is valid a client MUST remove the member from the list of `members`/`admins` of the chat, and no further message should be sent to them.

#### ADMIN_REMOVED

An admin-removed event is used to drop admin privileges.
Upon receiving this event a client MUST validate the `chatId` provided with the updates, MUST ensure that the author of the event is also the target of the event.

If the event is valid a client MUST remove the member from the list of `admins` of the chat.
