
scodec
======

scodec is a suite of libraries for working with binary data. Support ranges from simple, performant data structures for working with bits and bytes to streaming encoding and decoding.


There are three primary modules:

 - scodec-bits - Zero dependency library that provides persistent data structures, `BitVector` and `ByteVector`, for working with binary.
 - scodec-core - Combinator based library for encoding/decoding values to/from binary.
 - scodec-stream - Binding between scodec-core and [scalaz-stream](http://github.com/scalaz/scalaz-stream) that enables streaming encoding/decoding.

There are a few secondary modules as well:

 - scodec-spire - Binding between scodec-core and [spire](http://github.com/non/spire), mostly taking advantage of unsigned numeric types.
 - scodec-protocols - Library of general purpose implementations of common protocols.

This guide goes over each of these modules in detail.


scodec-bits
===========

The scodec-bits library contains data structures for working with binary. It has no dependencies on other libraries, which allows it to be used by other libraries without causing dependency conflicts.

The source is on [Github](http://github.com/scodec/scodec-bits) and binaries are available on Maven Central under `org.typelevel / scodec-bits_${scalaBinaryVersion}`. ScalaDoc is available for online browsing at [typelevel.org](http://docs.typelevel.org/api/scodec/bits/stable/). The library adheres to the Typelevel binary compatibility guidelines. In short, versions that share the same major.minor version number are forward binary compatible.

There are two primary data structures in the library, `ByteVector` and `BitVector`. Both are immutable collections and have performance characteristics that are optimized for use in the other scodec modules. However, each type has been designed for general purpose usage, even when other scodec modules are not used. For instance, `ByteVector` can be safely used as a replacement for immutable byte arrays.

ByteVector
----------

The `ByteVector` type is isomorphic to a `scala.collection.immutable.Vector[Byte]` but has much better performance characteristics. A `ByteVector` is represented as a balanced binary tree of chunks. Most operations have asymptotic performance that is logarthmic in the depth of this tree. There are also quite a number of convenience based features, like value based equality, a sensible `toString`, and many conversions to/from other data types.

It is important to note that `ByteVector` does not extend any types from the Scala collections framework. For instance, `ByteVector` is *not* a `scala.collection.immutable.Traversable[Byte]`. This allows some deviance, like `Long` based indexing instead of `Int` based indexing from standard collections. Additionally, it avoids a large category of bugs, especially as the standard library collections are refactored. Nonetheless, the methods on `ByteVector` are named to correspond with the methods in the standard library when possible.

### Getting Started

Let's create a `ByteVector` from a literal hexadecimal string:

```scala
scala> import scodec.bits._
import scodec.bits._

scala> val x: ByteVector = hex"deadbeef"
x: scodec.bits.ByteVector = ByteVector(4 bytes, 0xdeadbeef)

scala> val y: ByteVector = hex"DEADBEEF"
y: scodec.bits.ByteVector = ByteVector(4 bytes, 0xdeadbeef)

scala> x == y
res0: Boolean = true
```

We first start by importing all members of the `scodec.bits` package, which contains the entirety of this library. We then create two byte vectors from hexadecimal literals, using the `hex` string interpolator. Finally, we compare them for equality, which returns true, because each vector contains the same bytes.

TODO

BitVector
---------

The `BitVector` type is similar to `ByteVector` with the exception of indexing bits instead of bytes. This allows access and update of specific bits (via `apply` and `update`) as well as storage of a bit count that is not evenly divisible by 8.

### Getting Started

```scala
scala> val x: BitVector = bin"00110110101"
x: scodec.bits.BitVector = BitVector(11 bits, 0x36a)

scala> val y: BitVector = bin"00110110100"
y: scodec.bits.BitVector = BitVector(11 bits, 0x368)

scala> x == y
res0: Boolean = false

scala> val z = y.update(10, true)
z: scodec.bits.BitVector = BitVector(11 bits, 0x36a)

scala> x == z
res1: Boolean = true
```

In this example, we create two 10-bit vectors using the `bin` string interpolator that differ in only the last bit. We then create a third vector, `z`, by updating the 10th bit of `y` to true. Comparing `x` and `y` for equality returns false whereas comparing `x` and `z` returns true.

TODO

scodec-core
===========

TODO
