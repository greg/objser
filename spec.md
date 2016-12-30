# Object Graph Serialisation (ObjSer)

*This specification is incomplete.*

## Introduction

ObjSer is a binary data-interchange format. It differs from formats such as [JSON](http://json.org) and [MessagePack](http://msgpack.org) in that it is able to represent *arbitrary* object graphs, including those containing cycles or multiple instances of the same object, without duplication.

## Design goals

- **Small**: a maximally compact binary storage format
- **Portable**: platform and language-independent
- **Fast**: performant (de)serialisation

## Encodable types

The following primitive types are encodable by ObjSer:

- **Reference**: reference to another primitive, to avoid encoding the same object twice
- **Integer**: [-2<sup>63</sup>, 2<sup>64</sup> - 1]
- **Nil**
- **Boolean**: true or false
- **Float**: IEEE 754 single (32-bit) or double (64-bit) precision floating-point number
- **String**: null-terminated UTF-8 string
- **Data**: an opaque sequence of bytes of maximum length 2<sup>32</sup> - 1
- **Array**: an ordered collection of primitives
- **Map**: a key-value (ordered) collection of primitives
- **Typed Primitives**: type information (integer in [0, 2<sup>32</sup> - 1]) can be encoded for any primitive, with space optimisations for typed arrays and maps

Objects must be represented as one of these primitives for encoding.

Throughout this specification, the term *primitive* refers to any of the above.

## Diagram notation key

	  1 byte      bytes     primitive  primitives
	 --------    ~~~~~~~~    =======    ≈≈≈≈≈≈≈≈
	|        |  |        |  │       │  |        |
	 --------    ~~~~~~~~    =======    ≈≈≈≈≈≈≈≈

## File structure

A file consists of a consecutive sequence of primitives, which are *indexed* by unsigned integers ascending consecutively from 0, ending with the root primitive. The sequence of primitives ends at the end of the file or stream. Array, map, and typed primitives may contain other primitives; these are not indexed.

	 ======== ======== ======== ≈≈≈≈≈≈≈≈ ======== 
	│  id 0  │  id 1  │  id 2  |        |  root  │  EOF
	 ======== ======== ======== ≈≈≈≈≈≈≈≈ ======== 

## Formats

Each primitive is encoded in one of the supported *formats*.

Characters `x` appearing in binary numbers represent a single variable bit.

### Overview

format   | first byte | hex
-------- | ---------- | ---
ref6     | `00xxxxxx` | `0x00` – `0x3f`
ref8     | `01000000` | `0x40`
farray   | `010xxxxx` | `0x41` – `0x5f`
ref16    | `01100000` | `0x60`
fstring  | `0110xxxx` | `0x61` – `0x6f`
ref32    | `01110000` | `0x70`
fdata    | `0111xxxx` | `0x71` – `0x7f`
+int6    | `10xxxxxx` | `0x80` – `0xbf`
false    | `11000000` | `0xc0`
true     | `11000001` | `0xc1`
int8     | `11000010` | `0xc2`
int16    | `11000011` | `0xc3`
int32    | `11000100` | `0xc4`
int64    | `11000101` | `0xc5`
uint8    | `11000110` | `0xc6`
uint16   | `11000111` | `0xc7`
uint32   | `11001000` | `0xc8`
uint64   | `11001001` | `0xc9`
float32  | `11001010` | `0xca`
float64  | `11001011` | `0xcb`
map      | `11001100` | `0xcc`
varray   | `11001101` | `0xcd`
vstring  | `11001110` | `0xce`
sentinel | `11001111` | `0xcf`
nil      | `11010000` | `0xd0`
vdata8   | `11010001` | `0xd1`
vdata16  | `11010010` | `0xd2`
vdata32  | `11010011` | `0xd3`
typed8   | `11010100` | `0xd4`
typed16  | `11010101` | `0xd5`
typed32  | `11010110` | `0xd6`
typedv8  | `11010111` | `0xd7`
typedv16 | `11011000` | `0xd8`
typedv32 | `11011001` | `0xd9`
typedm8  | `11011010` | `0xda`
typedm16 | `11011011` | `0xdb`
typedm32 | `11011100` | `0xdc`
reserved | `110111xx` | `0xdd` – `0xdf`
-int5    | `111xxxxx` | `0xe0` – `0xff`

### References

Reference formats encode the unsigned integer index of an indexed primitive.

As an optimisation, implementations should sort primitives in descending order of reference frequency, so frequently-referenced primitives have small integer indices that fit in smaller formats.

	ref6 [0, 63]    ref8 [0, 255]
	 ---------    -------- --------
	|00xxxxxxx|  |  0x40  |xxxxxxxx|
	 ---------    -------- --------

	     ref16 [0, 65 535]
     -------- -------- --------
    |  0x60  |xxxxxxxx|xxxxxxxx|
	 -------- -------- --------

	           ref32 [0, 4 294 967 295]
	 -------- -------- -------- -------- --------
    |  0x70  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
     -------- -------- -------- -------- --------

### Integers

Formats beginning with `int` are signed integers in [two's complement representation](https://en.wikipedia.org/wiki/Two's_complement). Formats beginning with `uint` are unsigned integers starting at 0.

	+int6: unsigned (clear the first bit to obtain the value)
	 --------
	|10xxxxxx|
	 --------

	-int5: treat as 8-bit signed integer to obtain correct (negative) value
	 --------
	|111xxxxx|
	 --------

	       int8                 uint8
	 -------- --------    -------- -------- 
	|  0xc2  |xxxxxxxx|  |  0xc6  |xxxxxxxx|
	 -------- --------    -------- -------- 

	           int16                         uint16
	 -------- -------- --------    -------- -------- -------- 
	|  0xc3  |xxxxxxxx|xxxxxxxx|  |  0xc7  |xxxxxxxx|xxxxxxxx|
	 -------- -------- --------    -------- -------- -------- 

	                    int32
	 -------- -------- -------- -------- -------- 
	|  0xc4  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- 

	                    uint32
	 -------- -------- -------- -------- -------- 
	|  0xc8  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- 

	                                      int64
	 -------- -------- -------- -------- -------- -------- -------- -------- --------
	|  0xc5  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- -------- -------- -------- --------

	                                      uint64
	 -------- -------- -------- -------- -------- -------- -------- -------- --------
	|  0xc9  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- -------- -------- -------- --------

### Boolean

	  false        true
	 --------    --------
	|  0xc0  |  |  0xc1  |
	 --------    --------

### Nil

	   nil
	 --------
	|  0xd0  |
	 --------

### Float

Single-precision floating-point numbers are encoded in the IEEE 754 [binary32](https://en.wikipedia.org/wiki/Single-precision_floating-point_format) format, and double-precision floating point-numbers are encoded in the IEEE 754 [binary64](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) format.

	                   float32
	 -------- -------- -------- -------- -------- 
	|  0xca  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- 

	                                     float64
	 -------- -------- -------- -------- -------- -------- -------- -------- --------
	|  0xcb  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- -------- -------- -------- --------

### String

	fstring: fixed-length string (no null terminator)
	 -------- ~~~~~~~~~~~~~~
	|0110xxxx|  characters  |
	 -------- ~~~~~~~~~~~~~~

The 4-bit unsigned integer denoted `xxxx` gives the length of the string in bytes, 1–15. The first byte must be followed by exactly this many bytes of the string.

	 vstring: null-terminated string
	 -------- ~~~~~~~~~~~~~~ --------
	|  0xce  |  characters  |  0x00  |
	 -------- ~~~~~~~~~~~~~~ --------

**Note**: zero-length strings *cannot* be encoded in the `fstring` format, as this would conflict with the `ref16` format. Instead, encode `nil`, or an immediately null-terminated `vstring` if type information is required.

### Data

	fdata: fixed-length data
	 -------- ~~~~~~~~~
	|0111xxxx|  bytes  |
	 -------- ~~~~~~~~~

The 4-bit unsigned integer `xxxx` in the first byte gives the length of the data. The first byte must be followed by exactly this many bytes of the data.

	vdata8: data of 8-bit length       vdata16: data of 16-bit length
	 -------- -------- ~~~~~~~~~    -------- -------- -------- ~~~~~~~~~
	|  0xd1  |xxxxxxxx|  bytes  |  |  0xd2  |xxxxxxxx|xxxxxxxx|  bytes  |
	 -------- -------- ~~~~~~~~~    -------- -------- -------- ~~~~~~~~~

	             vdata32: data of 32-bit length
	 -------- -------- -------- -------- -------- ~~~~~~~~~
	|  0xd3  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|  bytes  |
	 -------- -------- -------- -------- -------- ~~~~~~~~~

The unsigned integer that follows the first byte gives the length of the data, and must be followed by exactly this many bytes of the data.

**Note**: zero-length data *cannot* be encoded in the `fdata` format, as this would conflict with the `ref32` format. Instead, encode `nil`, or `vdata8` of length 0 if type information is required.

### Array

	farray: fixed-length array
	 -------- ≈≈≈≈≈≈≈≈≈≈≈≈
	|010xxxxx| primitives |
	 -------- ≈≈≈≈≈≈≈≈≈≈≈≈

	varray: sentinel-terminated array
	 -------- ≈≈≈≈≈≈≈≈≈≈≈≈ --------
	|  0xcd  | primitives |  0xcf  |
	 -------- ≈≈≈≈≈≈≈≈≈≈≈≈ --------

**Note**: zero-length arrays *cannot* be encoded in the `farray` format, as this would conflict with the `ref8` format. Instead, encode `nil`, or an immediately null-terminated `varray` if type information is required.

### Map

	map: array-backed map
	 -------- =========
	|  0xcc  |  array  |
	 -------- =========

The array that follows the map byte contains alternating keys and values, in the form *key0, val0, key1, val1, ...*. `array` is `farray`, `varray`, or `nil`. If type information is not required, store `nil` instead of an empty map.

### Typed primitives

A typed primitive associates type information with a primitive.

#### Single primitives

The unsigned integer that follows the first byte gives the type information for the primitive that follows.

    typed8: primitive with 8-bit type  typed16: primitive with 16-bit type
     -------- -------- ===========    -------- -------- -------- ===========
    |  0xd4  |xxxxxxxx| primitive |  |  0xd5  |xxxxxxxx|xxxxxxxx| primitive |
     -------- -------- ===========    -------- -------- -------- ===========

               typed32: primitive with 32-bit type
     -------- -------- -------- -------- -------- ===========
    |  0xd6  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx| primitive |
     -------- -------- -------- -------- -------- ===========

#### Arrays

If an array contains primitives all of the same type, it is a waste of space to individually identify each one. The array type identifier byte is used to specify the type of all primitives in the array:

    typedv8: array with 8-bit primitive type  typedv16: array with 16-bit primitive type
     -------- -------- =======               -------- -------- -------- =======
    |  0xd7  |xxxxxxxx| array |             |  0xd8  |xxxxxxxx|xxxxxxxx| array |
     -------- -------- =======               -------- -------- -------- =======

          typedv32: array with 32-bit primitive type
     -------- -------- -------- -------- -------- =======
    |  0xd9  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx| array |
     -------- -------- -------- -------- -------- =======

where `array` is a `farray`, `varray`, or `nil`.

#### Maps

To encode a map where all keys and values are of the same type, simply identify the underlying array as described previously:

        map with 8-bit content type               map with 16-bit content type
     -------- -------- -------- =======    -------- -------- -------- -------- =======
    |  0xcc  |  0xd7  |xxxxxxxx| array |  |  0xcc  |  0xd8  |xxxxxxxx|xxxxxxxx| array |
     -------- -------- -------- =======    -------- -------- -------- -------- =======

                     map with 32-bit content type
     -------- -------- -------- -------- -------- -------- =======
    |  0xcc  |  0xd9  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx| array |
     -------- -------- -------- -------- -------- -------- =======

where `array` is one of `farray`, `varray`, or `nil`.

To encode a map where all values are of the same type, but there is no information about the keys, place the `typedv` byte before the map:

    map with 8-bit value type      map with 16-bit value type
     -------- -------- =====    -------- -------- -------- =====
    |  0xd7  |xxxxxxxx| map |  |  0xd8  |xxxxxxxx|xxxxxxxx| map |
     -------- -------- =====    -------- -------- -------- =====
     
                 map with 32-bit value type
     -------- -------- -------- -------- -------- ===== 
    |  0xd9  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx| map |
     -------- -------- -------- -------- -------- ===== 

The `typedm` formats encode a map where all keys are of the same type:

    typedm8: map with 8-bit key type  typedm16: map with 16-bit key type
     -------- -------- =====           -------- -------- -------- =====
    |  0xda  |xxxxxxxx| map |         |  0xdb  |xxxxxxxx|xxxxxxxx| map |
     -------- -------- =====           -------- -------- -------- =====

            typedm32: map with 32-bit value type
     -------- -------- -------- -------- -------- ===== 
    |  0xdc  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx| map |
     -------- -------- -------- -------- -------- ===== 

where `map` is a `map`, `nil`, or a map with a specified value type. This makes it possible to specify separate key and value types for a map.

**Note:** in the case of an empty map with a specified value type but *no* key type, `nil` cannot be stored for `map`, as this would conflict with an empty typed array.

## Implementation guidelines

- All multi-byte integers and floating-point numbers are encoded with **little-endian** byte order.

### Encoding requirements

- For each primitive, if there are multiple formats that can exactly represent its value, the one with the shortest length in bytes must be used.

- Primitives must be indexed if and only if they are referenced more than once.

- Exactly one root (last) primitive is encoded, and it is obviously exempt from the previous requirement.

### Decoding

- When possible, implementations should try to recover from minor errors in the input (e.g. a map's array containing an odd number of elements). All such errors must be reported in as much detail as practical.

- The byte values `0xdd`, `0xde`, and `0xdf` are reserved for future use, and should not appear as the first byte of any primitive. If an implementation unexpectedly encounters these bytes where a primitive should begin, decoding should fail with an error indicating that either the input is not valid or the implementation is out of date.

- The root (last) primitive is the only one returned directly to the user of the decoder. If there are indexed primitives that are not reachable by traversing the graph from the root primitive, the input is invalid. If the implementation chooses to return the partial decoding, it must make clear that either the root primitive was absent or the encoder was non-compliant.

### Limitations

- A single output file contains at most 2<sup>32</sup> indexed primitives. Hence, a serialisable object graph contains at most that many objects that are referenced by more than one object. Serialisation of an object graph that does not meet this requirement should fail.

- If required by language or platform limitations, implementations may impose the following limitations:
    - Integer bounds may be restricted up to [0, 2<sup>32</sup> - 1] for unsigned integers and [-2<sup>32</sup>, 2<sup>32</sup> - 1] for signed integers; integers outside of decodable bounds must be decoded as a double-precision floating-point number, or single-precision if it provides the exact value or double-precision floating-point numbers are not supported
    - If double-precision floating-point numbers are not supported, they must be truncated to single-precision on decoding
    - String and array primitives may be restricted to a maximum length of at least 2<sup>32</sup> - 1 characters or constrained primitives, respectively
    - Map primitives may be restricted to a maximum length of at least 2<sup>31</sup> - 1 key-value pairs

    If implementation-imposed limitations cause a loss of precision, the implementation must notify the user of the decoder. If implementation-imposed limitations cause a failure to completely decode a string, array, or map, the entire decoding process must fail with a descriptive error.

