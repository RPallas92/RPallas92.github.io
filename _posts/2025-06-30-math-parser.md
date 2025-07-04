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
Result: 15
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
Step 1: Input file read in 1.285800942s
Step 2: Calculation completed in 4.25785019s

--- Summary ---
Result: 15
Total time: 5.543693402s
```

**Wow! From 43 seconds down to just 5.54**. What an improvement. A small mistake can have a huge impact on performance. Fortunately, the flamegraph pointed us straight to the bottleneck!

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
Step 1: Input file read in 1.435023269s
Step 2: Calculation completed in 2.136090105s

--- Summary ---
Result: 15
Total time: 3.571143085s
```

Great improvement! From **5.54 to 3.57 seconds**. Almost 2 seconds faster!
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
Step 1: Input file read in 1.070629616s
Step 2: Calculation completed in 1.881817656s

--- Summary ---
Result: 15
Total time: 2.952483691s
```

From **3.56 to 2.95 seconds**. We are getting faster. Let's keep optimizing!



---

// custom number parsing with SIMD (like atoi lib?)
// custom parsing with match knowing there are 2-3 figures integers
// First pass with SIMD to find outter parentheses, then parallelize
// MMAP instead of file read