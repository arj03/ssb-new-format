This is WIP and not fully processed

# Feed

Signed hash-based linked list of messages - a sigchain

The overall goal is to separate the transport and signing format from
the database format. This document deals with the signing and
transport format.

The following defines a general base message format that serves as the
starting point for adding SSB specific considerations and content on
top. The signing format is heavily inspired from
[bamboo](https://github.com/AljoschaMeyer/bamboo). The "end of log
entry should be a status field for a more open-ended system" idea was
taken from [birch](https://github.com/cn-uofbasel/ssb-birch).

# Message signing format

 - backlink
 - lipmaalink (log link)
 - hash af content
 - status enum
 
The digital signature of the above fields will be hmac'ed with the
signing cap just as in the current system, to make sure that messages
from one network are never valid in another.

The id (or cipherlink) of the message is hash of the above fields.

# Transport format

 - message signing format fields
 - signature

 - version number
 - seqno
 - author
 - size of encoded content in bytes

 - content
 - attachments

I'm a big fan of version fields on protocol which is why I added
it. Version number indicate what to expect and makes it easier to
reason about. A specific version number should have a document that
describes exactly what to expect.

Seqno makes it much easier to reason about what is being sent, so
while not strictly not necessary, from a practical perspective I like
them and think we should keep them.

Content should be included by default, while the attachments can be
included at the clients request. It might be the case that the content
is not included because the sending peer has removed the content for
various reasons. There should be a mechanism similar to blobs for
fetching content or attachments directly.

As for encoding of the above, besides just encoding it in the order
specified, I think the biggest concern is chosing and encoding system
that can be used for the content as well, so we only have one system
to implement.

The hash functions should be encoded in a way that allows us to
upgrade them either by an extension as currently used or using
something like
[yamf-hash](https://github.com/AljoschaMeyer/yamf-hash).

While we are at it, we should support multiple signing key types.

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
 - Multiple signing key types:
   %tr82vmrplCxXQNxQsl3vbIzg+EjT0ZtT0jZMF5ZiHqM=.sha256

# Content

Should include the following fields from the old format: timestamp and
type. Still schemaless.

The content should be encoded in a canonical format. With the same
encoding as the overall message format. I don't think the format needs
very much more than what json support. Most importantly it must be
well spec'ed. Preferably with multiple existing implementations.

Define a better way of specifying private messages instead of it
being just a special string.

Considerations:
 - Require a seqno per type. This would allow one to fetch all
   messages of a specific type, say books, chess or abouts and verify
   that the server hasn't left some of them out. This combined with
   partial replication could open up for better onboarding and lighter
   clients. The idea of namespaces instead are starting to grow on me.

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

Performance of [canonical
cbor](https://tools.ietf.org/html/rfc7049#section-3.9) seems really
bad. More info
[here](https://github.com/dignifiedquire/borc/issues/22#issuecomment-445550315).

[Matrix](https://matrix.org/docs/spec/appendices.html#canonical-json) also uses "canonical" json. Relevant [discussion](https://github.com/matrix-org/matrix-doc/issues/1013).

## Links
 - existing SSB spec: https://spec.scuttlebutt.nz/
 - compact legacy message representation %043+a3lLbXxJM0L7QDJ3rAKTb6sWXuAkRuhS/wOT+hQ=.sha256
 - hsdt Hypothetical SSB Data Format) (cbor variant) %25BTnweXwHe134te24iGMOM%2Fn%2BQK6V429bLDVcUxnjI3E%3D.sha256
 - Argument against cannonical formats for signed data  %25FI3kBXdFD6j%2BuISXluhirJm70QZpJIYzc65OX1XPJ5c%3D.sha256
 - bamboo discussion %9LKTBkkOSfGuGkTbSTRBVVPrznIrr61qtLeLWLGclXI=.sha256
