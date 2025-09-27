---
title: "Fast Data Persistence: GrausDB + Zero-Copy Serialization"
layout: post
date: 2025-09-27 08:00
image: /assets/images/zero-copy/header.png
headerImage: false
tag:
- rust
- performance
- optimization
- zero copy
- GrausDB
category: blog
author: Ricardo Pallas
projects: true
description: How to persist structs with great performance using zero-copy serialization and GrausDB. No copies, no allocations—just direct memory-to-disk writes reaching over 1 million operations per second.
---

# Fast Data Persistence: GrausDB + Zero-Copy Serialization

## Table of Contents

- [Introduction](#introduction)
- [GrausDB Summary](#grausdb-summary)
- [The Problem with Traditional Serialization](#the-problem-with-traditional-serialization)
- [Zero-Copy Serialization](#zero-copy-serialization)
- [Example: The Product Struct](#example-the-product-struct)
- [Product Stock Update](#product-stock-update)
- [Results](#results)
- [Important Caveats and Pitfalls](#important-caveats-and-pitfalls)
- [Conclusion](#conclusion)

## Introduction

Experiment with short pieces of code to make it fast is a fun hobby. A while ago, I wrote about [Optimizing a simple math parser](https://rpallas.xyz/math-parser/), and the obsession hasn't faded. Today, I want to talk about a different kind of speed: the speed of saving and loading your data using [GrausDB](https://github.com/RPallas92/GrausDB).

We often overlook it, but serialization (the process of converting our data structures into a format that can be stored or transmitted) have a high impact on performance. We write our beautiful code, with structs and objects, and then, when it's time to save them to a database, we just any serialization library (even `JSON.stringify` in JavaScript) and hope for the best. But under the hood, a lot of copying, allocating, and processing is happening. It all counts for perf.

What if we could just... not do that? **What if we could take our in-memory data and persist it directly**, with no copies, no intermediate formats, no fuss? This is the promise of zero-copy serialization, and when you combine it with a high-performance embedded database like [GrausDB](https://github.com/grausdb/grausdb), the results are great!

**This blog post is about showing you how to persist structs in GrausDB easily and with incredible performance.**

## GrausDB Summary

I created GrausDB with a simple idea in mind: provide a very simple, low-level API that's easy to use. An API so straightforward that you don't need to constantly check the documentation. It has just 4 methods:

- `get(key)` - Retrieve a value
- `set(key, value)` - Store a value
- `delete(key)` - Remove a value
- `update_if(key, update_fn, value_if_missing, predicate)` - Atomically update a value if certain predicate is satisfied

That's it. You work directly with slices of bytes (`&[u8]`), giving you complete control. And with `update_if`, you can perform atomic updates without worrying about transactions, commits, or flush methods (I didn't include those because I don't want you to think about these details when using the lib).

## The Problem with Traditional Serialization

Imagine you have a book. Every time you want to save its contents, you get a fresh stack of paper and meticulously copy the entire book, word for word. When you want to read it back, you take your copy and read from it. This is how traditional serialization often works.

When you serialize an object (like a `struct` in Rust), the library typically allocates a new buffer and then walks through your object, copying its data field by field into that buffer. Deserialization is the reverse: it reads the buffer, allocates memory for a new object, and then copies the data from the buffer into this new object.

This "photocopying" has a cost:

- **CPU cycles:** All that data copying keeps your CPU busy.
- **Memory allocation:** Constantly creating new objects and buffers.
- **Cache misses:** When data is scattered around in memory, the CPU can't efficiently use its cache, which slows things down.

For most of the applications, this is fine. But when you're building a high-throughput system this overhead becomes a bottleneck.

## Zero-Copy Serialization

Now, imagine instead of photocopying the book, you could just place a bookmark in it. This bookmark tells you exactly where the book is and how it's laid out. To "read" it, you just look at the original. No copies needed.

This is the core idea of zero-copy serialization. We ensure that the way our data is organized in memory is *identical* to how we want to store it on disk. If the memory layout and the disk layout match, **we can just take a raw slice of memory and write it to our database**. To read it back, we can take the raw bytes from the database and interpret them directly as our data structure, without creating any new objects or allocating extra memory in the heap.

In Rust, we can achieve this with a couple of key tools:

1. `#[repr(C)]`: This attribute tells the Rust compiler to lay out a struct's fields in memory in the same way a C compiler would: sequentially, in the order they are defined. This gives us a predictable and stable memory layout.
2. **A zero-copy library:** While you could do this manually with unsafe code, libraries like `musli-zerocopy` make it safe and easy.

Let's see how this works in a real example.

## Example: The Product Struct

Here is the code we'll be benchmarking. It's a simple program that creates a `Product`, saves it to GrausDB, and then reads and modifies it 20,000 times. I added many comments to the code to over-explain it.

```rust
use graus_db::GrausDb;
use musli_zerocopy::{endian, Buf, OwnedBuf, Ref, ZeroCopy};
use std::error::Error;
use std::fs;
use std::mem;
use std::time::Instant;

/// Represents a product with a stock count and a name.
/// `#[derive(ZeroCopy)]` enables zero-copy serialization/deserialization with `musli-zerocopy`.
/// `#[repr(C)]` ensures a C-compatible memory layout, which is required for zero-copy.
#[derive(ZeroCopy)]
#[repr(C)]
struct Product {
    stock: u16,
    name: Ref<str>, // `Ref<str>` allows zero-copy referencing of string data within the buffer.
}

impl Product {
    /// Serializes a `Product` into an `OwnedBuf` using zero-copy principles.
    /// The `stock` is stored directly, and the `name_str` is stored as a `Ref<str>`.
    /// This avoids copying the string data during serialization.
    fn to_bytes(stock: u16, name_str: &str) -> OwnedBuf {
        // Create a new owned buffer, configured for little-endian byte order.
        let mut buf = OwnedBuf::new().with_byte_order::<endian::Little>();
        // Reserve space for the `Product` struct without initializing it.
        let product_ref = buf.store_uninit::<Product>();
        // Store the string data for the name, returning a `Ref<str>` to it within the buffer.
        let name = buf.store_unsized(name_str);
        // Load the uninitialized `Product` reference and write the actual `Product` data into it.
        buf.load_uninit_mut(product_ref)
            .write(&Product { stock, name });
        buf
    }

    /// Deserializes a `Product` reference from a byte slice using zero-copy.
    /// This function returns a reference to the `Product` directly from the input bytes,
    /// without allocating new memory for the struct itself.
    fn from_bytes<'a>(bytes: &'a [u8]) -> &'a Product {
        // Create a `Buf` from the input byte slice.
        let loaded_buf = Buf::new(bytes);
        // Create a `Ref` to the `Product` at the beginning of the buffer (offset 0).
        // This assumes the `Product` struct is at the start of the serialized data.
        let loaded_product_ref = Ref::<Product, endian::Little>::new(0 as usize);
        // Load the `Product` reference from the buffer. `unwrap()` is used here for simplicity,
        // but in a real application, error handling would be necessary.
        loaded_buf.load(loaded_product_ref).unwrap()
    }
}

const ITERATIONS: usize = 20_000;

/// Main function to demonstrate GrausDB usage with zero-copy serialization/deserialization structs.
/// It opens a database, sets a product, retrieves it, decreases its stock, and retrieves it again (20k times).
fn main() -> Result<(), Box<dyn Error>> {
    let db_path = "./grausdb_data";
    let _ = fs::remove_dir_all(db_path); // Just to clean up previous database data (if it exists)
    let db = GrausDb::open(db_path)?;

    println!("GrausDB opened at ='{:?}'", db_path);

    // Create a Product and serialize it into an OwnedBuf using zero-copy.
    let product_buf = Product::to_bytes(ITERATIONS as u16 + 1, "Yeezy Boost 350 V2");

    // Store it in the database.
    let key = b"yeezy".to_vec();
    db.set(key.clone(), &product_buf[..])?;

    let start_time = Instant::now();

    for _i in 0..ITERATIONS {
        // Retrieve the product bytes from the database.
        let loaded_bytes = db.get(&key)?.expect("Value not found");
        // Deserialize the bytes back into a `Product` reference using zero-copy.
        let loaded_product = Product::from_bytes(&loaded_bytes);

        // To access the name, which is a `Ref<str>`, we still need the original buffer.
        // This is a limitation of `Ref<str>` and zero-copy deserialization:
        // the `loaded_product` itself contains a `Ref<str>`, which needs a `Buf` to resolve
        // the actual string slice from the underlying byte buffer.
        let loaded_buf_for_name = Buf::new(&loaded_bytes);

        // In a real benchmark, you might want to assert values or perform some operation
        // to ensure the data is correctly loaded, but for a simple performance test,
        // just loading and accessing is sufficient to test deserialization.
        let _ = loaded_buf_for_name.load(loaded_product.name)?;

        // This function decreases the stock by 1, and stores the updated product in the db again.
        decrease_stock(key.clone(), &db)?;
    }

    let duration = start_time.elapsed();
    println!("Benchmark completed in {:?}", duration);

    // We retrieve the product and print it to check that the stock is 1.
    let loaded_bytes_final = db.get(&key)?.expect("Value not found after benchmark");
    let loaded_product_final = Product::from_bytes(&loaded_bytes_final);
    let loaded_buf_for_name_final = Buf::new(&loaded_bytes_final);

    println!(
        "Final Product state: stock = {}, name = {}",
        loaded_product_final.stock,
        loaded_buf_for_name_final.load(loaded_product_final.name)?
    );

    Ok(())
}

/// Decreases the stock of a product identified by `key` in the `GrausDb`.
/// This function performs an in-place, zero-copy update for max performance.
/// It directly modifies the `stock` field within the stored byte buffer.
fn decrease_stock(key: Vec<u8>, db: &GrausDb) -> Result<(), Box<dyn Error>> {
    // The `update_fn` closure is executed by `db.update_if` with mutable access
    // to the raw byte vector (`&mut Vec<u8>`) representing the stored value.
    let update_fn = |value: &mut Vec<u8>| {
        // Ensure the buffer is large enough to contain at least the stock field (u16).
        // The `stock` field is at the beginning of the `Product` struct due to `#[repr(C)]`.
        if value.len() < mem::size_of::<u16>() {
            panic!("Buffer too small to contain stock for key: {:?}", key);
        }

        // Read the current stock value (u16) from the first two bytes of the buffer.
        let current_stock = u16::from_le_bytes([value[0], value[1]]);

        // Decrement the stock, using `saturating_sub(1)` to prevent underflow (stock won't go below 0).
        let new_stock = current_stock.saturating_sub(1);

        // Convert the new stock value back into its little-endian byte representation.
        let new_stock_bytes = new_stock.to_le_bytes();
        // Write the new stock bytes directly back into the first two bytes of the buffer.
        value[0] = new_stock_bytes[0];
        value[1] = new_stock_bytes[1];
    };

    // Call `db.update_if` to atomically update the value associated with the key.
    //
    //
    // Note: The `None as Option<fn(&[u8]) -> bool>` cast is required by Rust compiler
    // infer the generic type `P` for the `predicate` parameter when `None` is provided,
    // resolving type inference ambiguity.
    db.update_if(
        key.clone(),
        update_fn,
        None,
        None as Option<fn(&[u8]) -> bool>,
    )
    .map_err(|e| e.into())
}
```

Our `Product` struct looks like this:

```rust
#[derive(ZeroCopy)]
#[repr(C)]
struct Product {
    stock: u16,
    name: Ref<str>,
}
```

- `#[derive(ZeroCopy)]`: This comes from `musli-zerocopy` and automatically implements the necessary traits for zero-copy operations.
- `#[repr(C)]`: As we discussed, this ensures the `stock` and `name` fields are laid out sequentially in memory.
- `Ref<str>`: This is the clever part. A `String` or `&str` can have any length, which complicates a fixed memory layout. `Ref<str>` solves this. It doesn't store the string data itself, but acts like a "pointer" or an offset to where the string data is located within the same byte buffer.

The `Product::from_bytes()` function is where the magic happens. It takes a slice of bytes (`&[u8]`) and, with a bit of pointer casting, gives us back a `&Product`. It's almost instantaneous because there's no real "work" to do. It's just re-interpreting the existing data. **This is key, we just read the bytes from the database and tell Rust to treat them as a `Product` (no other serialization operations)**.

## Product Stock Update

The benchmark doesn't just read data; it performs a read-modify-write cycle. Look at the `decrease_stock()` function. It uses `db.update_if`, which gives us direct, mutable access to the raw bytes of the value *as they exist in the database*.

```rust
let update_fn = |value: &mut Vec<u8>| {
    // Read the current stock from the first two bytes
    let current_stock = u16::from_le_bytes([value[0], value[1]]);
    // Decrement it
    let new_stock = current_stock.saturating_sub(1);
    // Write the new value back to the first two bytes
    let new_stock_bytes = new_stock.to_le_bytes();
    value[0] = new_stock_bytes[0];
    value[1] = new_stock_bytes[1];
};
```

Because we used `#[repr(C)]`, we know *exactly* where the `stock` value is: it's the first two bytes of our data. So, to decrease the stock, we don't need to deserialize the whole object. We can just read those two bytes, calculate the new value, and write them back.

## Results

So, this is the result of combining zero-copy deserialization with GrausDB on a cheap laptop:

```bash
rpallas@rpallas-Surface-Laptop-Go-2:~/workspace/GrausDB/examples/zero_copy_struct_serde$ cargo run --release
    Finished `release` profile [optimized] target(s) in 0.02s
     Running `/home/rpallas/workspace/GrausDB/target/release/grausdb_example`
GrausDB opened at ='"./grausdb_data"'
Benchmark completed in 54.992673ms
Final Product state: stock = 1, name = Yeezy Boost 350 V2
```

**55 milliseconds for 20,000 full read-modify-write cycles.**

Let me break down what's happening in each iteration:

1. `db.get(&key)` - Reading the product bytes from the database
2. `Product::from_bytes(&loaded_bytes)` - Zero-copy deserialization (basically free!)
3. `loaded_buf_for_name.load(loaded_product.name)` - Accessing the name field to ensure the data is valid and just to deserialize it for the benchmark
4. `decrease_stock(key.clone(), &db)` - Another database call with `db.update_if` to modify the stock atomically

So each iteration involves multiple database operations: two reads and one write. That means we're doing **60,000 database operations** in 55 milliseconds—that's over **1 million operations per second**. On a Surface Go 2! 

By avoiding data copies and using zero-copy deserialization, we can build systems that are fast, even on modest hardware.

## Important Caveats and Pitfalls

Zero-copy is powerful, but it has some caveats.

### 1. Endianness and Portability

The example uses `endian::Little`. If you write data on one architecture and read on another with different endianness, or if you change the byte order, things will break.

### 2. Alignment and Padding

`#[repr(C)]` helps, but be mindful of alignment and padding. When you add fields, check `mem::size_of` and `mem::align_of` to make sure offsets stay where you expect.

### 3. Partial Writes and Crash Consistency

Writing a `u16` in-place involves changing two bytes. On many systems, updating those two bytes is fast, but it is *not guaranteed* to be atomic with respect to power loss. If the process or machine crashes while the write is in-flight, you might end up with a partially updated value (corruption). GrausDB does not implement this yeat (that ios why is not production ready).

### 4. Durability vs Performance (fsync)

**⚠️ Warning:** GrausDB does **not** call `fsync` after every write by default. This is an explicit trade-off for speed.

**What is fsync?**

The operating system typically caches file writes in memory (page cache) and flushes them to disk later. `fsync` forces the OS to flush those buffers to the physical storage, ensuring the data is durable on disk before the call returns.

**Why GrausDB skips it:** Calling `fsync` is slow. Skipping `fsync` lets databases reach very high throughput, but at the cost that recently-written data may be lost if the machine crashes before the OS flushes the pages to disk.

**When this trade-off makes sense:** For many use cases, this is perfectly acceptable. Think about a server handling a product drop where all the stock is sold in 2 seconds. You need extreme performance during those 2 seconds. In fact, GrausDB can be better than Redis for this scenario because you don't have the network overhead of sending data back and forth. And if you need high availability, you can create a cluster of servers with local GrausDB instances and sync them asynchronously for maximum performance.

## Conclusion

We've seen how combining embedded database like GrausDB with zero-copy serialization in Rust can lead to great performance. My key takeaway is to avoid copying as it is hidden performance cost. Zero-copy eliminates it.

This approach requires a bit more thought than just using a standard serialization library, but the performance gains are worth it.

If you want to dive deeper into the mechanics of how libraries like `musli-zerocopy` work their magic, I highly recommend reading this excellent article:

- **Understanding Musli Zero-Copy:** [https://udoprog.github.io/rust/2023-10-19/musli-zerocopy.html](https://udoprog.github.io/rust/2023-10-19/musli-zerocopy.html)

**Also, you can find the full code example here:** [https://github.com/RPallas92/GrausDB/blob/main/examples/zero_copy_struct_serde/src/main.rs](https://github.com/RPallas92/GrausDB/blob/main/examples/zero_copy_struct_serde/src/main.rs)

Thanks for reading, and happy coding!