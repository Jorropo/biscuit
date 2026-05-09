# BISCUIT or BISCUITS (name pending)

TODO figure out what the acronym stands for.

***WIP*** expect breakages.

## What is it ?

BISCUIT is a schema based data interchange format.
You write a `.biscuit` file schema,
you then use the biscuit compiler to translate it into friendly structs, validation, decode and encode functions in C, C++, Go or Rust (*currently planned languages*),
finally you can use the encoded biscuit payloads on extremely bandwidth constrained payloads (database running an on MCU, sending packets of LoRa, ...).

It targets a more compact than protobuf encoding goal. We aim for zero to two bits of encoding overhead over the minimum bound.

## Want feature set:

- memory safe
- binary encoding
- binary decoding
- validation
- decoding is validation
- strings compression
- forward compatible
- LSP
- highlighting config
- ¿ compile `.biscuit` files to protobuf and add biscuit ←→ proto code generation ?

## What are the tradeoffs over protobuf ?

Theses tradeoff allows for biscuit to create smaller payloads:
- impossible to skip over unknown fields with the schema
- removed / deprecated fields forever cost ~1 bit.
  In protobuf you can remove a field from a schema, decoders will skip over it if they see it, and encoders wont emit it.
  The only cost you incur is the opportunity cost of using that field id.
  In biscuit fields always 
- history dependent encoding
  Each `.biscuit` file comes with a matching `.biscuit-history` file maintained by the biscuit compiler.
  Identical `.biscuit` files can have different binary encodings based on the order different fields were added.

## How does the binary format looks ?

The format is made of 3 nested data streams:
- byte datasteam
  - bit datastream
  - utf8 datastream *optional* used for compression

The top most level is made of a byte data stream, allocating more bytes just grows the size of the message.

We then expose a bit datastream, everytime a bit is needed we first look if any bits are free in the bit datastream, if so we use theses;
otherwise we allocate one byte from the byte datastream and use it to provide 8 bits to the bit stream, distributed from MSB then LSB.

## Can you show an example schema ?

*This is pre-alpha first draft:*
```go
const digiCodeLength = 6 // consts use perfect BIG number math and are evaluated at the schema's compile time

type UserRecord struct {
    Name alwaysPresent utf8(hardLimit(length, 1, 32)) // hardLimits can't generally be changed in a backward compatible way; they are leveraged for more compact encoding; here is the string's length is stored as 5 bits ranging from 0-31 (it does +1 and -1 for decoding and encoding).
    DigiCode optional ascii(hardLimit(length, digiCodeLength, digiCodeLength), softLimit(data, in "0123456789")) // softLimits are only applied in the validation layer (which also runs during encoding and decoding), they do not affect the encoding in anyway
    Age alwaysPresent varint(native=u32)
}
```

Here it is how it would be encoded:
```
llllld00 | <1 - 32 bytes for the name> | [6 bytes for the digicode] | <self sized 1-5 bytes for the varint>
lllll - length for the name encoded as length - min (thus length - 1)
     d - tells you if the 6 bytes for the digicode are present or not
      00 - unused bits usable by future bit sized types
```

# Elements:

# Limits

## `hardLimit`

A hardLimit can't generally be changed in a backward compatible maner.
Each `hardLimit` on a type is promoted to a `softLimit`.

`hardLimit` can then be leveraged by types to reduce their encoding size.

## `softLimit`

A softLimit only applies to the validation layer (which is also ran on encoding and decoding).

It can be changed as it has no impact on the encoding scheme.

# Types

## `boolean` type

Size: 1 bit

Encoding:
Consume a bit from the bit stream, if 0 false, if 1 true.

Schema: type `boolean`

Code Generation: language's own `bool` equivalent type

Validation: none

## `number` typeset

```go
type number = unsigned | signed | float
```

## `unsigned` & `signed` typeset & types

`unsigned` is a typeset of all the fixed size unsigned types.

`signed` is a typeset of all the fixed size unsigned types.

The types are `u1`, `u2`, `u3`, ... up to `u64`. Theses are two's complement unsigned numbers, the number indicate the number of bits.

Signed types are `i1`, `i2`, `i3`, ... up to `i64` and work in the same way.

Size: number of bits used in the two's complement representation.

Encoding:
*todo use zigzag encoding for ints ?*

First take the hard limit maximum value substract the hard limit minimum value,
if `bitlen(uint(hard.max) - uint(hard.min)) < bitlen(uint(hard.max)) && bitlen(uint(hard.max) - uint(hard.min)) < bitlen(uint(hard.min))` shift the value by `v - uint(hard.min)` (for decoding do `d + uint(hard.min)`) also set size to `size = bitlen(uint(hard.max) - uint(hard.min))`.

If `size % 8 == 0 && isPow2(size/8)` consume `size/8` bytes from the byte stream, then store the number in little endian form.
Otherwise consume size bits from the bitstream, store MSB first.

Schema: `u1`, `u2`, `u3`, ... up to `u64`, `i1`, `i2`, `i3`, ... up to `i64`, `signed` and `unsigned`

Code Generation: the smallest two's complement int that fits the required size.

Validation: default hard limit `0, (1<<size)-1`