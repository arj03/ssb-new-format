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
 - author
 
The digital signature of the above fields will be hmac'ed with the
signing cap just as in the current system, to make sure that messages
from one network are never valid in another.

The id (or cipherlink) of the message is hash of the above fields.

# Transport format

 - message signing format fields
 - signature

 - version number
 - seqno
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

Rejected considerations:
 - Require a seqno per type. This would allow one to fetch all
   messages of a specific type, say books, chess or abouts and verify
   that the server hasn't left some of them out. This combined with
   partial replication could open up for better onboarding and lighter
   clients. 
   
   A different way to implement the same thing as above is to
   introduce namespaces at the transport level. These work as
   subfeeds as a way to group mesages into collections, such as
   gatherings and their edits or books. It could also be used for
   having a subfeed for personal stuff, work stuff etc. The reason
   to add it at this level, is that it would allow a way for the
   replication to specify that they are only interested in a subset
   of all messages. And it would be 100% backwords compatible by
   just not using the field.
   
   The reason I'm putting this into the rejected ideas pile is that
   I don't think we should put these concerns into the protocol
   level. Similar to how #channels work and how tags are a much more
   general concept of the same idea, we should strive to keep the
   core as minimal as possible.
   
   See replication section for more info.

# Replication / RPC

Current JS parts:

- ssb-ebt: replicate multiple feeds
- Legacy replication (ssb-replicate) createHistoryStream per feed
- ssb-ooo request any message by hash id. These are not indexed!
- blobs: get (max size option), createWants

Related: [gossip-query](https://github.com/dominictarr/gossip-query)

On initial sync (see also performance section):

@Piet: This (replication limiter) module does what it's intended to
do, but it turns out the bottleneck is not actually in replication as
we originally thought
(%HJTzjmTcr8FsKt34wmvN5ett4imneEzHp7dCab2rGpg=.sha256). See also
"solving initial sync"
(%3hli4+jOqRZ1EE6EckvVh74DIda+hRzabTfvHecxfqU=.sha256).

## Performance

These are some notes on the current js implementation for 100k
messages on my slow machine:

 - bench-ssb json stringify ~1.9s, parse ~1.9s
 - more realistic json data: ~2.2s, parse ~2.1s
 - validate is 60s, 1.66 msg / ms
   - hash of message to key ~10%
   - ensuring canonical (regex) ~10%
   - string to buffer conversion and back around libsodium ~8%
   - libsodium verify ~72%

So this basically tells us that it will be hard to get much faster for
stuff like initial replication. Even rust, go and c should be using
libsodium or something similar so they shouldn't be able to much
faster. Instead I think we should focus on things that can be used to
improve initial replication.

A smaller signed format (see message signing format) doesn't seem to
make much improvements in the validation speed.

Wasm version compared c version of libsodium is almost the same
speedon my laptop, around 7% slower.

Performance on phone compared to laptop is that phone is 64% slower in
validate, but json performance is roughly the same.

There is a way to [not validate
everything](https://github.com/ssbc/ssb-validate#shortcuts-for-js-mode-api).

Browser (ws connection to sbot localhost):

Very high variance, numbers are avg of 3 runs

1 feed (6500 msgs)

- createHistoryStream ~ 16 s
- createHistoryStream + validate only end ~ 17.6 s
- createHistoryStream + validate all ~ 22.4 s

## Other

JS [Benchmark](https://github.com/ssbc/bench-ssb/tree/test-encodings) on i7: json 0.38, cbor 1.76, bipf 1.14

- [cbor](https://github.com/dignifiedquire/borc) non-canonical is roughly 5 times slower than json stringify
- [bipf](https://github.com/dominictarr/bipf) is roughly 3 times slower than json stringify

Performance of [canonical
cbor](https://tools.ietf.org/html/rfc7049#section-3.9) seems really
bad. More info
[here](https://github.com/dignifiedquire/borc/issues/22#issuecomment-445550315). Related to ["canonical"](https://github.com/pyfisch/cbor/issues/17) and a revised spec.

[Matrix](https://matrix.org/docs/spec/appendices.html#canonical-json) also uses "canonical" json. Relevant [discussion](https://github.com/matrix-org/matrix-doc/issues/1013).

### Message types

My database has the following top message (roughly 875k) types:

| Type       | Count  |
| ----       | ----   |
| vote       | 269447 |
| post       | 173264 |
| contact    | 167326 |
| private    | 93208  |
| channel    | 57087  |
| about      | 33476  |
| chess_move | 24199  |
| pub        | 17646  |
| image/jpeg | 10681  |
| git-update | 8915   |
| image/png  | 5809   |

456k messages within hops=1 (370 feeds, 1.2k on average, longest 25k)

409k messages within hops=1 and post within last 3 months (207 feeds, 1.98k on average, longest 25k)

Streaming (and filtering) locally by type using createHistStream:
9052.317ms, 16346 msgs = 1.8 msg/ms. Will be slightly faster with
flumelog-aligned-offset. Noauth is roughly 2.8 msg/ms.

Direct streaming of the log is 17.2 msg/ms.

The main reason this is slow is because it uses ssb client to connect
to the sbot server. Running the same functions locally is as fast as
streaming.

Filtering over network 450k message would be at least 250 seconds.
Validating the same messages would be 270 seconds.

----

Feed type is signature algorithm + signing format and validation rules?
Outer format is still json?

%LrMcs9tqgOMLPGv6mN5Z7YYxRQ8qn0JRhVi++OyOvQo=.sha256

## Links
 - existing SSB spec: https://spec.scuttlebutt.nz/
 - compact legacy message representation %043+a3lLbXxJM0L7QDJ3rAKTb6sWXuAkRuhS/wOT+hQ=.sha256
 - hsdt Hypothetical SSB Data Format) (cbor variant) %25BTnweXwHe134te24iGMOM%2Fn%2BQK6V429bLDVcUxnjI3E%3D.sha256
 - Argument against cannonical formats for signed data  %25FI3kBXdFD6j%2BuISXluhirJm70QZpJIYzc65OX1XPJ5c%3D.sha256
 - bamboo discussion %9LKTBkkOSfGuGkTbSTRBVVPrznIrr61qtLeLWLGclXI=.sha256
 - lipmaalinks %3W2iK1uLh29DQw7q9+vR2yrw5HmMHXJNXrZv1BQ0z/k=.sha256
 - redesign notes %N+mBPUgxLyYM9Ab3EXDSebR5idQ+M/xjopt5/ZOGZUY=.sha256
 - author field in signed data %1AsqTRxdVrbfypC69W7uWbMClQteNNnnl3ohzbpu3Xw=.sha256
 - cbor part of webauth spec: https://w3c.github.io/webauthn/#dom-authenticatorattestationresponse-attestationobject
