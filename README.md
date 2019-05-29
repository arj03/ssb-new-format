This is WIP and not fully processed

# Feed

Signed hash-based linked list of messages - a sigchain

The overall goal is to seperate the transport format from the database
format. This document deals with the transport format.

A base message format that serves as a good starting point and a
minimum of useful fields. This is more or less
[bamboo](https://github.com/AljoschaMeyer/bamboo), except the version
field and the encoding is a bit different.

# Message format

 - version field
 - seqno
 - backlink
 - lipmaalink (log link)
 - hash af content
 - size in bytes
 - end-of-log bit
 
 - digital signature of all the previous data

 - content?

I'm a big fan of version fields on protocol which is why I added
it. Bamboo uses
[yamf-hash](https://github.com/AljoschaMeyer/yamf-hash) for some
future compatability without changing the format, but I would rather
keep the format simple. The version number is global consensus, so in
that regard I'm not sure if it fits the spirit of ssb.

I think we should just encode the above using cbor in the order the fields are specified.

While the above is a general format, the rest of the document will
focus on ssb related topics.

Considerations:
 - Backwards compatability:
   - I think we should use this format for new messages only. Old stay
     the same. There is the issue of linking the new with the old
     log. One could include a backlink to the last entry using the old
     format. Or start with a specific same-as content message that
     links these. Or one could specify a way to include the old
     messages in content, with the old hash. I'm learning towards the
     same-as option.
 - Message size restriction? Might be good to lift it from 16kb to
   something like 64kb?
 - Content could be included as current design or could be fetched via some other channel (blobs, dat etc.) for offchain-content. Currently learning towards including it from a latency perspective, but this is not a hard requirement.

# Content

Must include the following fields from the old format: timestamp,
author and type. Still schemaless. Encoded as [canonical
cbor](https://tools.ietf.org/html/rfc7049#section-3.9).

The reason I like canonical cbor is that it is a well spec'ed
standard, there are multiple implementations and at least the js
supports the canonical format directly.

Considerations:
 - Require a type seq that would specify the seq number for that
   particular type. This would allow one to fetch all messages of a
   specific type, say books, chess or abouts and verify that the
   server hasn't left some of them out. This combined with partial
   replication could open up for better onboarding and lighter
   clients.
 - FIXME: better format for private messages
 - Performance of canonical cbor seems really bad, need to look into this some more. More info [here](https://github.com/dignifiedquire/borc/issues/22#issuecomment-445550315)

## Performance

These are some notes on the current js implementation for 100k messages on my slow machine:

 - json stringify is 1.5s
 - validate is 60s
   - hash of message to key is roughly 10%
   - ensuring canonical (regex) is roughly 10%
   - libsodium verify is roughly 80%

So this basically tells us that it will be hard to get much faster for
stuff like initial replication. Even rust, go and c should be using
libsodium or something similar so they shouldn't be able to much
faster. Instead I think we should focus on things that can be used to
improve initial replication such a seq no on type and partial
replication.

## Links
 - existing SSB spec: https://spec.scuttlebutt.nz/
