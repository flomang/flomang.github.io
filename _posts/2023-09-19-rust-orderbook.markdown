---
layout: post
title:  "Enhancing an Order Book in Rust"
date:   2023-09-19 11:01:43 -0700
categories: rust trading 
---

In the world of high-frequency trading and financial systems, precision and reliability are paramount. A small error in calculations can lead to significant financial losses. Today, I revisit a project where I took an existing order book implementation and made key improvements to enhance its precision. The original order book implementation was forked from dgtony's [project](https://github.com/dgtony/orderbook-rs), and I'm grateful for the foundation he provided.

My version of his orderbook can be found [here](https://github.com/flomang/paper/blob/main/orderbook/README.md)

### Transition from Sequential IDs to GUIDs
One of the critical changes I introduced to the order book was the replacement of sequential order IDs with globally unique identifiers (GUIDs). While sequential IDs work well for simple applications with a single database, they pose challenges in more complex, production-level environments where multiple databases are involved, especially in high-frequency trading platforms.

GUIDs ensure that each order has a globally unique identifier, even when distributed across multiple databases. This uniqueness is crucial in preventing conflicts and ensuring the integrity of the order book. 

### Precision Matters: Transitioning to BigDecimals
The second significant change I made to the original implementation was replacing floating-point numbers with BigDecimal to represent the order amounts. This transition was motivated by the need to eliminate floating-point rounding errors that could potentially lead to inaccuracies in financial computations.

Floating-point numbers, such as f64, are inherently imprecise when it comes to decimal arithmetic. They are susceptible to rounding errors, which can accumulate over time and result in significant discrepancies in financial calculations.

BigDecimal, on the other hand, provides precise decimal arithmetic. It allows us to work with decimal numbers without the risk of rounding errors. In the context of an order book system, where financial transactions and calculations are frequent and critical, BigDecimal ensures accuracy and reliability.

To illustrate the difference between using floating-point numbers and BigDecimal, consider the following Rust code snippet:

```
let x = 0.1_f64;
let y = 0.2_f64;
let z = x + y;

println!("x: {}", x);
println!("y: {}", y);
println!("x + y: {}", z);

if z == 0.3_f64 {
    println!("z is equal to 0.3");
} else {
    println!("z is not equal to 0.3");
}
```

Output:
```
x: 0.1
y: 0.2
x + y: 0.30000000000000004
z is not equal to 0.3
```

As demonstrated in the output, even a seemingly simple addition of 0.1 and 0.2 using floating-point numbers results in a value slightly off from the expected 0.3. This discrepancy can lead to significant issues in financial systems, highlighting the importance of using BigDecimal for precise calculations.

Here's the equivalent using BigDecmial: 

```
let x = BigDecimal::from_str("0.1").unwrap();
let y = BigDecimal::from_str("0.2").unwrap();
let z = x + y;

println!("x: {}", x);
println!("y: {}", y);
println!("x + y: {}", z);

let expected = BigDecimal::from_str("0.3").unwrap();

if z == expected {
    println!("z is equal to 0.3");
} else {
    println!("z is not equal to 0.3");
}
```

Output:
```
x: 0.1
y: 0.2
x + y: 0.3
z is equal to 0.3
```

### Conclusion
Upon revisiting the order book project, I've demonstrated the remarkable impact of subtle yet pivotal adjustments in bolstering the reliability and precision of a financial system. By shifting from sequential order IDs to GUIDs and adopting BigDecimal in lieu of floating-point numbers, we've fortified the system's foundations. As a result, it stands ready to provide not only heightened accuracy but also the guarantee of unique identifiers for our orders across a multitude of databases.