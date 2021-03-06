## compressed_vec Vector Format

The format for each vector is defined in detail below.  It is loosely based on the vector format in [FiloDB](https://github.com/filodb/FiloDB) used for histograms.  Each vector is divided into sections of 256 elements.

The vector bytes are wire-format ready, they can be written to and read from disk or network and interpreted/read with no further transformations needed.

The goals of the vector format are:
* Optimize for fast, SIMD-based decoding.
* Enable fast, aligned data filtering and processing by having fixed section boundaries
* Varied encoding techniques to bring compression to within roughly 2x of good general purpose compression techniques, or better
* Contain metadata to enable fast filtering

### Header

The header is 16 bytes.  The first 6 bytes are binary compatible with FiloDB vectors.  The structs are defined in `src/vector.rs` in the `BinaryVector` and `FixedSectStats` structs.

| offset | description |
| ------ | ----------- |
| +0     | u32: total number of bytes in this vector, NOT including these 4 length bytes |
| +4     | u8: Major vector type, see the `VectorType` enum for details  |
| +5     | u8: Vector subtype, see the `VectorSubType` enum for details  |
| +8     | u32: total number of elements in this vector                  |
| +12    | u16: number of null sections in this vector, used for quickly determining relative sparsity  |

For the vectors produced by this crate, the major type code used is `VectorType::FixedSection256` (0x10), while the minor type code is `Primitive`.

### Sections

Following the header are one or more sections of fixed 256 elements each.  If the last section does not have 256 elements, nulls are added until the section has 256 elements.  Sections cannot carry over state to adjacent sections; each section must contain enough state to completely decode itself.  This is needed for fast iteration and skipping over sections in filtering and data processing.

The first byte of a section contains the section code.  See the `SectionType` enum in `src/section.rs` for the up to date list of codes, but it is included here for convenience:

```rust
pub enum SectionType {
    Null = 0,                 // FIXED_LEN unavailable or null elements in a row
    NibblePackedMedium = 1,   // Nibble-packed u64/u32's, total size < 64KB
    DeltaNPMedium      = 3,   // Nibble-packed u64/u32's, delta encoded, total size < 64KB
    Constant           = 5,   // Constant value section
    XorNPMedium        = 6,   // XORed f64/f32, NibblePacked, total size < 64KB
}
```

### Null Sections

A null section represents 256 zeroes, and just contains the single Null section type byte.

Null sections are key to encoding sparse vectors efficiently, and should be leveraged as much as possible.

### NibblePacked U64/U32 sections

The NibblePacked section codes (1/2) represent 256 values (u32 or u64), packed in groups of 8 using the [NibblePacking](https://github.com/filodb/FiloDB/blob/develop/doc/compression.md#predictive-nibblepacking) algorithm from FiloDB (used in production at massive scale).  NibblePacking uses only 1 bit for zero values, and stores the minimum number of nibbles only.  From the start of the section, there are 3 header bytes, followed by the NibblePack-encoded data.

| offset | description |
| ------ | ----------- |
| +0     | u8: section type code: 1 |
| +1     | u16: number of bytes of this section, excluding these 3 header bytes  |
| +3     | Start of NibblePack-encoded data, back to back.   This starts with the bitmask byte, then the number of nibbles byte, then the nibbles, repeated for every group of 8 u64's/u32's |

### Delta-Encoded NibblePacked Sections

For values such as timestamps which are mostly in a certain narrow range, the naive NibblePacked algorithm above might result in more nibbles than necessary.  Delta-encoded sections store a delta from the minimum value in the stretch of 256 raw values, and the deltas are then NibblePack compressed.  The goal here is to attain higher compression as the deltas should be smaller.

| offset | description |
| ------ | ----------- |
| +0     | u8: section type code: 3 |
| +1     | u16: number of bytes of this section, excluding these 3 header bytes  |
| +3     | u8: number of bits needed for the largest delta.  Or the smallest n where 2^n >= max raw value.  |
| +4     | u64: The "base" value to which all deltas are added to form original value |
| +12     | Start of NibblePack-encoded deltas, back to back.   This starts with the bitmask byte, then the number of nibbles byte, then the nibbles, repeated for every group of 8 u64's/u32's |

### Constant Sections

These sections represent 256 repeated values.  

| offset | description |
| ------ | ----------- |
| +0     | u8: section type code: 5 |
| +1     | u32/u64/etc.: the constant value |

### XOR floating point NibblePacked sections

This is a Gorilla- and Prometheus- inspired algorithm but designed for fast SIMD unpacking.   Floating point numbers that are similar will XOR such that the result only contains a few set bits.  NibblePacking algorithm then packs only the nonzero nibbles, taking care of long trailing zero nibbles.  The algorithm starts with 0's, thus the initial octet gets NibblePacked in the stream.

| offset | description |
| ------ | ----------- |
| +0     | u8: section type code: 6 |
| +1     | u16: number of bytes of this section, including header bytes  |
| +3     | NibblePacked XORed values |

Each set of 8 values are XORed against the previous set of 8 values, and the difference is NibblePacked.

### Filtering and Vector Processing

Fast filtering and vector processing of multiple vectors is enabled by the following:
* All sections are the same number of elements across vectors, thus
* we can iterate through sections and process the same set of elements across multiple vectors at the same time
* processing of sparse vectors with null sections can be optimized and special cased

We can see examples of this in `src/filter.rs`, with the `VectorFilter` and `MultiVectorFilter` structs.
