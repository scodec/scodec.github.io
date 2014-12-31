
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

### Getting Started

```scala
scala> import scodec.Codec

scala> import scodec.codecs.implicits._

scala> case class Point(x: Int, y: Int)

scala> case class Line(start: Point, end: Point)

scala> case class Arrangement(lines: Vector[Line])

scala> val arr = Arrangement(Vector(
  Line(Point(0, 0), Point(10, 10)),
  Line(Point(0, 10), Point(10, 0))))
arr: Arrangement = ...

scala> val arrBinary = Codec.encodeValid(arr)
arrBinary: scodec.bits.BitVector =
  BitVector(288 bits, 0x0000000200000000000000000000000a0000000a000000000000000a0000000a00000000)

scala> val decoded = Codec[Arrangement].decodeValidValue(arrBinary)
decoded: Arrangement = Arrangement(Vector(Line(Point(0,0),Point(10,10)), Line(Point(0,10),Point(10,0))))
```

We start by importing the primary type in scodec-core, the `Codec` type, along with all implicit codecs defined in `scodec.codecs.implicits`. The latter provides commonly useful implicit codecs, but is opinionated -- for instance, it provides a `Codec[Int]` that encodes to 32-bit 2s complemenet big endian format.

Aside: the predefined implicit codecs are useful at the REPL and when your application does not require a specific binary format. However, scodec-core is designed to support "contract-first" binary formats -- ones in which the format is fixed in stone. For binary serialization to arbitrary formats, consider toos like [scala-pickling](https://github.com/scala/pickling), [Avro](http://avro.apache.org), and [protobuf](https://code.google.com/p/protobuf/).

We then create three case classes followed by instantiating them all and assigning the result to the `arr` val. We encode `arr` to binary using `Codec.encodeValid`, then decode the resulting binary back to an `Arrangement`. In this example, both encoding and decoding rely on an implicitly available `Codec[Arrangement]`, which is automatically derived based on *compile time* reflection on the structure of the `Arrangement` class and its product types.

We use `encodeValid`, which throws an `IllegalArgumentException` if encoding fails, because we know that our arrangement codec cannot fail to encode. To decode, we summon the implicit arrangement codec via `Codec[Arrangement]` and then use `decodeValidValue` for REPL convenience -- which throws an `IllegalArgumentException` if decoding fails and throws away any bits left over after decoding finishes. In this case, we know that decoding will succeed and there will be no remaining bits, so this is safe. It is generally better to use the `encode` and `decode` methods instead of the "valid" conveniences.

Running the same code with a different implicit `Codec[Int]` in scope changes the output accordingly:

```scala

scala> import scodec.codecs.implicits.{ implicitIntCodec => _, _ }

scala> implicit val ci = scodec.codecs.uint8
ci: scodec.Codec[Int] = 8-bit unsigned integer

...

scala> val arrBinary = Codec.encodeValid(arr)
arrBinary: scodec.bits.BitVector = BitVector(72 bits, 0x0200000a0a000a0a00)

scala> val decoded = Codec.decodeValidValue[Arrangement](arrBinary)
decoded: Arrangement = Arrangement(Vector(Line(Point(0,0),Point(10,10)), Line(Point(0,10),Point(10,0))))
```

In this case, we import all predefined implicits except for the `Codec[Int]` and then we define an implicit `Int` codec for 8-bit unsigned big endian integers. The resulting encoded binary is 1/4 the size. However, our arrangement codec is no longer total in encoding -- that is, it may result in errors. Consider:

```scala
scala> val arr2 = Arrangement(Vector(
  Line(Point(0, 0), Point(10, 10)),
  Line(Point(0, 10), Point(10, -1))))
arr2: Arrangement = Arrangement(Vector(Line(Point(0,0),Point(10,10)), Line(Point(0,10),Point(10,-1))))

scala> val encoded = Codec.encode(arr2)
encoded: scalaz.\/[scodec.Err,scodec.bits.BitVector] =
  -\/(lines/1/end/y: -1 is less than minimum value 0 for 8-bit unsigned integer)

scala> val encoded = Codec.encodeValid(arr2)
java.lang.IllegalArgumentException: lines/1/end/y: -1 is less than minimum value 0 for 8-bit unsigned integer
```

Attempting to encode an arrangement that contains a point with a negative number resulted in an error being returned from `encode` and an exception being thrown from `encodeValid`. The error includes the path to the error -- `lines/1/end/y`. In this case, the `lines` field on `Arrangement`, the line at the first index of that vector, the `end` field on that line, and the `y` field on that point.

If you prefer to avoid using implicits, do not fret! The above example makes use of implicits and uses Shapeless for compile time reflection, but this is built as a layer on top of the core algebra of scodec-core. The library supports a usage model where implicits are not used.

With the first example under our belts, let's look at the core algebra in detail.

Core Algebra
============

We saw the `Codec` type when we used it to encode a value to binary and decode binary back to a value. The ability to decode and encode come from two fundamental traits, `Decoder` and `Encoder`. Let's look at these in turn.

## Decoder

```scala
trait Decoder[+A] {
  def decode(b: BitVector): Err \/ (BitVector, A)
}
```

A decoder defines a single abstract operation, `decode`, which converts a bit vector in to a pair containing the unconsumed bits and a decoded value, or returns an error. For example, a decoder that decodes a 32-bit integer returns an error when the supplied vector has less than 32-bits, and returns the supplied vector less 32-bits otherwise.

The result type is a disjunction with `scodec.Err` on the left side. `Err` is an open-for-subclassing data type that contains an error message and a context stack. The context stack contains strings that provide context on where the error occurred in a large structure. We saw an example of this earlier, where the context stack represented a path through the `Arrangement` class, in to a `Vector`, and then in to a `Line` and `Point`. The type is open-for-subclassing so that codecs can return domain specific error types and then pattern match on the received type. An `Err` is *not* a subtype of `Throwable`, so it cannot be used (directly) with `scala.util.Try`. Also note that codecs never throw exceptions (or should never!). All errors are communicated via the `Err` type.

### map

A function can be mapped over a decoder, resulting in our first combinator:

```scala
trait Decoder[+A] { self =>
  def decode(b: BitVector): Err \/ (BitVector, A)
  def map[B](f: A => B): Decoder[B] = new Decoder[B] {
    def decode(b: BitVector): Err \/ (BitVector, B) =
      self.decode(b) map { case (rem, a) => (rem, f(b)) }
  }
}
```

Note that the *implementation* of the `map` method is not particularly important -- rather, the type signature is the focus.

As a first use case for `map`, consider creating a decoder for the following case class by reusing the built-in `int32` codec:

```scala
case class Foo(x: Int)
val fooDecoder: Decoder[Foo] = int32 map { i => Foo(i) }
```

### emap

The `map` operation does not allow room for returning an error. We can define a variant of `map` that allows the supplied function to indicate error:

```scala
trait Decoder[+A] { self =>
  ...
  def emap[B](f: A => Err \/ B): Decoder[B] = new Decoder[B] {
    def decode(b: BitVector): Err \/ (BitVector, B) =
      self.decode(bits) flatMap { case (rem, a) =>
        f(a).map { b => (rem, b) }
      }
  }
}
```

### flatMap

Further generalizing, we can `flatMap` a function over a decoder to express that the "next" codec is *dependent* on the decoded value from the current decoder:

```scala
trait Decoder[+A] { self =>
  def decode(b: BitVector): Err \/ (BitVector, A)
  def flatMap[B](f: A => Decoder[B]): Decoder[B] = new Decoder[B] {
    def decode(b: BitVector): Err \/ (BitVector, B) =
      self.decode(b) flatMap { case (rem, a) =>
        val next: Codec[B] = f(a)
        next.decode(rem)
      }
  }
}
```

The resulting decoder first decodes a value of type `A` using the original decoder. If that's successful, it applies the decoded value to the supplied function to get a `Decoder[B]` and then decodes the bits remaining from decoding `A` using that decoder.

As mentioned previously, `flatMap` models a dependency between a decoded value and the decoder to use for the remaining bits. A good use case for this is a bit pattern that first encodes a count followed by a number of records. An implementation of this is not provided because we will see it later in a different context.

## Encoder

TODO

## Codec

TODO

## GenCodec

TODO
