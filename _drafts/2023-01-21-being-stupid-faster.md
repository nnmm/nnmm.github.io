---
layout: post
title:  "Being stupid faster"
date:   2023-01-21 21:00:00 +0000
author: me
categories: rust
tags:   rust perf
---

An exercise in stubbornness.

# Parsing

# Basic solution

* Calculate the cumulative sum of the array elements
* Find the sum of each subarray `&array[i..j]` by subtracting `cs[i]` from `cs[j]`
* Check if the sum is square

## Integer type

The minimum/maximum possible value for `cs` is ±100 * 10⁵ = ±10⁷. That fits into an `i32`, but not an `i16`. So let's just use `i32` for the numbers and their sums.


## Checking that a number is a square

Floating-point numbers are scary – I doubt that `sqrt(n*n)` is guaranteed to return `n`, for instance. Calculating the square root is computationally expensive anyway, so we can surely do better using only integer operations.

For starters, I'll create a set of squares and check if the sum is in it:

```rust
let mut cs = Vec::with_capacity(self.vec.len() + 1);
cs.push(0);
let mut cumsum = 0;
cs.extend(self.vec.into_iter().map(|x| {
    cumsum += i32::from(x);
    cumsum
}));

let mut num_perfect_subarrays = 0;
for i in 0..cs.len() {
    for j in 0..i {
        let sum = cs[i] - cs[j];
        if squares.contains(&sum) {
            num_perfect_subarrays += 1;
        }
    }
}
num_perfect_subarrays
```

That takes 140s on release mode according to `hyperfine`. And 3330s on debug mode.

# Lookup table

```rust
// The number of elements/bits in the LUT. It's 100*100_000 generously
// rounded up to a power of two: 2^16 bits = 8kb.
const SQUARE_LUT_RANGE: usize = 16777216;

// A lookup table. Its size in bytes is its size in bits divided by 8 (right shifted by 3).
static mut SQUARE_LUT: [u8; SQUARE_LUT_RANGE >> 3] = [0; SQUARE_LUT_RANGE >> 3];

fn precompute_squares() {
    for i in 0..=NUM_SQUARES_MAX {
        let sq = i * i;
        unsafe { SQUARE_LUT[sq >> 3] |= 1 << (sq as u8 & 7) };
    }
}

fn check_square(sum: i32) -> bool {
    // This looks wrong, but negative sums will wrap around to a
    // high number that is always > SQUARE_LUT_RANGE.
    let sum = sum as usize;
    if sum < SQUARE_LUT_RANGE {
        unsafe {
            // Get value out of the bitfield, >> 3 is division by 8
            ((SQUARE_LUT[sum >> 3] >> (sum & 7)) & 1) == 1
        }
    } else {
        false
    }
}
```

and in the solve function, of course

```diff
         for i in 0..cs.len() {
             for j in 0..i {
                 let sum = cs[i] - cs[j];
-                if squares.contains(&sum) {
+                if check_square(sum) {
                     num_perfect_subarrays += 1;
                 }
             }
```

This takes 7.133 s ±  0.021 s according to `hyperfine`. What if we replace the array indexing with `cs.get_unchecked(i)`? … huh, 9.633 s ±  0.048 s.


That is probably where I'd normally be happy. It's orders of magnitudes faster and still readable.

But I'm curious if we can go faster. Don't care much if it's just on my computer or everyone's computer. Let's leave safety and sanity behind.

# Some measurement

```
cargo install flamegraph
cargo flamegraph
echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid
```

```
perf stat target/release/perfect_subarray < input.txt

 Performance counter stats for 'target/release/perfect_subarray':

          8.047,98 msec task-clock                #    1,000 CPUs utilized
                46      context-switches          #    5,716 /sec
                 0      cpu-migrations            #    0,000 /sec
               712      page-faults               #   88,469 /sec
    26.543.905.855      cycles                    #    3,298 GHz
   107.785.530.199      instructions              #    4,06  insn per cycle
    31.985.151.619      branches                  #    3,974 G/sec
        27.887.043      branch-misses             #    0,09% of all branches
```

Neat! I believe 4.06 instructions per cycle is really good throughput. For instance, `rustc` compiling my program also ranges between 1.15 (when compiling with optimizations) and 1.23 (when compiling with optimizations).


# How to get faster?

There are a few general-purpose approaches:
* Using threads – not going to do that here though, the point is performance per thread.
* Eliminating branches
  * … by using unchecked indexing operations
  * … by writing branchless code
  * … by unrolling loops
* Tweaking compilation settings
* SIMD
* Inlining or preventing inlining
* Being more cache-friendly
  * Fitting more stuff in less memory
  * Access memory in a more cache-friendly pattern
* Changing the program in random ways

I'll try each one of these in turn and keep the optimizations that help. So, a kind of local hill-climbing optimization. Of course it might be that two or more of these optimizations interact, so that they individually look like they don't help, but together they have a positive impact.

## Compiler settings

What about compiler settings?
* `codegen_units = 1` – no difference. I'll leave it in though, as that means 
* `lto = 1` – no difference
* [Profile Guided Optimization](https://doc.rust-lang.org/rustc/profile-guided-optimization.html) – a bit more than twice as slow ಠಿ_ಠ
* `-C target-cpu=native` – no difference

Huh. I'm okay with things making no difference, but more information in the form of PGO making the generated code so much dumber is crazy.

We're on our own.


## Inlining

* Slapping an `#[inline(never)]` or `#[inline(always)]` on `solve()` doesn't change anything.
* The `checks_square()` function is already inlined, and forbidding inlining makes the code more than twice as slow.
* Factoring out the cumulative sum calculation into a non-inlined function gives a small speed increase! That kinda makes sense. And it also means the assembly code for `solve()` will be more readable.

## `i16` fast path

```rust
fn solve(self) -> usize {
    // Compress is true if we know that a subtraction of any two values will never overflow i16
    let (cs, compress) = self.cumsum();
    if compress {
        let cs: Vec<_> = cs.into_iter().map(|x| x as i16).collect();
        solve_i16(&cs)
    } else {
        solve_i32(&cs)
    }
}
```

That's around 5% slower.

Why? It turns out that it's slower with `codegen-units=1`, but faster with `codegen-units=3`.

I dug around a lot and concluded that at least _part_ of the story is that the number of codegen units can affect the ordering of functions in the executable.

```
gdb -batch -ex "set disassembly-flavor intel" -ex "disassemble/s _ZN16perfect_subarray8TestCase5solve17hf7756a29ab89ebe4E" target/release/perfect_subarray
```

and could see that there are a lot of SIMD instructions in the `i16` version that weren't there before. I think my computer just does not like SIMD maybe? Let's try again with `cargo rustc --release -- -C target-feature=-sse2`.

That makes the SIMD go away, but it's still slow. At second glance, 

I still maintain that it's not a bad idea.


Anyway, I'll revert that.



## Branches

Let me use my brain. Using unchecked indexing operations should definitely be a win, as it should get rid of any branches towards the panicking code for invalid indices.

* Unchecked indexing in `check_square()` – no difference
* Unchecked indexing in `solve()`, for calculating `sum` – much worse. One third worse. Instructions per second dropped to `2,49`. Hwat

Ok. So much for logical thinking. Undoing that.

Less compact lookup table

# TODO
TODO: Check assembly for unchecked indexing
For more detailed metrics, I don't really trust or understand `perf` output. E.g. 

```
            39.939      iTLB-loads
            40.891      iTLB-load-misses          #  102,38% of all iTLB cache accesses
```
