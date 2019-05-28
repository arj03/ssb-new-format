# Feed

Signed hash-based linked list of messages - a sigchain

Base format that serves as a good starting point and a minimum of useful fields. This is more or less bamboo, except the version field and the encoding is a bit different.

Allows for partial replicating using lipmaalinks.

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

I'm a big fan of version fields on protocol which is why I added it. Bamboo uses [yamf-hash](https://github.com/AljoschaMeyer/yamf-hash) for some forward security, but I would rather keep the format simple. The version number is global consensus, so in that regard I'm not sure if it fits the spirit of ssb.

I think we should just encode the above using [canonical cbor](https://tools.ietf.org/html/rfc7049#section-3.9).

Considerations:
 - Backwards compatability:
   - I think we should use this format for new messages. If one includes a backlink to the last entry using the old format, one can still see that they are from the same author.

## Links
 - SSB spec: https://spec.scuttlebutt.nz/
 - Bamboo: https://github.com/AljoschaMeyer/bamboo

