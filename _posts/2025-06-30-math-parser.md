---
title: "Optimizing a Math Expression Parser in Rust"
layout: post
date: 2025-06-30 08:00
image: /assets/images/1brc/flamegraph1.jpg
headerImage: false
tag:
- algorithms
- data structures
- rust
- performance
- multithreading
- optimization
- bitwise
- hash maps
category: blog
author: Ricardo Pallas
projects: true
description: Optimizing a math expression parser for speed and memory.
---

# Optimizing a Math Expression Parser in Rust


## Table of contents

1. [Baseline implementation](#baseline-implementation)
    1. [How it works](#how-it-works)
    1. [Parser Example: `(1 + 2) - 3`](#parser-example-1--2---3)
    1. [Sequence Diagram](#sequence-diagram)
    1. [It works! But we can do better](#it-works-but-we-can-do-better)
1. [Optimizations for speed and memory](#optimizations-for-speed-and-memory)
    1. [Optimization 1: Do not allocate a Vector when tokenizing](#optimization-1-do-not-allocate-a-vector-when-tokenizing)
    1. [Optimization 2: Zero allocations — parse directly from the input bytes](#optimization-2-zero-allocations--parse-directly-from-the-input-bytes)
    1. [Optimization 3: Do not use `Peekable`](#optimization-3-do-not-use-peekable)
    1. [Optimization 4: Multithreading and SIMD](#optimization-4-multithreading-and-simd)

---

In a [previous post](https://rpallas.xyz/1brc/) I explored how to optimize file parsing for max speed. This time, we’ll look at different and self-contained problem: writing a math expression parser in Rust, and making it as fast and memory-efficient as possible.

Let’s say we want to parse simple math expressions with addition, subtraction, and parentheses. For example:

```
4 + 5 + 2 - 1       => 10
(4 + 5) - (2 + 1)   => 6
(1 + (2 + 3)) - 4   => 2
```

We’ll start with a straightforward implementation and optimizite it step by step.

---

## Baseline implementation

Here’s the first version of our parser:

```rust
use std::fs;
use std::io::Result;
use std::{iter::Peekable, time::Instant};

fn main() -> Result<()> {
    let total_start = Instant::now();
    let mut step_start = Instant::now();

    let input = read_input_file()?;
   
    println!("Step 1: Input file read in {:?}", step_start.elapsed());

    step_start = Instant::now();
    
    let result = eval(&input);
    
    println!(
        "Step 2: Calculation completed in {:?}",
        step_start.elapsed()
    );

    let total_duration = total_start.elapsed();
    
    println!("--- Summary ---");
    println!("Result: {}", result);
    println!("Total time: {:?}", total_duration);

    Ok(())
}

fn read_input_file() -> Result<String> {
    fs::read_to_string("data/input.txt")
}

fn eval(input: &str) -> u32 {
    let mut tokens = tokenize(input).into_iter().peekable();
    parse_expression(&mut tokens)
}

fn tokenize(input: &str) -> Vec<Token> {
    input
        .split_whitespace()
        .map(|s| match s {
            "+" => Token::Plus,
            "-" => Token::Minus,
            "(" => Token::OpeningParenthesis,
            ")" => Token::ClosingParenthesis,
            n => Token::Operand(n.parse().unwrap()),
        })
        .collect()
}

fn parse_expression(tokens: &mut Peekable<impl Iterator<Item = Token>>) -> u32 {
    let mut left = parse_primary(tokens);

    while let Some(Token::Plus) | Some(Token::Minus) = tokens.peek() {
        let operator: Option<Token> = tokens.next();
        let right = parse_primary(tokens);
        left = match operator {
            Some(Token::Plus) => left + right,
            Some(Token::Minus) => left - right,
            other => panic!("Expected operator, got {:?}", other),
        }
    }

    return left;
}

fn parse_primary(tokens: &mut Peekable<impl Iterator<Item = Token>>) -> u32 {
    match tokens.peek() {
        Some(Token::OpeningParenthesis) => {
            tokens.next(); // consume '('
            let val = parse_expression(tokens);
            tokens.next(); // consume ')'
        }
        _ => parse_operand(tokens),
    }
}

fn parse_operand(tokens: &mut Peekable<impl Iterator<Item = Token>>) -> u32 {
    match tokens.next() {
        Some(Token::Operand(n)) => n,
        other => panic!("Expected number, got {:?}", other),
    }
}

#[derive(Debug, Clone, PartialEq)]
enum Token {
    Operand(u32),
    Plus,
    Minus,
    OpeningParenthesis,
    ClosingParenthesis,
}
```

---

### How it works

Let’s break it down.

The program reads from a file called `input.txt`, which contains a math expression in a single line. That expression is passed to the `eval()` function.

The `tokenize()` function walks through the input string, splits it by whitespace, and maps each part into a token. For example, this input:

```
7 - 3 + 1
```

...is turned into this list of tokens:

```
[Operand(7), Minus, Operand(3), Plus, Operand(1)]
```

The parser then uses a simple recursive strategy:

- `parse_expression()` handles sequences of additions and subtractions.
- `parse_primary()` handles numbers and expressions inside parentheses.
- `parse_operand()` handles the actual integer values.

The recursive call to `parse_expression()` inside `parse_primary()` allows us to evaluate nested expressions (parentheses).

---

### Parser Example: `(1 + 2) - 3`

Let’s walk through parsing the expression `(1 + 2) - 3` using our functions:

```rust
fn eval(input: &str) -> u32 {
    let mut tokens = tokenize(input).peekable();
    parse_expression(&mut tokens)
}
```

**Input string:** `(1 + 2) - 3`

**Tokens with index:**

| Index | Token                    |
|-------|--------------------------|
| 0     | OpeningParenthesis `(`   |
| 1     | Operand(1)               |
| 2     | Plus `+`                 |
| 3     | Operand(2)               |
| 4     | ClosingParenthesis `)`   |
| 5     | Minus `-`                |
| 6     | Operand(3)               |

We begin by calling `parse_expression` at **depth 1**:

1. **`parse_expression` (depth 1)**  
   - Calls `parse_primary` to get the first value.  

2. **`parse_primary` (depth 2)**  
   - Sees `OpeningParenthesis` at index 0.  
   - Consumes `(` and calls **`parse_expression` (depth 3)** for the parenthesized subexpression.

3. **`parse_expression` (depth 3)**  
   - Calls `parse_primary` (depth 4).

4. **`parse_primary` (depth 4)**  
   - Sees `Operand(1)` at index 1.  
   - Calls `parse_operand` (depth 5), which consumes index 1 and returns `1`.

5. **`parse_expression` (depth 3)** (resuming)  
   - Sees `Plus` at index 2.  
   - Consumes `+` and calls `parse_primary` (depth 4) again.

6. **`parse_primary` (depth 4)**  
   - Sees `Operand(2)` at index 3.  
   - Calls `parse_operand` (depth 5), which consumes index 3 and returns `2`.

7. **`parse_expression` (depth 3)**  
   - Combines `1 + 2 = 3`.  
   - Returns `3` to the caller at depth 2.

8. **`parse_primary` (depth 2)** (resuming)  
   - Now at index 4 sees `ClosingParenthesis`.  
   - Consumes `)` and returns the inner value `3`.

9. **`parse_expression` (depth 1)** (resuming)  
   - Left side is `3`.  
   - Sees `Minus` at index 5.  
   - Consumes `-` and calls `parse_primary` (depth 2).

10. **`parse_primary` (depth 2)**  
    - Sees `Operand(3)` at index 6.  
    - Calls `parse_operand` (depth 5), consumes it, and returns `3`.  

11. **`parse_expression` (depth 1)**  
    - Computes `3 - 3 = 0`.  
    - No more operators, so it returns `0` as the final result.

---

### It works! But we can do better

This baseline parser works well, but it's not optimized.

If we compile it in release mode and execute it for the test file of **1.6GB**, it takes **43.87 seconds** to execute on my laptop:

```
Step 1: Input file read in 1.189915008s
Step 2: Calculation completed in 41.876205675s

--- Summary ---
Result: 2652
Total time: 43.06795088s

```

In the next sections, we’ll go step by step to optimize this parser:

1. Avoid allocating the token list.
2. Use iterators and slices instead of `Vec<Token>`.
3. Parse directly from the input string.

The goal is to make a parser that’s not only simple and correct, but also lean and fast.

Let’s go!

---

## Optimizations for speed and memory

### Optimization 1: Do not allocate a Vector when tokenizing

Let's use [cargo flamegraph](https://github.com/brendangregg/FlameGraph) to visualize the stack of the current solution to know what we can start optimizing.

`cargo flamegraph --dev --bin parser`

We get the following flame graph:

![First flamegraph](../assets/images/math_parser/flamegraph_1_naive_solution.png)

We can see that the majority of the time is spent in the tokenizer function, which reads the input string and allocates a vector of tokens.

To profile memory usage, we can use [dhat](https://github.com/nnethercote/dhat-rs) to generate a profile JSON file and view it at https://nnethercote.github.io/dh_view/dh_view.html:

![First memory profiling](../assets/images/math_parser/memory_1_naive_solution.png)

Notice how **4 GB of RAM** is used just to allocate the token vector!

---


I made a mistake in my initial implementation. Why does the `tokenize` function return a vector if we’re converting it into an iterator later anyway? Let’s just return a lazy iterator directly instead of allocating a vector:

```rust
fn eval(input: &str) -> u32 {
    let mut tokens = tokenize(input).peekable();
    parse_expression(&mut tokens)
}

fn tokenize(input: &str) -> impl Iterator<Item = Token> + '_ {
    input.split_whitespace().map(|s| match s {
        "+" => Token::Plus,
        "-" => Token::Minus,
        "(" => Token::OpeningParenthesis,
        ")" => Token::ClosingParenthesis,
        n => Token::Operand(n.parse().unwrap()),
    })
}
```

If we run the parser again after this small change, the speed improves significantly:

```
Step 1: Input file read in 1.249408413s
Step 2: Calculation completed in 5.204344393s

--- Summary ---
Result: 2652
Total time: 6.45377661s
```

**Wow! From 43 seconds down to just 6.45**. What an improvement. A small mistake can have a huge impact on performance. Fortunately, the flamegraph pointed us straight to the bottleneck!

---

### Optimization 2: Zero allocations — parse directly from the input bytes

After removing the initial `Vec<Token>` allocation, performance improved significantly. But we can still do better.

If we analyze the flamegraph again, we notice that although we no longer allocate a vector of tokens, we’re still splitting the input string by whitespace and allocating temporary &str slices during parsing:

![Second flamegraph](../assets/images/math_parser/flamegraph_2.png)

The boxes marked in pink/violet correspond to the `split_whitespaces` funciton used by our tokenizer:

```rust
fn tokenize(input: &str) -> impl Iterator<Item = Token> + '_ {
    input.split_whitespace().map(|s| match s {
        "+" => Token::Plus,
        "-" => Token::Minus,
        "(" => Token::OpeningParenthesis,
        ")" => Token::ClosingParenthesis,
        n => Token::Operand(n.parse().unwrap()),
    })
}
```

We're paying a cost for each `split_whitespace` call and every, which allocates intermediate slices. This churns memory and CPU cycles.

Let’s go even lower level.

#### The idea: Use &[u8]

Instead of working with UTF-8 strings and &str, we can use raw bytes (&[u8]) and manually scan for digits and operators, to avoid temporary string allocations.


Here is our new zero-memory allocation tokenizer:

```rust
fn read_input_file() -> Result<Vec<u8>> {
    fs::read("data/input.txt")
}

struct Tokenizer<'a> {
    input: &'a [u8],
    pos: usize,
}

impl<'a> Iterator for Tokenizer<'a> {
    type Item = Token;

    fn next(&mut self) -> Option<Self::Item> {
        if self.pos >= self.input.len() {
            return None;
        }

        let byte = self.input[self.pos];

        self.pos += 1;

        let token = match byte {
            b'+' => Some(Token::Plus),
            b'-' => Some(Token::Minus),
            b'(' => Some(Token::OpeningParenthesis),
            b')' => Some(Token::ClosingParenthesis),
            b'0'..=b'9' => {
                let mut value = byte - b'0';
                while self.pos < self.input.len() && self.input[self.pos].is_ascii_digit() {
                    value = 10 * value + (self.input[self.pos] - b'0');
                    self.pos += 1;
                }

                Some(Token::Operand(value))
            }
            other => panic!("Unexpected byte: '{}'", other as char),
        };

        self.pos += 1; // skip whitespace

        return token;
    }
}
```

The only heap allocation occurs when the file is read into a vector. The tokenizer operates on references to that vector of bytes and does not perform any intermediate allocations.

If we execute the program again, we get:

```
Step 1: Input file read in 1.212080967s
Step 2: Calculation completed in 2.471639289s

--- Summary ---
Result: 2652
Total time: 3.683753465s
```

Great improvement! From **6.45 to 3.68 seconds**. Almost 2 seconds faster!
---


### Optimization 3: Do not use `Peekable`

The new flamegraph shows several samples related to `Peekable`:

- `core::iter::adapters::peekable::Peekable::peek::_{{closure}}`
- `core::iter::adapters::peekable::Peekable::peek`

![Third flamegraph](../assets/images/math_parser/flamegraph_2.png)


This is because we wrap our tokenizer into Rust’s `Peekable`, which allows us to look at the next token without consuming it. We initially used it to look ahead when parsing expressions like `1 + (2 - 3)` to know whether to continue parsing or return early.

However, in our use case, `peek()` isn't necessary. We can restructure the algorithm to work directly with a plain iterator.

Here's the old version:

```rust
fn parse_expression(tokens: &mut Peekable<impl Iterator<Item = Token>>) -> u32 {
    let mut left = parse_primary(tokens);

    while let Some(Token::Plus) | Some(Token::Minus) = tokens.peek() {
        let operator = tokens.next();
        let right = parse_primary(tokens);
        left = match operator {
            Some(Token::Plus) => left + right,
            Some(Token::Minus) => left - right,
            other => panic!("Expected operator, got {:?}", other),
        };
    }

    left
}
```

And here’s the new version that eliminates `Peekable`:

```rust
fn parse_expression(tokens: &mut impl Iterator<Item = Token>) -> u32 {
    let mut left = parse_primary(tokens);

    while let Some(token) = tokens.next() {
        if token == Token::ClosingParenthesis {
            break;
        }

        let right = parse_primary(tokens);
        left = match token {
            Token::Plus => left + right,
            Token::Minus => left - right,
            other => panic!("Expected operator, got {:?}", other),
        };
    }

    left
}
```

We replaced the `peek()` logic with a `match` on the current token. If it’s a `+` or `-`, we consume the right-hand operand and compute the result. If it’s a closing parenthesis, we `break` (this is an important point: **we no longer manually skip the closing parenthesis after parsing a sub-expression**).

Previously, with `peekable`, we consumed the `(`, parsed the sub-expression, and then had to explicitly `next()` again to discard the `)` after the recursive call. Now, since we're using a flat iterator, we simply let the closing `)` token be returned by `next()`, and our `while let Some(token)` loop handles it. If the token is a `)`, we break out of the loop, and the recursive call returns.

We also simplified `parse_primary` in a similar way:

```rust
fn parse_primary(tokens: &mut impl Iterator<Item = Token>) -> u32 {
    match tokens.next() {
        Some(Token::OpeningParenthesis) => {
            let val = parse_expression(tokens);
            val
        }
        Some(Token::Operand(n)) => n as u32,
        other => panic!("Expected number, got {:?}", other),
    }
}
```

By avoiding `peek()` and handling the tokens linearly, we improve the performance:

```
Step 1: Input file read in 1.116952011s
Step 2: Calculation completed in 2.094806113s

--- Summary ---
Result: 2652
Total time: 3.21178544s
```

From **3.68 to 3.21 seconds**. We are getting faster. Let's keep optimizing!

---

### Optimization 4: Multithreading and SIMD

The next logical step is to parallelize the computation. Ideally, if we have a CPU with 8 cores, we want to split our giant input file into 8 perfectly equal chunks and have each core work on one chunk simultaneously. This should, in theory, make our program up to 8 times faster.

However, this is not as simple as just cutting the file into 8 pieces. We are bound by the rules of mathematics and syntax, which introduce two critical restrictions:

1.  **We cannot split inside parentheses.** A split can only happen at the "top level" (depth 0) of the expression. Splitting `(1 + 2|)` is invalid.
2.  **We cannot split at a `-` operator.** This is a subtle but crucial mathematical point. Addition is *associative*, meaning `(a + b) + c` is the same as `a + (b + c)`. This property allows us to group additions freely. Subtraction is *not* associative: `(a - b) - c` is not the same as `a - (b - c)`. Splitting on a `-` would change the order of operations and produce the wrong answer.

These restrictions mean we can't just split the file at `(total_size / 8)`. We need an intelligent way to find the *closest valid split point* (a `+` sign at depth 0) to that ideal boundary.

A naive scan to do this would be slow, requiring a full pass over the data just to find the split points before we even start the real work. So, isn't this solution slower (2 passes vs 1 pass)? Not necessarily. We can make the first pass incredibly fast if we use **SIMD**.

#### The Overall Data Flow

Before diving into the code, let's look at the high-level plan. The entire process is orchestrated by our `parallel_eval` function, which follows this data flow:

```
[ Input File ]
      |
      v
.-----------------------.
|     parallel_eval     |
'-----------------------'
      |
      | 1. Find Splits
      v
.----------------------------------.
| find_best_split_indices_simd     |------> [ Split Indices ]
'----------------------------------'             |
      |                                          |
      | 2. Create Chunks                         |
      v                                          |
[ Chunk 1 ] [ Chunk 2 ] ... [ Chunk N ] <--------+
      |         |               |
      |         |               | 3. Process in Parallel
      v         v               v
.------------------------------------.
|         Rayon Thread Pool          |
|                                    |
|  eval(c1)  eval(c2) ...  eval(cN)  |
'------------------------------------'
      |
      | 4. Collect Results
      v
[ Result 1, Result 2, ... Result N ]
      |
      | 5. Sum Results
      v
[ Final Answer ]
```

#### What is SIMD? A Quick Introduction

**SIMD** stands for **S**ingle **I**nstruction, **M**ultiple **D**ata. It's a powerful feature built into virtually all modern CPUs, from laptops to servers to mobile phones. At its core, SIMD allows the CPU to perform the same operation on multiple pieces of data *at the same time*, with a single instruction.

Think of a cashier at a grocery store. A traditional CPU core operates like a cashier scanning items one by one. This is a **scalar** operation—one instruction processes one piece of data.

```
Scalar Operation (One by one)
Instruction: Is this byte '+'?
      |
      V
[ H | e | l | l | o |   | + |   | W | o | r | l | d ]
  ^--- Processed sequentially --->
```

A SIMD-enabled CPU is like a cashier with a giant, wide scanner that can read the barcodes of an entire row of items in the cart all at once. This is a **vector** operation.

```
SIMD Operation (All at once)
Instruction: For all 64 of these bytes, tell me which ones are '+'?
      |
      V
[ H | e | l | l | o |   | + |   | W | o | r | l | d | ... (up to 64 bytes) ]
[ 0 | 0 | 0 | 0 | 0 | 0 | 1 | 0 | 0 | 0 | 0 | 0 | 0 | ... (result mask)  ]
\_______________________________________________________________________/
                         Processed in a single cycle
```

For highly repetitive tasks, like searching for a specific character in a massive string, the performance gain is enormous.

##### A Concrete Example: Finding `+`

In our project, we need to find all the `+` characters.

**The Scalar Way:**
Without SIMD, we would have to write a simple `for` loop to check every single byte:

```rust
let mut positions = Vec::new();
for (i, &byte) in input.iter().enumerate() {
    if byte == b'+' {
        positions.push(i);
    }
}
```
This is simple and correct, but for a 2GB file, this loop runs 2 billion times.

**The SIMD Way:**
With SIMD (specifically, the AVX-512 instructions we use), the process is different:

1.  **Load:** We load a big chunk of our input string (64 bytes at a time) into a special, wide 512-bit CPU register.
2.  **Compare:** We use a single instruction (`_mm512_cmpeq_epi8_mask`) to compare all 64 bytes in our register against a template register that contains 64 copies of the `+` character.
3.  **Get Mask:** The CPU returns a single 64-bit integer (`u64`) as a result. This is a **bitmask**. If the 5th bit of this integer is `1`, it means the 5th byte of our input chunk was a `+`.

In one instruction, we've done the work of 64 loop iterations. While using SIMD requires more complex code, this massive reduction in the number of instructions is the key to unlocking the next level of performance for data-intensive tasks.

#### The Code

Here are the two key functions that implement our parallel strategy. First, `parallel_eval` orchestrates the process, and second, the `find_best_split_indices_simd` function uses SIMD to find the valid split points.

```rust
fn parallel_eval(input: &[u8], num_threads: usize) -> i64 {
    if num_threads <= 1 || input.len() < 1000 {
        return eval(input);
    }

    // 1. Find the best places to split the input.
    let split_indices = unsafe { find_best_split_indices_simd(input, num_threads - 1) };

    // If we couldn't find any valid split points, fall back to single-threaded.
    if split_indices.is_empty() {
        return eval(input);
    }

    // 2. Create the chunks based on the indices.
    let mut chunks = Vec::with_capacity(num_threads);
    let mut last_idx = 0;
    for &idx in &split_indices {
        // Slice from the last index to just before the operator's space.
        chunks.push(&input[last_idx..idx - 1]);
        // The next chunk starts after the operator and its space.
        last_idx = idx + 2;
    }
    chunks.push(&input[last_idx..]);

    // 3. Process all chunks in parallel with Rayon.
    let chunk_results: Vec<i64> = chunks.par_iter().map(|&chunk| eval(chunk)).collect();

    // 4. Since we only split on '+', the final result is the sum of all parts.
    chunk_results.into_iter().sum()
}

#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx512f")]
#[target_feature(enable = "avx512bw")]
unsafe fn find_best_split_indices_simd(input: &[u8], num_splits: usize) -> Vec<usize> {
    let mut final_indices = Vec::with_capacity(num_splits);
    if num_splits == 0 {
        return final_indices;
    }

    let chunk_size = input.len() / (num_splits + 1);
    let mut target_idx = 1;
    let mut last_op_at_depth_zero = 0;
    let mut depth: i32 = 0;
    let mut i = 0;
    let len = input.len();

    let open_parens = _mm512_set1_epi8(b'(' as i8);
    let close_parens = _mm512_set1_epi8(b')' as i8);
    let pluses = _mm512_set1_epi8(b'+' as i8);

    'outer: while i + 64 <= len {
        if final_indices.len() >= num_splits {
            break;
        }
        let chunk = _mm512_loadu_si512(input.as_ptr().add(i) as *const _);
        let open_mask = _mm512_cmpeq_epi8_mask(chunk, open_parens);
        let close_mask = _mm512_cmpeq_epi8_mask(chunk, close_parens);
        let plus_mask = _mm512_cmpeq_epi8_mask(chunk, pluses);

        let mut all_interesting_mask = open_mask | close_mask | plus_mask;

        while all_interesting_mask != 0 {
            let j = all_interesting_mask.trailing_zeros() as usize;
            let current_idx = i + j;
            if (open_mask >> j) & 1 == 1 {
                depth += 1;
            } else if (close_mask >> j) & 1 == 1 {
                depth -= 1;
            } else { // Is a '+' operator
                if depth == 0 {
                    last_op_at_depth_zero = current_idx;
                    let ideal_pos = target_idx * chunk_size;
                    if current_idx >= ideal_pos {
                        final_indices.push(current_idx);
                        target_idx += 1;
                        if final_indices.len() >= num_splits {
                            break 'outer;
                        }
                    }
                }
            }
            all_interesting_mask &= all_interesting_mask - 1;
        }
        i += 64;
    }

    // ... scalar remainder and fill logic ...
    while i < len && final_indices.len() < num_splits {
        let char_byte = *input.get_unchecked(i);
        if char_byte == b'(' { depth += 1; }
        else if char_byte == b')' { depth -= 1; }
        else if char_byte == b'+' && depth == 0 {
            last_op_at_depth_zero = i;
            let ideal_pos = target_idx * chunk_size;
            if i >= ideal_pos {
                final_indices.push(i);
                target_idx += 1;
            }
        }
        i += 1;
    }
    while final_indices.len() < num_splits && last_op_at_depth_zero > 0 {
        final_indices.push(last_op_at_depth_zero);
    }
    final_indices
}
```

#### Algorithm Breakdown: `find_best_split_indices_simd`

This function's job is to find the best `+` signs to split on. It combines the raw speed of SIMD with a small amount of serial logic.

##### Step 1: The SIMD Scan

The code enters a main loop, processing the input in 64-byte chunks. Inside, it uses `_mm512_cmpeq_epi8_mask` to generate bitmasks. This instruction compares all 64 bytes of the chunk against a target character and returns a 64-bit integer (`u64`) where the N-th bit is `1` if the N-th byte was a match.

##### Step 2: The Fast Serial Scan with Bit-Hacking

Next, we combine these masks and loop through only the interesting bits. This is the most clever part of the algorithm.

```rust
let mut all_interesting_mask = open_mask | close_mask | plus_mask;

while all_interesting_mask != 0 {
    let j = all_interesting_mask.trailing_zeros() as usize;
    let current_idx = i + j; // + j because the mask is a u64 little endian, so trailing zeros are the leading 0 in reality
    if (open_mask >> j) & 1 == 1 {
        depth += 1;
    } else if (close_mask >> j) & 1 == 1 {
        depth -= 1;
    } else {
        if depth == 0 {
            last_op_at_depth_zero = current_idx;
            if current_idx >= ideal_pos {
                final_indices.push(current_idx);
                target_idx += 1;
                if final_indices.len() >= num_splits {
                    break 'outer;
                }
            }
        }
    }
    all_interesting_mask &= all_interesting_mask - 1;
}
```

This loop does **not** run 64 times. It only runs for the number of set bits in `all_interesting_mask`. To understand how it processes characters from left-to-right, we need to look at two key details:

1.  **`trailing_zeros()` and Little-Endian:** You might think `trailing_zeros` starts from the *end* of the string, but it's the opposite. Modern x86-64 CPUs are **little-endian**. When a block of memory is loaded into a large integer register, the first byte in memory (e.g., `chunk[0]`) becomes the least significant byte (LSB) of the integer. The `trailing_zeros()` instruction counts from this LSB. Therefore, it always finds the set bit that corresponds to the character with the **lowest index** in our chunk.

    ```
    Memory (Bytes in a chunk):
      Byte Index:   0   1   2   3   ...   63
      Content:     '(' '1' '+' '2'  ...   'X'

          |
          |  Load into a 64-bit integer
          v

    Resulting u64 Bitmask:
      Bit Position:  63  ...   3   2   1   0   <-- LSB (The "trailing" end)
      Corresponds to: 'X' ...  '2' '+' '1' '('
    ```
    As you can see, `trailing_zeros` starts from the right of the integer, which corresponds to the left of our string chunk.


2.  **`    if (open_mask >> j) & 1 == 1 {`**: This is just to check if there is an open parenthesis at position `j`. If so, we increment our counter `depth`.

3.  **`all_interesting_mask &= all_interesting_mask - 1`**: This is a classic bit-hacking trick that clears the lowest set bit we just found. On the next iteration, `trailing_zeros` finds the *new* lowest set bit, which corresponds to the character at the *next lowest index*.

This combination allows us to visit every interesting character in the correct, forward order, but without a slow byte-by-byte scan. Inside the loop, we just update our `depth` counter and check our splitting logic.

#### Putting It All Together: A Concrete Example

Let's trace the entire flow with a small, concrete example.

*   **Input String:** `(1-2) + (3-4) + (5-6)` (Length is 23 bytes)
*   **Goal:** Find 1 split point (`num_splits = 1`) to create 2 chunks.
*   **Ideal Split Position:** `1 * (23 / 2) = 11`. We are looking for the first `+` at depth 0 at or after byte 11.

##### Part 1: `find_best_split_indices_simd` runs

The function will scan the input to find the best split point.

```
Input:        ( 1 - 2 )   +   ( 3 - 4 )   +   ( 5 - 6 )
Index:        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2
                                  1 1 1 1 1 1 1 1 1 1 2 2 2
Depth:        1 1 1 1 1 0 0 0 1 1 1 1 1 0 0 0 1 1 1 1 1 0 0
Ideal Split ->                      ^
```

1.  The code starts scanning. It finds the first `+` at index `7`.
2.  It checks the depth. The `(` at index 0 increased depth to 1, and the `)` at index 5 decreased it back to 0. So, at index 7, `depth == 0`.
3.  It checks the splitting logic: `is current_idx (7) >= ideal_pos (11)?` The answer is **No**. The code continues scanning.
4.  The code finds the next `+` at index `15`.
5.  It checks the depth. The `(` at index 9 and `)` at index 13 have kept the depth at 0.
6.  It checks the splitting logic: `is current_idx (15) >= ideal_pos (11)?` The answer is **Yes!**
7.  **Action:** The code pushes `15` into `final_indices` and immediately breaks out of all loops because it has found the 1 split it was looking for.
8.  The function returns `[15]`.

##### Part 2: `parallel_eval` receives the result

1.  `split_indices` is now `[15]`.
2.  The `for` loop runs once for the index `15`.
    *   It creates the first chunk by slicing from `0` to `15 - 1 = 14`. The chunk is `(1-2) + (3-4)`.
    *   It updates `last_idx` to `15 + 2 = 17`.
3.  The loop finishes. It creates the final chunk by slicing from `17` to the end. The chunk is `(5-6)`.

    ```
    Original:     (1-2) + (3-4)   +   (5-6)
                  <-- chunk 1 -->   <-- chunk 2 -->

    Split Index:                    ^ (15)
    ```

4.  The two chunks, `(1-2) + (3-4)` and `(5-6)`, are sent to the Rayon thread pool.
5.  Thread 1 gets `(1-2) + (3-4)`, calls `eval`, and gets the result `-2`.
6.  Thread 2 gets `(5-6)`, calls `eval`, and gets the result `-1`.
7.  `collect()` gathers the results into a vector: `[-2, -1]`.
8.  Finally, `sum()` adds them together: `-2 + -1 = -3`.

The final answer is **-3**, which is correct. The entire process worked perfectly.

By combining these techniques, we've transformed a serial problem into a highly parallel one, leveraging modern CPU architecture to achieve a significant performance boost.




















---

# DEPRECATED 

Mention that I tried it but it did't improve the performance, mainly because the compiler is smart enough to optimize the previous solution, and because the numbers we have are small (between 1 and 4 digits). In any case, this was a good lesson of **always measure**.

#### Better parsing with bitwise hacking

the frlamegrpah shows that tokenizer.next() is taking most of the time. it also shows is_ascii_digit has wide segment

My intuition is that this is because of how we parse intergers. We don't know the length of the integers as they are variable, so we need to iterate byte by byte until we find a non-integer character.

Let's add a restriction, our math parser, only accepts integers from 1 to 4 digits.

Then the next optiomization is to remove the loop, and somehow write a function that tells us the length of the number we are parsing. we can use some bitwise for it


https://rust-malaysia.github.io/code/2020/07/11/faster-integer-parsing.html



// custom number parsing with SIMD (like atoi lib?)
// custom parsing with match knowing there are 2-3 figures integers
// First pass with SIMD to find outter parentheses, then parallelize
// MMAP instead of file read