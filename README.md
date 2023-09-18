# node-bswap

x86: ![x86 Build Status](https://github.com/zbjornson/node-bswap/actions/workflows/ci.yaml/badge.svg)  
ARM: [![ARM Build Status](https://cloud.drone.io/api/badges/zbjornson/node-bswap/status.svg)](https://cloud.drone.io/zbjornson/node-bswap)

The fastest function to swap bytes (a.k.a. reverse the byte ordering, change
endianness) of TypedArrays in-place for Node.js, Bun and browsers. Uses SIMD
when available. Works with all of the TypedArray types, including BigUint64Array
and BigInt64Array. Also works on Buffers if you construct a TypedArray view on
the underlying ArrayBuffer (see below).

Install:
```
$ npm install bswap
$ bun install bswap  # but see https://bun.sh/guides/install/trusted
```

Use:
```js
import bswap from "bswap";
const x = new Uint16Array([1, 2, 3, 4, 5, 6, 7, 8]);
bswap(x);
// now: Uint16Array [ 256, 512, 768, 1024, 1280, 1536, 1792, 2048 ]

// With buffers:
const b = Buffer.alloc(128);
// This constructs a "view" on the same memory; it does not allocate new memory:
const ui32 = new Uint32Array(b.buffer, b.byteOffset, b.byteLength / Uint32Array.BYTES_PER_ELEMENT);
bswap(ui32);
```

In Node.js/Bun when native code and a recent x86 or ARM processor is available,
this library uses the fastest available SIMD instructions ([PSHUFB (SSSE3) or
VPSHUFB (AVX2)](http://www.felixcloutier.com/x86/PSHUFB.html), [REVn
(NEON)](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0489h/Cihjgdid.html)),
which process multiple array elements simultaneously.

In the browser or when native code is unavailable, this library falls back to
the fastest JavaScript implementation. The JavaScript implementation is also
always explicitly available:

```js
import {js} from "bswap"; // Use javascript implementation explicitly
```

## Benchmarks

Showing millions of elements processed per second when invoked with a
10,000-element array. (Run the benchmark suite to see results for varying array
lengths and other libraries.) Ran on an Intel i9-11900H 2.50 GHz processor (AVX2
supported) or Cavium ThunderX 2.0 GHz processor (ARM NEON); Node.js v16.x;
Windows 11 (MSVC) or Ubuntu 20.04 (GCC, Clang). (Note that a 10,000-element
Int16Array fits in L1 cache, whereas a 10,000-element Int32Array or Float64Array
does not.)

| compiler  |    C++ |   JS   | Native:JS | Node.js | Native:Node |
| --------- | -----: | ---:   | --------: | ------: | ----------: |
| **16 bit types (Uint16Array, Int16Array)**
| MSVC 2022 | 46,221 |    722 |     64.0x |  18,213 |        2.5x |
| GCC 9.4   | 40,945 |     〃 |     56.8x |  13,720 |        2.9x |
| Clang 15  | 47,398 |     〃 |     65.6x |      〃 |        3.5x |
| GCC-ARM   |  2,677 |    183 |     14.6x |     297 |        9.0x |
| **32 bits types (Uint32Array, Int32Array, Float32Array)**
| MSVC 2022 | 27,459 |    342 |     36.7x |   9,431 |        2.9x |
| GCC 9.4   | 23,613 |     〃 |     61.9x |   2,842 |        8.3x |
| Clang 15  | 29,013 |     〃 |     84.8x |      〃 |       10.2x |
| GCC-ARM   |    670 |     94 |      7.1x |     249 |        2.7x |
| **64 bit types (Float64Array)**
| MSVC 2022 |  9,005 |    179 |     38.2x |   4,348 |        2.1x |
| GCC 9.4   |  8,774 |     〃 |     49.1x |   2,642 |        3.3x |
| Clang 15  |  8,937 |     〃 |     49.9x |      〃 |        3.4x |
| GCC-ARM   |    382 |     49 |      7.8x |     213 |        1.8x |

There's an AVX512 implementation that is disabled by default. On the Cascade
Lake CPU that I tested on, it is ~28% faster than the AVX2 version when the data
fit in the L1 cache. However, it is ~10% slower than the AVX2 version when the
data come from L2 and ~15% slower from L3. Under the assumption that this module
is more often used with arrays larger than 32KB, I've thus left it disabled.
Sometime maybe I'll make it select between AVX2 and AVX512 depending on the
array length, but this module has no ability to know if the data is resident in
the L1 cache.

## Comparison to other libraries

| Library | Operand | In-Place | 64-bit Type Support | Browser | Speed (vs. bswap)* |
| --- | --- | --- | --- | --- | --- |
| bswap (this) | TypedArray | yes | yes | yes | 1.00 |
| node [`buffer.swap16/32/64`](https://nodejs.org/api/buffer.html#buffer_buf_swap16) | Buffer | yes | since 6.3.0 | no | 0.05 to 0.38 |
| [network-byte-order](https://github.com/mattcg/network-byte-order) | Number/\[Octet\] | no | no | yes | 0.010 |
| [endian-toggle](https://github.com/substack/endian-toggle) | Buffer | no | yes | no | 0.0056 |

\* Higher is better. For 16-bit types, 10k-element arrays. Range given for
Node.js version reflects Windows vs. Linux benchmark.

* **Node.js' built-in
  [buffer.swap16|32|64](https://nodejs.org/api/buffer.html#buffer_buf_swap16)
  methods** (16/32 since v5.10.0; 64 since 6.3.0). Operates in-place. No browser
  support. Slower except for tiny arrays (where it uses the JS implementation).

  In 6.3.0 I added some optimizations to Node.js' implementation. The
  optimizations are effective on Windows, but GCC does not do the same automatic
  vectorization that MSVC does, nor does Node's default build config enable the
  newer SIMD instructions that this library uses.

  <details><summary>Usage</summary>

  ```js
  > Buffer.from(typedArray.buffer).swap16()
  ```

  </details>

* **[endian-toggle](https://github.com/substack/endian-toggle)**. Simple usage,
  operates on a Node.js Buffer, handles any byte size, returns a new buffer
  (does not operate in-place).

  <details><summary>Usage</summary>

  ```js
  > const x = new Uint16Array([2048])
  > toggle(Buffer.from(x.buffer), x.BYTES_PER_ELEMENT * 8)
  <Buffer d2 04 09 07>
  ```

  </details>

* **[network-byte-order](https://github.com/mattcg/network-byte-order)**.
  Operates on a single value at a time (i.e. needs to be looped to operate on an
  array) and has separate `hton` and `ntoh` methods, which do effectively the
  same thing but have different syntaxes. It can operate on strings, but it
  cannot swap 64-bit types.

  <details><summary>Usage</summary>

  ```js
  // Using hton
  > const b = [];
  > nbo.htons(b, 0, 2048);
  > b
  [8, 0]
  
  // or using ntoh
  > const x = new Uint16Array([2048])
  > nbo.ntohs(new Uint8Array(x.buffer, x.byteOffset, 2), 0)
  8
  > const z = new Uint16Array([8])
  > new Uint8Array(z.buffer, z.byteOffset, 2)
  Uint8Array [ 8, 0 ]
  ```

  </details>
