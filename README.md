This is WIP and not fully processed

# Feed

Signed hash-based linked list of messages - a sigchain

The overall goal is to seperate the transport or signing format from
the database format. This document deals with the signing format.

The following defines a general base message format that serves as the
starting point for adding SSB specific considerations and content on
top. This is more or less
[bamboo](https://github.com/AljoschaMeyer/bamboo), except the version
field and the encoding.

# Message format

 - version
 - seqno
 - backlink
 - lipmaalink (log link)
 - hash af content
 - size in bytes
 - end-of-log bit
 
 - digital signature of all the previous data

 - content?

I'm a big fan of version fields on protocol which is why I added
it. Version number indicate what to expect and makes it easier to
reason about. The version number should have a document that describes
exactly what to expect.

I think we should encode the above using cbor in the order the fields
are specified. Taking care to encode things such as hashes that can
change in a way that allows us to upgrade them either by an extension
as currently used or using something like
[yamf-hash](https://github.com/AljoschaMeyer/yamf-hash).

SSB related Considerations:
 - Backwards compatibility: I think we should use this format for new
   messages only. Old stay the same. There is the issue of linking the
   new with the old log. One could include a backlink to the last
   entry using the old format. Or start with a specific same-as
   content message that links these. Or one could specify a way to
   include the old messages in content, with the old hash. I'm
   learning towards the same-as option.
 - Message size restriction? Might be good to lift it from 16kb to
   something like 64kb?
 - Content could be included as current design or could be fetched via
   some other channel (blobs, dat etc.) for
   offchain-content. Currently learning towards including it from a
   latency perspective, but this is not a hard requirement.

# Content

Must include the following fields from the old format: timestamp,
author and type. Still schemaless. 

The content should be encoded in a canonical format. With the same
encoding as the overall message format. I don't think the format needs
very much more than what json support. It just to be well
spec'ed. Preferably with multiple implementations already. This could
be either cbor, bipf or something else that.

While we are at it, we should support multiple signing types.

Also define a better way of specifying private messages instead of it
being just a special string.

Considerations:
 - Require a type seq that would specify the seq number for that
   particular type. This would allow one to fetch all messages of a
   specific type, say books, chess or abouts and verify that the
   server hasn't left some of them out. This combined with partial
   replication could open up for better onboarding and lighter
   clients.
 - Performance of [canonical
cbor](https://tools.ietf.org/html/rfc7049#section-3.9) seems really
bad, need to look into this some more. More info
[here](https://github.com/dignifiedquire/borc/issues/22#issuecomment-445550315). Might
also be possible to go for [bipf](https://github.com/dominictarr/bipf)
instead of cbor, but I would rather use something with multiple
implementations already for the exchange format.
 - Multiple signing key types: %tr82vmrplCxXQNxQsl3vbIzg+EjT0ZtT0jZMF5ZiHqM=.sha256

## Performance

These are some notes on the current js implementation for 100k
messages on my slow machine:

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

## Other

JS [Benchmark](https://github.com/ssbc/bench-ssb/tree/test-encodings) on i7: json 0.38, cbor 1.76, bipf 1.14

- [cbor](https://github.com/dignifiedquire/borc) non-canonical is roughly 5 times slower than json stringify
- [bipf](https://github.com/dominictarr/bipf) is roughly 3 times slower than json stringify

## Links
 - existing SSB spec: https://spec.scuttlebutt.nz/
 - compact legacy message representation %043+a3lLbXxJM0L7QDJ3rAKTb6sWXuAkRuhS/wOT+hQ=.sha256
 - hsdt Hypothetical SSB Data Format) (cbor variant) %25BTnweXwHe134te24iGMOM%2Fn%2BQK6V429bLDVcUxnjI3E%3D.sha256
 - Argument against cannonical formats for signed data  %25FI3kBXdFD6j%2BuISXluhirJm70QZpJIYzc65OX1XPJ5c%3D.sha256
 - bamboo discussion %9LKTBkkOSfGuGkTbSTRBVVPrznIrr61qtLeLWLGclXI=.sha256
