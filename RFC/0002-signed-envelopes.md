# RFC 0002 - Signed Envelopes

- Start Date: 2019-10-21
- Related RFC: [0003 Address Records][addr-records-rfc]

## Abstract

This RFC proposes a "signed envelope" structure that contains an arbitray byte
string payload, a signature of the payload, and the public key that can be used
to verify the signature.

This was spun out of an earlier draft of the [address records
RFC][addr-records-rfc], since it's generically useful.

## Problem Statement

Sometimes we'd like to store some data in a public location (e.g. a DHT, etc),
or make use of potentially untrustworthy intermediaries to relay information. It
would be nice to have an all-purpose data container that includes a signature of
the data, so we can verify that the data came from a specific peer and that it hasn't
been tampered with.

## Domain Separation

Signatures can be used for a variety of purposes, and a signature made for a
specific purpose MUST NOT be considered valid for a different purpose.

Without this property, an attacker could convince a peer to sign a payload in
one context and present it as valid in another, for example, presenting a signed
address record as a pubsub message.

We separate signatures into "domains" by prefixing the data to be signed with a
string unique to each domain. This string is not contained within the payload or
the outer envelope structure. Instead, each libp2p subsystem that makes use of
signed envelopes will provide their own domain string when constructing the
envelope, and again when validating the envelope. If the domain string used to
validate is different from the one used to sign, the signature validation will
fail.

Domain strings may be any valid UTF-8 string, but should be fairly short and
descriptive of their use case, for example `"libp2p-routing-record"`.

## Type Hinting

The envelope record can contain an arbitrary byte string payload, which will
need to be interpreted in the context of a specific use case. To assist in
"hydrating" the payload into an appropriate domain object, we include a "type
hint" field. The type hint consists of a [multicodec][multicodec] code,
optionally followed by an arbitrary byte sequence.

This allows very compact type hints that contain just a multicodec, as well as
"path" multicodecs of the form `/some/thing`, using the ["namespace"
multicodec](https://github.com/multiformats/multicodec/blob/master/table.csv#L23),
whose binary value is equivalent to the UTF-8 `/` character.

## Wire Format

Since we already have a [protobuf definition for public keys][peer-id-spec], we
can use protobuf for this as well and easily embed the key in the envelope:


```protobuf
message SignedEnvelope {
  PublicKey publicKey = 1; // see peer id spec for definition
  bytes typeHint = 2;      // type hint
  bytes contents = 3;      // payload
  bytes signature = 4;     // see below for signing rules
}
```

The `publicKey` field contains the public key whose secret counterpart was used
to sign the message. This MUST be consistent with the peer id of the signing
peer, as the recipient will derive the peer id of the signer from this key.

The `typeHint` field contains a [multicodec][multicodec]-prefixed type hint as
described in the [Type Hinting section](#type-hinting).

The `contents` field contains the arbitrary byte string payload.

The `signature` field contains a signature of all fields except `publicKey`,
generated as described below.

## Signature Production / Verification

When signing, a peer will prepare a buffer by concatenating the following:

- The length of the [domain separation string](#domain-separation) string in
  bytes, encoded as an [unsigned varint][uvarint]
- The domain separation string, encoded as UTF-8
- The length of the `typeHint` field in bytes, encoded as an [unsigned
  varint][uvarint]
- The value of the `typeHint` field
- The length of the `contents` field in bytes, encoded as an [unsigned
  varint][uvarint]
- The value of the `contents` field

Then they will sign the buffer according to the rules in the [peer id
spec][peer-id-spec] and set the `signature` field accordingly.

To verify, a peer will "inflate" the `publicKey` into a domain object that can
verify signatures, prepare a buffer as above and verify the `signature` field
against it.

[addr-records-rfc]: ./0003-address-records.md
[peer-id-spec]: ../peer-ids/peer-ids.md
[multicodec]: https://github.com/multiformats/multicodec
[uvarint]: https://github.com/multiformats/unsigned-varint