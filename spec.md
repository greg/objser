# AOGF

*This specification is currently in early stages of development and may contain errors or inconsistencies. Breaking changes may be made at any time.*

## Introduction

AOGF (Archived Object Graph Format) is a binary data-interchange format. It differs from formats such as JSON and MessagePack in that it is able to represent *arbitrary* object graphs, including those containing cycles or multiple instances of the same object, without duplication.

## Design goals

- **Small**: a maximally compact binary storage format
- **Portable**: easy to implement and platform-independent
- **Fast**: performant archiving and unarchiving operations

## Types

The following types are supported by AOGF:

- **Integer**: unsigned [0, 2<sup>64</sup> - 1], signed [-2<sup>63</sup>, 2<sup>63</sup> - 1]
- **Nil**
- **Boolean**: true or false
- **Float**: IEEE 754 single (32-bit) or double (64-bit) precision floating-point number
- **String**: null-terminated UTF-8 string
- **Data**: an opaque sequence of bytes
- **Pair**: stores two values, each of any AOGF type; useful for tagging custom application data types, or creating a map using an array of tuples
- **Array**: an ordered collection of objects
- **Map**: a key-value collection of objects (essentially an array of pairs, but handled by implementations to store and create native map/dictionary objects)

Objects of other types must be converted to one of the above listed types for archiving.

## Limitations

- An object graph must contain at most 2<sup>32</sup> - 1 (see references format section for details) objects to be archived. The **String**, **Data**, **Array**, and **Map** types are considered objects.

## Format

### Diagram notation key

	  1 byte      bytes       object      objects
	 --------    ~~~~~~~~    ========    ≈≈≈≈≈≈≈≈≈
	|        |  |        |  │        │  |         |
	 --------    ~~~~~~~~    ========    ≈≈≈≈≈≈≈≈≈

### File structure

Objects are stored consecutively, indexed by unsigned integers ascending from 0, ending with the root object. The sequence of objects ends at the end of the file or stream.

	 ======== ======== ======== ≈≈≈≈≈≈≈≈ ======== 
	│  id 0  │  id 1  │  id 2  |        |  root  │  EOF
	 ======== ======== ======== ≈≈≈≈≈≈≈≈ ======== 

### Value format overview

format   | first byte | hex
------   | ---------- | ---
ref6     | `00xxxxxx` | `0x00` – `0x3f`
ref8     | `01000000` | `0x40`
ref16    | `01100000` | `0x60`
ref32    | `01110000` | `0x70`
+int6    | `10xxxxxx` | `0x80` – `0xbf`
-int5    | `111xxxxx` | `0xe0` – `0xff`
false    | `11000000` | `0xc0`
true     | `11000001` | `0xc1`
nil      | `11000010` | `0xc2`
int8     | `11000011` | `0xc3`
int16    | `11000100` | `0xc4`
int32    | `11000101` | `0xc5`
int64    | `11000110` | `0xc6`
uint8    | `11000111` | `0xc7`
uint16   | `11001000` | `0xc8`
uint32   | `11001001` | `0xc9`
uint64   | `11001010` | `0xca`
float32  | `11001011` | `0xcb`
float64  | `11001101` | `0xcd`
pair     | `11001100` | `0xcc`
fstring  | `0111xxxx` | `0x71` – `0x7f`
vstring  | `11001110` | `0xce`
estring  | `11001111` | `0xcf`
fdata    | `0110xxxx` | `0x61` – `0x6f`
vdata8   | `11010000` | `0xd0`
vdata16  | `11010001` | `0xd1`
vdata32  | `11010010` | `0xd2`
vdata64  | `11010011` | `0xd3`
edata    | `11010100` | `0xd4`
farray   | `010xxxxx` | `0x41` – `0x5f`
varray   | `11010101` | `0xd5`
earray   | `11010110` | `0xd6`
map      | `11010111` | `0xd7`
emap     | `11011000` | `0xd8`
reserved | `11011xxx` | `0xd9` – `0xdf`

### References

A reference stores an integer reference to a top-level object. The smallest possible reference type should be used. As an optimisation, implementations may sort objects in descending order of use frequency, so frequently-used objects have small integer ids that fit in smaller reference types.

	ref6 [0, 63]      ref8 [0, 255]
	 ---------      -------- --------
	|00xxxxxxx|    |01000000|xxxxxxxx|
	 ---------      -------- --------

	     ref16 [0, 65 535]
     -------- -------- --------
    |01100000|xxxxxxxx|xxxxxxxx|
	 -------- -------- --------

	           ref32 [0, 4 294 967 295]
	 -------- -------- -------- -------- --------
    |01110000|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
     -------- -------- -------- -------- --------

### Integers

