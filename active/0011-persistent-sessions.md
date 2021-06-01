# Persistent sessions

## Change log

* 2021-XX-XX: @gorillainduction Initial draft

## Abstract

Persistent sessions is today relying on that there is a process alive
to collect all delivered messages. The queue of undelivered messages
to the session is stored in memory in the context of this process.

We propose a way of persisting sessions and messages so that they
survive a node restart.

## Motivation

A cluster of brokers should be able to maintain persistent sessions
even if the node the session last was seen on goes away or is
restarted.

## Design

The foundation of a persistent session is setting the
`'Session-Expiry-Interval'`.

* The Client and Server MUST NOT discard the Session State while the
  Network Connection is open `[MQTT-4.1.0-1]`.
* The Server MUST discard the Session State when the Network
  Connection is closed and the Session Expiry Interval has passed
  `[MQTT-4.1.0-2]`.

When connecting with a new client, using the same session id as an
active session will eiter take over the session or create a new
session with the same name.

* If a CONNECT packet is received with `Clean Start` is set to 1, the
  Client and Server MUST discard any existing Session and start a new
  Session. `[MQTT-3.1.2-4]`
* If a CONNECT packet is received with `Clean Start` set to 0 and
  there is a Session associated with the Client Identifier, the Server
  MUST resume communications with the Client based on state from the
  existing Session. `[MQTT-3.1.2-5]`

### Current implementation

In the current implementation, this is handled by keeping the
connection process alive even after a disconnect until the session
expiry interval has passed.

When a new connection with a given client id is established:
* A session is opened through `emqx_cm` (channel manager).
* The session id is locked through `emqx_cm_locker`.
* The `emqx_cm_registry` is searched for an active session.

For non-persistent sessions (i.e., `Clean Start` is set to 1) the
handling is straightforward. For persistent sessions (i.e., `Clean
Start` is set to 0) the session state needs to be taken over.

1. The active session is told to pause its handling of messages. This
   entails buffering any new messages that arrive (through a
   `gen_server:call`).
1. The active session confirms that it is on hold and sends the
   session state to the new process (i.e., the `#session{}`).
1. The new process inherits the session state.
1. The session state contains information about its active
   subscriptions, which the new process can use to subscribe to the
   same topics.
1. The new process tells the old process to shut down (through a
   `gen_server:call`).
1. The new process confirms that it is shutting down and sends any new
   messages it got while being on hold.
1. The new process registers itself as the new process for the session
   id through `emqx_cm_registry`. This data is stored in Mnesia (in a
   `ram_copies` table)
1. The new process releases the lock on the session id.
1. The new process sends an `CONNACK` to the client with the flag
   `Session Present` set to 1, indicating that the broker had knowledge
   of the session and that it was taken over.
1. Any pending messages are delivered to the client.

### Alternatives for persistent storage

#### Semi-persistent storage in ram

When a connection with 'Session-Expiry-Interval' is started at one
node corresponding processes could be started at all (or some) other
nodes, without a connection process. This would mimic the current
behavior and require some interaction between the different session
processes. As long as one node would remain up, the session would be
persisted.

Pros:
* Relatively small changes to the current approach.
* Easy to ensure subscriptions keeps all messages.
* Can probably make the current split brain of sessions situation
  better on netsplits.

Cons
* Requires some coordination from the "master" process to the replicants.
* Requires more resources since all persistent sessions are replicated
  troughout the cluster.
* If the whole cluster restarts, information is lost.

#### Persistent session storage using database
The relevant information about the session (e.g., `#session{}`) can be
stored on update by the connection process as well as any undelivered
messages. A new database table would store the session. The initial
version would probably be stored in a disk-based Mnesia table, but
there could probably be pluggable backends for this.

When a node goes down, some new component would restart sessions that
was lost but still was not due for expiration. Some care would need to
be taken to not lose messages that were sent during the time when no
session process existed.

Pros:
* Relatively stable information in the DB instead of keeping the
  information in volatile memory (i.e., process memory).
* Information survives node restarts.

Cons:
* It is somewhat hard to bridge the gap of messages to a subscription
  while the session is unavailable.

#### Persistent publish to persistent subscribers.
A different approach, or rather a complement to the approaches above,
would be to store messages to persistent sessions before they are
forwarded to the subscribing process.

The subscribing mechanism could keep track of which subscriptions that
are persistent, and store the messages on publish. In that way, a
connection process could ask for any messages on connect rather than
getting them from another process' state.

The message storage should be in a DB that all nodes have access to so
that the information is spread around the cluster.

Note that the session state would have to be stored in a different way
as well.


## Configuration Changes

> This section should list all the changes to the configuration files (if any).

## Backwards Compatibility

> This sections should shows how to make the feature is backwards compatible.
> If it can not be compatible with the previous emqx versions, explain how do you
> propose to deal with the incompatibilities.

## Document Changes

> If there is any document change, give a brief description of it here.

## Testing Suggestions

> The final implementation must include unit test or common test code. If some
> more tests such as integration test or benchmarking test that need to be done
> manually, list them here.

## Declined Alternatives

> Here goes which alternatives were discussed but considered worse than the current.
> It's to help people understand how we reached the current state and also to
> prevent going through the discussion again when an old alternative is brought
> up again in the future.
