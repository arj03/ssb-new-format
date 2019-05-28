# Feed

Signed hash-based linked list of messages - a sigchain

The overall goal is to seperate the transport format from the database
format. This document deals with the transport format.

A base message format that serves as a good starting point and a
minimum of useful fields. This is more or less
[bamboo](https://github.com/AljoschaMeyer/bamboo), except the version
field and the encoding is a bit different.

# Message format

## Format
 - version field
 - seqno
 - backlink
 - lipmaalink (log link)
 - hash af content
 - size in bytes
 - end-of-log?
 
 - digital signature of all the previous data, issued with the log's public key

 - content?

I'm a big fan of version fields on protocol which is why I added
it. Bamboo uses
[yamf-hash](https://github.com/AljoschaMeyer/yamf-hash) for some
future compatability without changing the format, but I would rather
keep the format simple. The version number is global consensus, so in
that regard I'm not sure if it fits the spirit of ssb.

I think we should just encode the above using [canonical
cbor](https://tools.ietf.org/html/rfc7049#section-3.9).

While the above is a general format, the rest of the document will
focus on ssb related topics.

Considerations:
 - Backwards compatability:
   - I think we should use this format for new messages only. Old stay
     the same. If one includes a backlink to the last entry using the
     old format, one can still see that they are from the same
     author. This will not allow for partial replication of old
     messages. One could specify a way to include the old messages in
     content, with the old hash, but I don't think we need this.
 - Message size restriction? Might be good to lift it from 16kb to
   something like 64kb?

# Content

Must include the following fields from the old format: timestamp,
author and type. Still schemaless. Encoding is also [canonical
cbor](https://tools.ietf.org/html/rfc7049#section-3.9).

The reason I like canonical cbor is that it is a well specced
standard, there are multiple implementations and at least the js
supports the canonical format directly.

Considerations:
 - Require a type seq that would specify the seq number for that
   particular type. This would allow one to fetch all messages of a
   specific type, say books, chess or abouts and verify that the
   server hasn't left some of them out. This combined with partial
   replication makes up for some interesting cases.

## Performance

These are some notes on the current js implementation for 100k messages:

 - json stringify is 1.5s
 - validate is 60s
   - hash of message to key is roughly 10%
   - ensuring canonical (regex) is roughly 10%
   - libsodium verify is 80%

So this basically tells us that it will be hard to get much faster for
stuff like initial replication. Even rust and go should be using
libsodium or something similar so they shouldn't be able to much
faster. Instead I think we should focus on things that can be used to
improve initial replication such a seq no on type and partial
replication.

## Links
 - SSB spec: https://spec.scuttlebutt.nz/

