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

1. [Table of contents](#table-of-contents)
1. [Baseline implementation](#baseline-implementation)
    1. [How it works](#how-it-works)
    1. [Parser Example: `(1 + 2) - 3`](#parser-example-1--2---3)
    1. [Sequence Diagram](#sequence-diagram)
    1. [It works! But we can do better](#it-works-but-we-can-do-better)
1. [Optimizations for speed and memory](#optimizations-for-speed-and-memory)
    1. [Optimization 1: Do not allocate a Vector when tokenizing.](#optimization-1-do-not-allocate-a-vector-when-tokenizing)

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
    let mut tokens = tokenize(input).peekable();
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

If we compile it in release mode and execute it for the test file of **3.2GB**, it takes **15.56 seconds** to execute on my laptop:

```
Step 1: Input file read in 1.922002912s
Step 2: Calculation completed in 8.646109663s

--- Summary ---
Result: 15
Total time: 10.568144221s
```

In the next sections, we’ll go step by step to optimize this parser:

1. Avoid allocating the token list.
2. Use iterators and slices instead of `Vec<Token>`.
3. Parse directly from the input string.

The goal is to make a parser that’s not only simple and correct, but also lean and fast.

Let’s go!

---

## Optimizations for speed and memory

### Optimization 1: Do not allocate a Vector when tokenizing.

flamegraph and memory



---