Integer formats beginning with `int` are signed integers in 2's complement form. Formats beginning with `uint` are unsigned integers starting at 0.

	+int6: unsigned, clear the first bit to obtain the value
	 --------
	|10xxxxxx|
	 --------

	-int5: treat as 8-bit signed integer to obtain correct (negative) value
	 --------
	|111xxxxx|
	 --------

	       int8                 uint8
	 -------- --------    -------- -------- 
	|  0xc3  |xxxxxxxx|  |  0xc7  |xxxxxxxx|
	 -------- --------    -------- -------- 

	           int16                         uint16
	 -------- -------- --------    -------- -------- -------- 
	|  0xc4  |xxxxxxxx|xxxxxxxx|  |  0xc8  |xxxxxxxx|xxxxxxxx|
	 -------- -------- --------    -------- -------- -------- 

	                    int32
	 -------- -------- -------- -------- -------- 
	|  0xc5  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- 

	                    uint32
	 -------- -------- -------- -------- -------- 
	|  0xc9  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- 

	                                      int64
	 -------- -------- -------- -------- -------- -------- -------- -------- --------
	|  0xc6  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- -------- -------- -------- --------

	                                      uint64
	 -------- -------- -------- -------- -------- -------- -------- -------- --------
	|  0xca  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- -------- -------- -------- --------

### Nil

	   nil
	 --------
	|  0xc2  |
	 --------

### Boolean

	   false       true
	 --------    --------
	|  0xc0  |  |  0xc1  |
	 --------    --------

### Float

Floating-point numbers are stored in the corresponding IEEE 754 format.

	                   float32
	 -------- -------- -------- -------- -------- 
	|  0xcb  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- 

	                                     float64
	 -------- -------- -------- -------- -------- -------- -------- -------- --------
	|  0xcd  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|
	 -------- -------- -------- -------- -------- -------- -------- -------- --------

### String

	fstring: fixed-length string (no null terminator)
	 --------
	|0111xxxx|
	 --------

The 4-bit unsigned integer denoted `xxxxx` stores the length of the string, 1–15.

	vstring: null-terminated string
	 -------- ~~~~~~~~ --------
	|  0xce  |  text  |  0x00  |
	 -------- ~~~~~~~~ --------

**Note**: zero-length strings *cannot* be stored in the `fstring` format, as this would conflict with the `ref32` format. Instead, store an `estring`:

	estring: empty string
	 --------
	|  0xcf  |
	 --------

### Data

	fdata: fixed-length data
	 -------- ~~~~~~~~~
	|0110xxxx|  bytes  |
	 -------- ~~~~~~~~~

The length of an `fdata` is given by the 4-bit unsigned integer in the first byte.

	vdata8: data of 8-bit length       vdata16: data of 16-bit length
	 -------- -------- ~~~~~~~~~    -------- -------- -------- ~~~~~~~~~
	|  0xd0  |xxxxxxxx|  bytes  |  |  0xd1  |xxxxxxxx|xxxxxxxx|  bytes  |
	 -------- -------- ~~~~~~~~~    -------- -------- -------- ~~~~~~~~~

	             vdata32: data of 32-bit length
	 -------- -------- -------- -------- -------- ~~~~~~~~~
	|  0xd2  |xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|  bytes  |
	 -------- -------- -------- -------- -------- ~~~~~~~~~

	                               vdata64: data of 64-bit length
	 -------- --------- -------- -------- -------- ------- -------- -------- -------- ~~~~~~~~~
	|  0xd3  |xxxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxx|xxxxxxxx|xxxxxxxx|xxxxxxxx|  bytes  |
	 -------- --------- -------- -------- -------- ------- -------- -------- -------- ~~~~~~~~~

The length of a `vdata` is given by the unsigned integer that follows the `vdata` signature byte.

**Note**: zero-length data *cannot* be stored in the `fdata` format, as this would conflict with the `ref16` format. Instead, store an `edata`:

	edata: empty data
	 --------
	|  0xd4  |
	 --------

### Pair

	             pair
	 -------- ========= ==========
	|  0xcc  |  first  |  second  |
	 -------- ========= ==========

### Array

	farray: fixed-length array
	 -------- ≈≈≈≈≈≈≈≈≈
	|010xxxxx| objects |
	 -------- ≈≈≈≈≈≈≈≈≈

	varray: nil-terminated array
	 -------- ≈≈≈≈≈≈≈≈≈ =======
	|  0xd5  | objects |  nil  |
	 -------- ≈≈≈≈≈≈≈≈≈ =======

**Note**: zero-length arrays *cannot* be stored in the `farray` format, as this would conflict with the `ref8` format. Instead, store an `edata`:

	earray: empty array
	 --------
	|  0xd6  |
	 --------

### Map

	     map: array-backed map
	 -------- ====================
	|  0xd7  |  farray or varray  |
	 -------- ====================

The array that follows the map designator contains alternating keys and values, in the form (key0, val0, key1, val1, ...).
	
To conserve space, use `emap` to store an empty map in one byte:

	emap: empty map
	 --------
	|  0xd8  |
	 --------

