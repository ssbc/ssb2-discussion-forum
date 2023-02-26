# Minibutt feed format

Status: **Design**

Minibutt is a new binary feed format for [SSB]. It draws inspiration
from many different sources: [buttwoo], [bamboo], [meta feeds],
[earthstar].

The format is designed to be simple, support edit and delete, support
multiple devices sharing the same key, enable partitioning data into
subfeeds and lastly to be performant.

Minibutt uses [bipf] for encoding.

Let's delve a bit further into the simplity.

SSB has a lot of complexity and it is hard and time consuming to build
stuff on top of it. While you can always argue to use more standard
tools, what we try here is to change the foundation in such a way as
to make it simpler to build things on top.

First of all by using the same author for the all the subfeeds we
remove a lot of complexity that [meta feeds] has. This was one of the
benefits of [buttwoo]. Just simply debugging things are massively
easier because you don't need a whole mapping structure to know what
feeds belong together.

Similar can be said about multi device support. We already created a
[fusion identity] spec for tying multiple identities together, and
there exists an implementation. It has never been but into production,
because the implemenation is the easy part, the UI is a lot
trickier. And once you start creating things like private groups where
it is important who exactly has access, then that needs to take fusion
identities or multiple identities into account as well. Thus
complicating things because everything are abstractions built on top
of simple linear feed foundations. If instead we changed the
foundation such that a key can be shared, then we use the same author
again and things become simpler.

The feeds in classic SSB are linear and has to be. This also means
that it has problems with forking an identity where, most commonly,
when restoring an identity the latest messsage was not properly
fetched before writing new messages. If we remove this strictly linear
restriction then we don't have this problem. Now for some applications
we need to order, but we already have an abstraction we can use for
this, the tangle that ties messages from multiple author together into
a directed acyclic graph.

Lastly we would also like to enable partial replication, not only of
particular partition (subfeed), but also maybe just the latest 20
messages from and author. And efficiently handle data that can be
heavily edited using a sparse replication mode.

## Format

A minibutt message consists of a bipf-encoded array of 3 fields:

- `metadata`
- `signature`
- `content`

The message ID is the first 16 bytes of the author concatenated with
the 16 first bytes of the the [blake3] hash of the concatenation of
`metadata` bytes with `signature` bytes.

### Metadata

The `metadata` field is a bipf-encoded array with 7 fields:

 - `author`: an ed25519 public key
 - `subfeed`: a string to identity the subfeed if used.
 - `sequence`: an integer representing the position for this message in
   the feed. Starts from 1.
 - `timestamp`: a double representing the UNIX epoch timestamp of 
   message creation
 - `tag`: a byte with extensible tag information (the value `0x00`
   means a standard message, `0x01` means subfeed, `0x02` means
   end-of-feed). In future versions other tags to mean something
   else.
 - `contentLength`: an integer encoding the length of the bipf-encoded `content` in bytes
 - `contentHash`: the first 16 bytes of the [blake3] hash of the bipf-encoded
    `content` bytes
    
It is important to note that one author can have multiple feeds, each
feed is defined as author + subfeed. `sequence` relates to the feed.

FIXME: we might need to BFE encode these for the database. Maybe it's
a bad idea not to encode them, I'm just not sure we ever will need to
ability to change e.g. the hash function used.

### Signature

The `signature` uses the same HMAC signing capability (`sodium.crypto_auth`) 
and `sodium.crypto_sign_detached` as in the classic SSB format (ed25519).

### Content

The `content` is a free form field. When unencrypted, it MUST be a
bipf-encoded object. If encrypted, `content` MUST be an [ssb-bfe
encrypted data format].

## Subfeeds

To be spec'ed. We can start small and just say that we use the
following special names for feeds useful for a social media app:

 - profile: stores the latest profile information, it is enough to
   just store the latest message in this feed unless you want the
   history.
 - follow: stores the list of other feeds you follow, it is enough to
   just store the latest message in this feed unless you want the
   history.
 - block: stores the list of other feeds you block, it is enough to
   just store the latest message in this feed unless you want the
   history.
 - post: a stream of messages for text messsages
 - reaction: a stream of messages for reactions to messsages
 
FIXME: should we store room / pub information in a feed?

FIXME: should we store blocked blobs similar to blocked feeds?

## Performance

To be tested, but should be faster than buttwoo.

