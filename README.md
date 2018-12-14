lexical
=======

[![Build Status](https://api.travis-ci.org/Alexhuszagh/rust-lexical.svg?branch=master)](https://travis-ci.org/Alexhuszagh/rust-lexical)
[![Latest Version](https://img.shields.io/crates/v/lexical.svg)](https://crates.io/crates/lexical)

Fast lexical conversion routines for both std and no_std environments. Lexical provides routines to convert numbers to and from decimal strings. Lexical also supports non-base 10 numbers, for both integers and floats. Lexical is simple to use, focuses on performance and correctness, and exports only 10 functions in the high-level API. Finally, lexical is suitable for environments without a memory allocator, without any internal allocations required for the low-level API.

**Table of Contents**

- [Getting Started](#getting-started)
- [Benchmarks](#benchmarks)
- [Documentation](#documentation)
- [Validation](#validation)
- [Caveats](#caveats)
- [Details](#details)
  - [Float to String](#float-to-string)
  - [String to Float](#string-to-float)
  - [Arbitrary-Precision Arithmetic](#arbitrary-precision-arithmetic)
  - [Comparison to Algorithm M and dtoa](#comparison-to-algorithm-m-and-dtoa)
- [Roadmap](#roadmap)
- [License](#license)
- [Contributing](#contributing)

# Getting Started

Add lexical to your `Cargo.toml`:

```yaml
[dependencies]
lexical = "1"
```

And get started using lexical:

```rust
extern crate lexical;

// Number to string
lexical::to_string(3.0);            // "3.0", always has a fraction suffix, 
lexical::to_string(3);              // "3"
lexical::to_string_radix(3.0, 2);   // "11.0", non-base 10 representation.
lexical::to_string_radix(1.5, 2);   // "1.1"
lexical::to_string_radix(1.25, 2);  // "1.01"

// String to number.
let i: i32 = lexical::parse("3");            // 3, auto-type deduction.
let f: f32 = lexical::parse("3.5");          // 3.5
let d = lexical::parse::<f64, _>("3.5");     // 3.5, explicit type hints.
let d = lexical::try_parse::<f64, _>("3.5"); // Ok(3.5), error checking parse.
let d = lexical::try_parse::<f64, _>("3a");  // Err(Error(_)), failed to parse.
```

Lexical's parsers can be either error-checked and unchecked. The unchecked parsers continue to parse until they encounter invalid data or overflow, returning a number was successfully parsed up until that point. This is analogous to C's `strtod`, which may not be desirable for many applications. Therefore, lexical also includes checked parsers, which ensure the entire buffer is used while parsing, without discarding characters, and that the resulting number did not overflow. Upon erroring, the checked parsers will return the an enum indicating overflow or the index where the first invalid digit  was found.

```rust
// This will return Err(Error(ErrorKind::InvalidDigit(3))), indicating 
// the first invalid character occurred at the index 3 in the input 
// string (the space character).
let x: i32 = lexical::try_parse("123 456");

// This will return Ok(123), since that is the value found before invalid
// character was encountered.
let x: i32 = lexical::parse("123 456");
```

For floating-points, Lexical also includes `parse_lossy` and `try_parse_lossy`, which may lead to minor rounding error (relative error of ~1e-16) in rare cases (see [details](#details) for more information), without using slow algorithms that lead to serious performance degradation.

```rust
let x: f32 = lexical::parse_lossy("3.5");       // 3.5
let x: f32 = lexical::try_parse_lossy("3.5");   // Ok(3.5)
```

In order to use lexical in generics, the type may use the trait bounds `FromBytes` (for `parse` and `try_parse`), `ToBytes` (for `to_string`), or `FromBytesLossy` (for `parse_lossy` and `try_parse_lossy`).

# Benchmarks

The following benchmarks measure the time it takes to convert 10,000 random values, for different types. The values were randomly generated using NumPy, and run in both std (rustc 1.29.2) and no_std (rustc 1.31.0) contexts (only std is shown) on an x86-64 Intel processor. More information on these benchmarks can be found in the [benches](benches) folder and in the source code for the respective algorithms. Adding the flags "target-cpu=native" and "link-args=-s" were also used, however, they minimally affected the relative performance difference between different lexical conversion implementations.

For cross-language benchmarks, the C++ benchmark was done using GCC 8.2.1 with glibc/libstdc++ using Google Benchmark and the `-O3` flag. The Python benchmark was done using IPython on Python 3.6.6. The Go benchmark was done using go1.10.4. All benchmarks used the same data.

For all the following benchmarks, lower is better.

**Float to String**

![ftoa benchmark](https://raw.githubusercontent.com/Alexhuszagh/rust-lexical/master/assets/ftoa.png)

**Integer To String**

![itoa benchmark](https://raw.githubusercontent.com/Alexhuszagh/rust-lexical/master/assets/itoa.png)

**String to Integer**

![atoi benchmark](https://raw.githubusercontent.com/Alexhuszagh/rust-lexical/master/assets/atoi.png)

**String to f32 Simple, Random Data**

![atof32 benchmark](https://raw.githubusercontent.com/Alexhuszagh/rust-lexical/master/assets/atof_simple_f32.png)

**String to f64 Simple, Random Data**

![atof64 benchmark](https://raw.githubusercontent.com/Alexhuszagh/rust-lexical/master/assets/atof_simple_f64.png)

**String to f64 Complex, Large Data Cross-Language Comparison**

![atof64 simple language benchmark](https://raw.githubusercontent.com/Alexhuszagh/rust-lexical/master/assets/atof_large_f64.png)

**String to f64 Complex, Denormal Data Cross-Language Comparison**

Note: Rust was unable to parse all but the 20-digit benchmark, producing an error result of `ParseFloatError { kind: Invalid }`. It performed ~2,000x worse than lexical for that benchmark.

![atof64 simple language benchmark](https://raw.githubusercontent.com/Alexhuszagh/rust-lexical/master/assets/atof_denormal_f64.png)

# Backends

For Float-To-String conversions, lexical uses one of three backends: an internal, Grisu2 algorithm, an external, Grisu3 algorithm, and an external, Ryu algorithm (~2x as fast).

# Documentation

Lexical's documentation can be found on [docs.rs](https://docs.rs/lexical).

# Validation

Float parsing is difficult to do correctly, and major bugs have been found in implementations from [libstdc++'s strtod](https://www.exploringbinary.com/glibc-strtod-incorrectly-converts-2-to-the-negative-1075/) to [Python](https://bugs.python.org/issue7632). In order to validate the accuracy of the lexical, we employ the following external tests:

1. Hrvoje Abraham's [strtod](https://github.com/ahrvoje/numerics/tree/master/strtod) test cases.
2. Rust's [test-float-parse](https://github.com/rust-lang/rust/tree/64185f205dcbd8db255ad6674e43c63423f2369a/src/etc/test-float-parse) unittests.
3. Testbase's [stress tests](https://www.icir.org/vern/papers/testbase-report.pdf) for converting from decimal to binary.
4. [Various](https://www.exploringbinary.com/glibc-strtod-incorrectly-converts-2-to-the-negative-1075/) [difficult](https://www.exploringbinary.com/how-glibc-strtod-works/) [cases](https://www.exploringbinary.com/how-strtod-works-and-sometimes-doesnt/) reported on blogs.

Although Lexical is likely to contain bugs leading to rounding error, it is tested against a comprehensive suite of random-data and near-halfway representations, and should be fast and correct for the vast majority of use-cases.

# Caveats

Lexical uses unsafe code in the back-end for performance, and therefore may introduce memory-safety issues. Although the code is tested with wide variety of inputs to minimize the risk of memory-safety bugs, no guarantees are made and you should use it at your own risk.

Finally, for non-decimal (base 10) floats, lexical's float-to-string implementation is lossy, resulting in rounding for a small subset of inputs (up to 0.1% of the total value).

# Details

## Float to String

For more information on the Grisu2 and Grisu3 algorithms, see [Printing Floating-Point Numbers Quickly and Accurately with Integers](https://www.cs.tufts.edu/~nr/cs257/archive/florian-loitsch/printf.pdf).

For more information on the Ryu algorithm, see [Ryū: fast float-to-string conversion](https://dl.acm.org/citation.cfm?id=3192369).

## String to Float

For float parsing, lexical tries the following algorithms, using the first algorithm that produces a correct result:

1. **Fast Path** Create an exact representation using a native floating-point type from the significant digits and exponent.
2. **Moderate Path** Create an approximate representation, exact within rounding error, using a custom floating-point type with 80-bits of precision, preventing rounding-error in the significant digits.
3. **Fallback Moderate Path** Create an approximate 128-bit representation of the numerator and denominator for the halfway point between two potentially valid representations, to determine which representation is correct by comparing the actual digits to theoretical digits generated from the halfway point. This is accurate for ~36 digits with decimal float strings.
4. **Slow Path** Use arbitrary-precision arithmetic to disambiguate the correct representation.
    - **Default** Create an exact representation of the numerator and denominator for the halfway point between both representation, using arbitrary-precision integers, and determine which representation is accurate by comparing the actual digits to the theoretical digits. This is accurate for any number of digits, and the required amount of memory does not depend on the number of digits.
    - **Algorithm M** Create an exact representation of the digits as a big integer, to generate the significant digits (mantissa). If there is a fraction or a negative exponent, create a big ratio of both the numerator and the denominator, and generate the significant digits from the exact quotient and remainder.

Although using the 4<sup>th</sup> algorithm only occurs when a float is indistinguishable from a halfway representation (between two neighboring floating-point representations) to within 36 digits, maliciously constructed input could force use of the 4<sup>th</sup> algorithm. This has seriously implications for performance, since arbitrary-precision arithmetic scales poorly with large numbers, and can lead to performance degradation in excess of 100-50,000 fold. The lossy parser therefore avoids using arbitrary-precision arithmetic, and returns the result from the 2<sup>nd</sup> step, creating a native float accurate to within 1 bit in the mantissa, without major performance regressions.

To use Algorithm M (which requires heap allocation internally), use the feature `algorithm_m` when compiling lexical.

## Arbitrary-Precision Arithmetic

Lexical uses arbitrary-precision arithmetic to exactly represent strings between two floating-point representations with more than 36 digits, with various optimizations for multiplication and division relative to Rust's current implementation. The arbitrary-precision arithmetic logic is not independent on memory allocation: the default slow-path algorithm only uses the stack, while Algorithm M uses the heap.

## Comparison to Algorithm M

For close-to-halfway representations of a decimal string `s`, where `s` is close between two representations, `b` and the next float `b+u`, arbitrary-precision arithmetic is used to determine the correct representation. This means `s` is close to `b+h`, where `h` is the halfway point between `b` and `b+u`.

For the following example, we will use the following values for our test case: 

* `s = 2.4703282292062327208828439643411068618252990130716238221279284125033775363510437593264991818081799618989828234772285886546332835517796989819938739800539093906315035659515570226392290858392449105184435931802849936536152500319370457678249219365623669863658480757001585769269903706311928279558551332927834338409351978015531246597263579574622766465272827220056374006485499977096599470454020828166226237857393450736339007967761930577506740176324673600968951340535537458516661134223766678604162159680461914467291840300530057530849048765391711386591646239524912623653881879636239373280423891018672348497668235089863388587925628302755995657524455507255189313690836254779186948667994968324049705821028513185451396213837722826145437693412532098591327667236328125001e-324`
* `b = 0.0`
* `b+h = 2.4703282292062327208828439643411068618252990130716238221279284125033775363510437593264991818081799618989828234772285886546332835517796989819938739800539093906315035659515570226392290858392449105184435931802849936536152500319370457678249219365623669863658480757001585769269903706311928279558551332927834338409351978015531246597263579574622766465272827220056374006485499977096599470454020828166226237857393450736339007967761930577506740176324673600968951340535537458516661134223766678604162159680461914467291840300530057530849048765391711386591646239524912623653881879636239373280423891018672348497668235089863388587925628302755995657524455507255189313690836254779186948667994968324049705821028513185451396213837722826145437693412532098591327667236328125e-324`
* `b+u = 5e-324`

**Algorithm M**

Algorithm M is described [here](https://www.exploringbinary.com/correct-decimal-to-floating-point-using-big-integers/) in depth, and Rust's libcore uses it internally. First, Algorithm M converts both the fraction and the exponent to a fraction of big integers. For example, `3.14159e2` is the same as `314159e-3`, which means a numerator of `314159` and denominator of `1000`. Next, Algorithm M calculates the quotient and remainder of the numerator and denominator until the quotient is the same representation as the mantissa in the native float, for example, in range range `[2^52, 2^53)` for f64, or in the range `[2^23, 2^24)` for f32. If the quotient is below the range, multiply the numerator by 2, and decrease the binary exponent.  If the quotient is above the range, multiply the denominator by 2, and increase the binary exponent.

A naive implementation, in Python, is as follows:

```python
def algorithm_m(num, b):
    # Ensure numerator >= 2**52
    bits = int(math.ceil(math.log2(num)))
    if bits <= 53:
        num <<= 53
        b -= 53

    # Track number of steps required (optional).
    steps = 0
    while True:
        steps += 1
        c = num//b
        if c < 2**52:
            b //= 2
        elif c >= 2**53:
            b *= 2
        else:
            break

    return (num, b, steps-1)
```

For example, to scale `s` into the range `[2^52, 2^53)`, we would need a a numerator of `247....`, and a denominator of `10^1078`, requiring 1127 iterations to scale the value. Similarly, scaling `2.4703282292062327e-324` requires 1127 iterations to scale, each in `O(N^2)` time. 

Luckily, Rust uses a simple optimization: a quick estimate of the bitshift required to generate the correct ratio is calculated from the number of bits in the numerator and denominator. Although this optimization  Rust's libcore also uses a [slow division algorithm](https://en.wikipedia.org/wiki/Division_algorithm#Restoring_division), and a slow exponentiation algorithm, scaling poorly for most inputs. 

Lexical's `algorithm_m` feature uses various optimizations to dramatically improve the approach.

1. It requires only 1-2 iterations to scale a value so the resulting quotient is in the range `[2^52, 2^53)`. It does this by multiplying the numerator to be 52-bits larger than the denominator on the first iteration, resulting in a quotient of 52 or 53 bits. If the quotient is 52-bits, multiply the numerator by 2 and repeat.
2. It uses basecase division (an implementation of Knuth's Algorithm D), which is an `O(q*m)` algorithm, requiring only `q` native divisions and `q*m` multiplications and subtractions, where `m` is the number of limbs in the divisor, and `n` is the number of limbs of the numerator, and `q = n - m`. Since we set the numerator to be 52 or 53 bits larger than the denominator, this means `q <= 3`, and therefore the algorithm is `O(m)`. In comparison, restoring division, used by libcore, is `O(n^2)`, and iterates over each bit in the numerator. To put this into perspective, for a numerator/denominator pair with ~2100 bits (1e307), restoring division requires ~140,000 native subtractions and comparisons, compared to ~96 multiplications and subtractions in lexical.
3. It limits the number of parsed digits used when representing the mantissa. Any decimal string for a binary float has a finite representation, after which, only the presence or absence of non-zero digits may affect the mantissa. By removing these trailing digits from the mantissa and only comparing them to '0', lexical provides an upper-bound on the computational cost in representing the significant digits of a float via an arbitrary-precision integer.

**bigcomp**

David M. Gay's `bigcomp` implementation in `dtoa` is the canonical string-to-float parser, and uses another approach for performance, described in depth [here](https://www.exploringbinary.com/bigcomp-deciding-truncated-near-halfway-conversions/). Bigcomp represents `b` as an N-bit integer (24 for f32, 53 for f64) and a binary exponent and calculates `b+h` from `b`. Finally, bigcomp scales `b+h` by a power of 10 such that the calculated value would be from [0, 10), and creates a fraction of big integers. It then calculates the generated digits from `b+h` by iteratively using calculating the quotient (the digit) and remainder of the fraction, setting the numerator to `10*remainder`. Each iteration requires ~1 division operation and `N` multiplication, addition and subtraction operations operations.

When the digits present in `s` or `b+h` differ from each other, the correct representation is determined. Therefore, bigcomp requires `N` division operations and `N^2` multiplication operations, where `N` is the number of native ints in the big integer. Furthermore, since the number of digits in the input string does not affect the size of the numerator/denominator in bigcomp (but does in algorithm M), bigcomp uses less memory in edge cases and has a lower-order `N` for complex inputs. Finally, all halfway representations stop producing digits after 767 digits, requiring only a simple comparison to categorize larger strings. 

**Conclusion**

In practice, this means for a 750 digit input string with a value near 5e-324, theoretically, algorithm M would perform ~250 times as many division/multiplication operations as bigcomp; for a 15,000 digit input string, nearly 100,000 times as many operations. In the worst scenario, for a 1MB string, on x86-64 hardware with a 2.20 GHz CPU, using a correct float parser based off algorithm M could take nearly 6 minutes of computation time, while only 50 μs with bigcomp. Luckily, Rust does not implement a correct parser, avoiding denial-of-service attacks based off float parsing, however, it does so at the expense of correctness. Bigcomp both is more correct and more performant in all non-trivial cases, and the optimizations for trivial cases make lexical competitive with other float parsers for any input string.

# Roadmap

Ideally, Lexical's float-parsing algorithm or approach would be incorporated into libcore. Although Lexical greatly improves on Rust's float-parsing algorithm, in its current state it's insufficient to be included in the standard library, including numerous "anti-features":

1. It supports non-decimal radices for float parsing, leading to significant binary bloat and increased code branching, for almost non-existent use-cases.
2. It inlines aggressively, producing significant binary bloat.
3. It makes extensive use of unsafe Rust, potentially hiding serious memory errors or other bugs.
4. It contains dead code for efficient higher-order arbitrary-precision integer algorithms, in case rare use-cases require the use of more complex algorithms.

I would love to contribute lexical back to the Rust community, however, some significant modifications would be needed. If there's a desire by the Rust team to incorporate lexical's float-parsing algorithm into libcore, I would glad re-write lexical, supporting only decimal strings and minimizing the use of unsafe code.

Finally, there is work needed to optimize lexical: ideally, Rust should be able to match the performance of glibc's strtod. To do so, we would swap basecase division with a faster algorithm, only calculating the top 24- or 53-bits of the quotient, while retaining the ability to distinguish halfway cases for normal and denormal floats. Although generating 24- or 53-bits of a quotient is simple, I personally have no idea how to distinguish halfway cases without calculating the remainder, which incurs significant computational expense.

# License

Lexical is dual licensed under the Apache 2.0 license as well as the MIT license. See the LICENCE-MIT and the LICENCE-APACHE files for the licenses.

Lexical also ports some code from [V8](https://github.com/v8/v8), [libgo](https://golang.org/src) and [fpconv](https://github.com/night-shift/fpconv), and therefore might be subject to the terms of a 3-clause BSD license or BSD-like license.

# Contributing

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in lexical by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
