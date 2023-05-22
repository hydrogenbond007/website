---
sidebar_position: 1
slug: ./
---

# Proof System Overview

When a the RISC Zero [zkVM](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/) executes, it produces a computational [receipt](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/receipt/struct.Receipt.html) that consists of:
- a [journal](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/receipt/struct.SessionReceipt.html#structfield.journal), which contains the public outputs of the computation, and
- a [seal](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/receipt/struct.Receipt.html#structfield.seal), which is a [zk-STARK](../../reference-docs/about-starks.md). 

Given a receipt and an [ImageID](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/struct.MemoryImage.html#structfield.root), a skeptical third party can [verify](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/receipt/struct.Receipt.html#method.verify) the purported output of the computation. 
The time to veirfy the zk proof is orders of magnitude less compared to running the whole program again to check its computational integrity.

## Learn More
### About the zkVM
- [Video Explainer from zkHack](https://www.youtube.com/watch?v=8hwY88xJoyM&list=PLcPzhUaCxlCgig7ofeARMPwQ8vbuD6hC5&index=8)
- [Written Explainer](../zkvm/zkvm_overview.md)

### About the Proof System
- [ZKP Whitepaper](https://www.risczero.com/proof-system-in-detail.pdf)
- [Prover-Verifier Sequence Diagram](proof-system-sequence-diagram.md)
- [Stark Whitepaper](https://eprint.iacr.org/2018/046.pdf)
- STARK by hand
  - [Website version](stark-by-hand.md)
  - [Google Sheet version](https://docs.google.com/spreadsheets/d/1Onr41OozD62y-B0jIL7bHAH5kf771-o4xvmnHUFpOyo/edit?usp=sharing) 
  - [Printable Version](assets/fibonacci-stark.pdf)
 

### Helpful Background
[RISC Zero Study Club](../../../studyclub) is intended to help make the technology underpinning RISC Zero more accessible. Past sessions include: 
  - [What is RISC-V and how does it relate to RISC Zero?](https://www.youtube.com/watch?v=11DIflEwx50&list=PLcPzhUaCxlCjdhONxEYZ1dgKjZh3ZvPtl&index=5)
  - [Intro to Finite Fields](https://www.youtube.com/watch?v=11DIflEwx50&list=PLcPzhUaCxlCjdhONxEYZ1dgKjZh3ZvPtl&index=2)
  - [Intro to Reed-Solomon Codes](https://www.youtube.com/watch?v=11DIflEwx50&list=PLcPzhUaCxlCjdhONxEYZ1dgKjZh3ZvPtl&index=3)
  - [Intro to Number Theoretic Transforms](https://www.youtube.com/watch?v=11DIflEwx50&list=PLcPzhUaCxlCjdhONxEYZ1dgKjZh3ZvPtl&index=4)
  - [Intro to the FRI Protocol](https://www.youtube.com/watch?v=11DIflEwx50&list=PLcPzhUaCxlCjdhONxEYZ1dgKjZh3ZvPtl&index=1)
  - [Finite Field Implementation Options: Barrett & Montgomery Multiplication](https://www.youtube.com/watch?v=hUl8ZB6hpUM&list=PLcPzhUaCxlCjdhONxEYZ1dgKjZh3ZvPtl&index=6)

