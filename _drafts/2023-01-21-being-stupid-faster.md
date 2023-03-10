---
layout: post
title:  "Being stupid faster"
date:   2023-01-21 21:00:00 +0000
author: me
categories: rust
tags:   rust perf optimization
---

An exercise in stubbornness.

# The problem

A while back, I participated in a coding competition and encountered [a problem](https://codingcompetitions.withgoogle.com/kickstart/round/000000000019ff43/00000000003381cb) where I failed to think of the correct algorithm.

The task is super simple: In an array of integers, count all the contiguous subarrays (of non-zero length) whose sum is a square number.

The brute force approach is

```python
count = 0
for i in 0..len(arr)
    for j in 0..i
        if sum(arr[j..=i]) is square  # `..=` means an inclusive range
            count += 1
```

but I wasn't _that_ stupid. I did this:

```python
cumsum = 0
cs = [0]
for i in 0..len(arr)
    cumsum += arr[i]
    cs.push(cumsum)

count = 0
for i in 0..len(cs)
    for j in 0..i
        if cs[i] - cs[j] is square
            count += 1
```

That's still `O(n²)`, whereas the intended algorithm is `O(n log(n))` or something. On the larger test set, my solution exceeded the time limit imposed by the coding competition, which was 20 seconds.

But I can be stubborn and Rust is fast, so how far can you optimize this dumb quadratic algorithm?

# Test set

The array in the coding competition contained numbers between -100 and 100 inclusive, and had a up to 10⁵ elements. So, for my own local tests, I created an input file that had three short arrays for debugging, and two that had the maximum allowed length: One with random numbers, and one with a pattern like `0, 1, 2, 3, … 98, 99, 100, 99, 98 … -99, -100, -99, …`.

# A baseline solution

First, let's translate the above pseudocode into Rust. I used `i8` for the input array and `i32` for the cumulative sums, since the minimum/maximum possible value for `cs` is ±100 * 10⁵ = ±10⁷. That fits into an `i32`, but not an `i16`.

```rust
let array: Vec<i8> = parse_input();
let mut cs = Vec::with_capacity(array.len() + 1);
cs.push(0);
let mut cumsum = 0;
cs.extend(array.into_iter().map(|x| {
    cumsum += i32::from(x);
    cumsum
}));

let mut num_perfect_subarrays = 0;
for i in 0..cs.len() {
    for j in 0..i {
        let sum = cs[i] - cs[j];
        if is_square(sum) {
            num_perfect_subarrays += 1;
        }
    }
}
```

The crucial question is, what's a good way to check whether a number is square?

You can convert the number to a floating-point number and take the square root, checking if it is an integer. That seems like a bad idea to me, I doubt that `sqrt(n*n)` is guaranteed to return `n`, for instance. Also, large ints cannot be represented exactly by floats, so I'd have to check to be sure. Calculating the square root is computationally expensive anyway, so we can surely do better using only integer operations.

The algorithms for calculating the integer square root are all iterative, so that sounds complicated (you couldn't use external crates in the coding competition) and computationally heavy too. We have a `O(1)` solution though, the good old hash set.

```rust
fn is_square(sum: i32, squares_map: &HashSet<i32>) {
    // squares_map can be precomputed once, in main,
    // and needs to be passed into this function
    squares_map.contains(&sum)    
}
```

That will surely be blazing fast. Oh wait it's not: that takes 140s in release mode according to `hyperfine`. And 3330s in debug mode.

We'll skip the part where I use a more suitable hash function and jump right to the …

## Lookup table

We can just create a table of values indicating whether a number is a square. And of course we should make it a compressed table, also called bitmap or bitset, that stores 8 _is-this-square_ values per byte.

To avoid passing around a reference to the table, I'll make it a static variable.

```rust
// The last number whose square is below 100*100_000
const NUM_SQUARES_MAX: usize = 3162;

// The number of elements/bits in the LUT. It's 100*100_000 generously
// rounded up to a power of two: 2^24 bits = 2^21 bytes = 2MB.
const SQUARE_LUT_RANGE: usize = 16777216;

// A lookup table. Its size in bytes is its size in bits divided by 8 (right shifted by 3).
static mut SQUARE_LUT: [u8; SQUARE_LUT_RANGE >> 3] = [0; SQUARE_LUT_RANGE >> 3];

fn precompute_squares() {
    for i in 0..=NUM_SQUARES_MAX {
        let sq = i * i;
        unsafe { SQUARE_LUT[sq >> 3] |= 1 << (sq as u8 & 7) };
    }
}

fn is_square(sum: i32) -> bool {
    // This looks wrong, but negative sums will wrap around to a
    // high number that is always > SQUARE_LUT_RANGE.
    // You could instead also check if sum >= 0 before casting to usize.
    let sum = sum as usize;
    if sum < SQUARE_LUT_RANGE {
        unsafe {
            // Get value out of the bitset
            ((SQUARE_LUT[sum >> 3] >> (sum & 7)) & 1) == 1
        }
    } else {
        false
    }
}
```

This takes 7.133 s ±  0.021 s according to `hyperfine`. Cool cool cool! That is where I'd normally be happy and leave it be. It's orders of magnitudes faster, still quite readable, and also uses no `unsafe` except for the static variable.

But I'm curious if we can go faster. Don't care if it's just on my computer. Let's leave safety and sanity behind.

# What's the plan?

There are a few general-purpose approaches:

* Using threads – not going to do that here though, the point is performance per thread.
* Eliminating branches
  * … by using unchecked indexing operations
  * … by writing branchless code
  * … by unrolling loops
* Tweaking compilation settings
* Doing less work, e.g. by
  * … preventing unnecessary copies and conversions
  * … performing fewer overall memory accesses
* SIMD/autovectorization
* Inlining or preventing inlining
* Being more cache-friendly
  * Fitting more stuff in less memory
  * Access memory in a more cache-friendly pattern
* Changing the program in random ways :)

I'll try each one of these in turn and keep the optimizations that help. So, a kind of local hill-climbing optimization. Of course it might be that two or more of these optimizations interact, so that they individually look like they don't help, but together they have a positive impact.

Of course we also have a flamegraph, but it's not very interesting. I'm also not sure how much it is distorted by [skid](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid) and optimizations.

![flamegraph](/assets/img/flamegraph.svg)

## Inlining

* Slapping an `#[inline(never)]` or `#[inline(always)]` on `solve()` doesn't change anything.
* The `checks_square()` function is already inlined, and forbidding inlining makes the code more than twice as slow.
* Factoring out the cumulative sum calculation into a non-inlined function gives a small speed increase! That kinda makes sense. And it also means the assembly code for `solve()` will be more readable.


# Starting small: unchecked indexing

What if we replace the array indexing with `cs.get_unchecked(i)`? It's a small change and we'll do more interesting stuff later, but you can't go wrong with this one: We index into `cs` often, and I'm not sure if Rust was able to eliminate the bounds checks, so explicitly using the no-bounds-check version will either not change anything, or improve things.

And just like that, we went from 7.133 s to … huh, 9.633 s ±  0.048 s.

* Unchecked indexing in `check_square()` – no difference
* Unchecked indexing in `solve()`, for calculating `sum` – much worse. One third worse. Instructions per second dropped to `2,49`. Hwat

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

Neat! I believe 4.06 instructions per cycle is really good throughput. For instance, `rustc` compiling my program also ranges between 1.15 (when compiling with optimizations) and 1.23 (when compiling with optimizations). `bazel query` is at 0.72.



## Compiler settings

What about compiler settings?
* `codegen_units = 1` – no difference. I'll leave it in though, as that means 
* `lto = 1` – no difference
* [Profile Guided Optimization](https://doc.rust-lang.org/rustc/profile-guided-optimization.html) – a bit more than twice as slow ಠಿ_ಠ
* `-C target-cpu=native` – no difference

Huh. I'm okay with things making no difference, but more information in the form of PGO making the generated code so much dumber is crazy.

We're on our own.


## Impossible last digits 

I was first confronted with this idea in [this StackOverflow question](https://stackoverflow.com/questions/295579/fastest-way-to-determine-if-an-integers-square-root-is-an-integer).

In multiplication, the n-th digit from the right in the result is only affected by the n last digits in the input. Specifically for binary multiplication, an output bit depends only on input bits to the right, but not to the left. So let's look at the possible last bits (or binary digits) for the three last digits:

```
|-----|--------------------|
|  i  | last digits of i*i |
|-----|--------------------|
| 000 |                000 |
| 001 |                001 |
| 010 |                100 |
| 011 |                001 |
| 100 |                000 |
| 101 |                001 |
| 110 |                100 |
| 111 |                001 |
|-----|--------------------|
```

Obviously, a square number can only end in `000`, `001`, or `100`, and all numbers whose last three bits are different can be excluded. You can do the same for a higher number of bits to be able to exclude even more numbers. For instance, the same table for four digits tells you that `1000` and `1100` can also be excluded.

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




## Obligatory handwritten SIMD

That function is looking thick. Solid. Tight.


* https://lemire.me/blog/2018/09/07/avx-512-when-and-how-to-use-these-new-instructions/


# TODO
TODO: Check assembly for unchecked indexing
For more detailed metrics, I don't really trust or understand `perf` output. E.g. 

```
            39.939      iTLB-loads
            40.891      iTLB-load-misses          #  102,38% of all iTLB cache accesses
```

You can right away forget about:
* pulling the indexing with `i` out of the inner loop
* use unchecked indexing operations