Similar to classic it is possible for lite clients to queue up
messages for validation and only check the signature of a random or
the latest message. This can improve the bulk validation substantially
in onboarding situations.

## Size

A message is roughly 137 bytes + encoding. This is a ~35% improvement
over buttwoo which is already a 20% size reduction in network traffic
compared to classic format.

## Validation

To be spec'd (quite similar to buttwoo) except:

 - no message size limit (needed for follow/block), instead we rely on
   clients using their best judgement to skip messages if they are
   gigantic.
 - Since we don't have previous (to enable multi-device and fork
   recovery) and seq's are not unique, we can only really rely on
   timestamps, so they must always be increasing. This makes simple
   replication easier.

## Simple replication

It will probably be a good idea to have some sort of set replication
for these messages, but coming from the complexity standpoint I find
it important that we also have a simple replication mechanism as well.

### Request

A dictionary of:

```  
 feedId => { 
    subfeed => {
      timestamp, // double
      limit, // int
      reverse, // bool
      sparse // bool
    }
  }
```

Sparse here means: don't replicate a message that has been edited or
deleted. The request could also include a live flag.

There should be a special variant of this that just returns the count
instead of the messages so you can check if you are missing something.

Sending this dictionary does come with some overhead as it would be
sent on each connection. The overhead should be roughly: 32 + 5
feeds * (8 + 8 + 4 + 1 + 1) = 142 bytes for each author. For 300
authors this is ~ 42.5kb. Note that this is per author not per device
as in SSB. So this scales okay for hops 1, but wouldn't work for hops
2 or 3 kind of replication where you have several thousand feeds. This
is also why EBT included same caching at the cost of complexity, so
you would only need to send what has actually changed.

Performance of this has not been tested in real world.

### Response

Messages sent over the wire should be bipf encoded as:

```
transport:  [metadata, signature, content]
metadata:   [author, subfeed, sequence, timestamp, tag, contentLen, contentHash]
```

## Design choices

### Edit / delete

While an edit message can seem like an application level concern, the
reason why it is included in this document is to make sure that we can
efficiently handle edits without compromising existing guarantees. An
edit message is a message that changes an existing content of a
message, it needs to reference the existing message in the edit
tangle:

```
  "tangles" => {
    "edit" => {
      "root" => originalMsgId,
      "previous" => originalMsgId or previousEditMsgId
    }
  },
```

A delete is just a special message in that it replaces the existing
message with an empty message. Thus it is also part of the same edit
tangle. A peer upon receiving a delete message should remove the
content from the original message and all edits.

Because we store the content hash in the chain, the chain is intact,
but can be efficiently replicated in sparse mode where these messages
will be left out. It's hard to prove that someone did not leave out
anything they should not in sparse mode, which is why it is important
to replicate from multiple peers and to check the count of messages
and not only rely on latest messages based on timestamp.

It might be that you replicate with a peer that doesn't have the
content of previous versions in an edit. The chain is still intact,
but you might want to content to show the history. For this, a special
api is needed to request these similar to blobs.

To make things simple, you can edit your own messages.

### Message id

Normally the message id has always been the hash of the complete
message. Instead here we want to include the author, such that if
there is a message in a thread you can't read, you know it might be
from someone you blocked. 16 bytes should be enough to keep things
secure (see https://github.com/BLAKE3-team/BLAKE3/issues/123). 

### Same author on multiple devices

This was as explained in the into mainly done to simplify
things. There is one place where this breaks down though, which is in
multiserver and rooms where peers are not unique anymore. I believe it
should be possible to fix this in secret handshake ([SHS]) instead
with some simple numbering system.

[SSB]: https://ssbc.github.io/scuttlebutt-protocol-guide/
[buttwoo]: https://github.com/ssbc/ssb-buttwoo-spec
[meta feeds]: https://github.com/ssbc/ssb-meta-feeds-spec
[bamboo]: https://github.com/AljoschaMeyer/bamboo/
[earthstar]: https://github.com/earthstar-project/
[bipf]: https://github.com/ssbc/bipf
[ssb-bfe encrypted data format]: https://github.com/ssbc/ssb-bfe-spec#5-encrypted-data-formats
[blake3]: https://github.com/BLAKE3-team/BLAKE3
[fusion identity]: https://github.com/ssbc/fusion-identity-spec
[SHS]: https://github.com/auditdrivencrypto/secret-handshake
