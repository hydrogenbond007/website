---
slug: continuations
title: Using Continuations to Prove Any EVM Transaction
authors: [timzerrell]
tags: [zk, continuations, evm]
---

[zkevm-vs-zkvm]: https://www.risczero.com/blog/zkVM
[evm-example-github]: https://github.com/risc0/risc0/tree/main/examples/evm
[odra-zkvm-evm]: https://odra.dev/blog/evm-at-risc0/
[SputnikVM]: https://github.com/rust-blockchain/evm/
[zkPoEX]: https://github.com/zkoranges/zkPoEX
[revm]: https://crates.io/crates/revm
[crate-validation]: https://risc0.github.io/ghpages/dev/crate-validation/index.html
[small-transaction]: https://etherscan.io/tx/0x671a3b40ecb7d51b209e68392df2d38c098aae03febd3a88be0f1fa77725bbd7
[precompile-transaction]: https://etherscan.io/tx/0x600d18676aef439ec6ba33d143b78878a520682be7fd8331c74bdf672988a2b1
[precompile-contract]: https://etherscan.io/address/0x6b175474e89094c44da98b954eedeac495271d0f#code
[Bonsai]: https://www.bonsai.xyz/
[bonsai-waitlist]: https://fmree464va4.typeform.com/to/t6hZD54Z
[segment-docs]: https://docs.rs/risc0-zkvm/latest/risc0_zkvm/struct.Segment.html
[session-docs]: https://docs.rs/risc0-zkvm/latest/risc0_zkvm/struct.Session.html
[cycles-docs]: https://docs.rs/risc0-zkvm/latest/risc0_zkvm/prove/struct.Prover.html#structfield.cycles
[segment-limit-docs]: https://docs.rs/risc0-zkvm/latest/risc0_zkvm/struct.ExecutorEnvBuilder.html#method.segment_limit_po2
[session-receipt-docs]: https://docs.rs/risc0-zkvm/latest/risc0_zkvm/receipt/struct.SessionReceipt.html

At RISC Zero, we envision a future with boundless computation built on zero knowledge proofs.
Today, we took a major step toward implementing this vision with tools available to all.
We published v0.15 of the RISC Zero zkVM, which includes one of my favorite features: continuations.

In the context of our zkVM, continuations are a mechanism for splitting a large program into several smaller segments that can be computed and proven independently.
This has many benefits, for instance:
* Parallelizing proving
* Enabling pausing and resuming a zkVM (similar to a “warm start” on AWS Lambda)
* Limiting memory requirements to a fixed amount, regardless of program size

I discuss each of these a bit more at the end of this post, but the main benefit I'll focus on today is that with continuations, programs are no longer bounded by a fixed maximal length of computation.
With continuations, programs can run for however many instructions it takes to get the job done.

But what does an unbounded cycle count enable in practice?
The pithy answer is that the possibilities are endless — our zkVM is general purpose and [can run][crate-validation] anything that compiles to RISC-V (e.g. Rust, but also C++, Go, etc.), and now, just like the device you're reading this on, our zkVM will execute programs for however long they take to complete.
This possibility, while endless, is comprised of innumerable specific examples.
For instance, with continuations, you can run an EVM engine inside the RISC Zero zkVM, and prove the state changes caused by an Ethereum transaction.
By "you," I mean you, personally, right now, on your laptop.
Whenever you like, you can head on over to [our EVM example][evm-example-github], check out the source code, and run it for yourself!
In the meantime, keep reading for a deeper explanation on what continuations enable, including how we use them to prove the results of Ethereum transactions.

## Running the EVM on the RISC Zero zkVM

Erik recently described how [the RISC Zero zkVM differs from a zkEVM][zkevm-vs-zkvm].
That post is worth reading in full, but I'm going to gloss over the details and nuances and instead pull a single key quote:
> On the zkVM, you can run just about any software that runs on a computer rather than anything that can run on Ethereum.

The EVM is software that runs on a computer.
This quote, then, suggests that the EVM can be run inside our zkVM — and indeed it can. In fact, a few different teams have done this already:
[Odra wrote a proof of concept][odra-zkvm-evm] with [SputnikVM],
[zkPoEX] also uses SputnikVM to produce proofs of exploits,
and we have an EVM example using [revm].

The [revm] crate is a Rust-based EVM interpreter.
Like [most Rust crates][crate-validation], the revm crate can be run in the RISC Zero zkVM.
Our [example][evm-example-github] does exactly that.
When you point it at an Ethereum transaction, our example will use [revm] to execute the transaction to compute the new state.
It will then create a receipt containing the difference in state after the transaction has been executed and a zero-knowledge proof that this result is accurate.

Before continuations, this process worked only for small transactions.
Therefore, when we published our EVM example, we used [this small transaction][small-transaction] as a demonstration so we could prove the transaction without hitting the cycle cap.
However, since we hadn't yet added continuations to the zkVM, we also published our EVM example with a warning that it wouldn't work for all transactions, and in particular was unlikely to be able to prove transactions with elliptic curve precompiles.

With continuations, we no longer have this limitation.
You can, for instance, prove [this fairly heavy transaction][precompile-transaction] for a [contract][precompile-contract] using the `ecrecover` precompile and 5 `KECCAK256`.
On my M1 Max MacBook Pro, this took about 12 minutes and about 12 GB of memory.
The runtime will of course vary from system to system, and depends on the length of the program execution.
The memory requirements, however, should be similar regardless of what system you run the proof on, and also regardless of the length of your program.

And to be clear, it’s not just that this particular transaction can now be proven on the zkVM. It’s all transactions:
with continuations, any Ethereum transaction can be proven on the RISC Zero zkVM using our EVM demo, and there is no limit on transaction size.

## What Is a Continuation?

I mentioned at the start of this post that continuations are "a mechanism for splitting a large program into several smaller segments that can be computed and proven independently."
This mechanism works by tracking the state of the zkVM at the start and end of each of these smaller segments, in the form of Merkle trees of the memory image (plus the program counter).
This lets us compare the ending state of one segment to the starting state of the next.

A zkVM program is automatically split into [segments][segment-docs], based on the [cycle count][cycles-docs].
If the program would run for more cycles than allowed in a single segment, it is automatically split.
We use the term "[session][session-docs]" to mean sequence of segments where the first segment was initiated by the user and the final segment was terminated by the user (as opposed to being terminated by an automatically generated split).
Thus, while segments have arbitrary boundaries determined automatically to stay within the per-segment [cycle limit][segment-limit-docs], sessions instead represent semantic boundaries, both starting and ending at the request of a user.

A [session receipt][session-receipt-docs] consists of multiple segment receipts, and is validated by confirming that:
1. each segment receipt is valid,
2. the starting receipt of each segment matches the ending state of the prior segment, and
3. the initial state (or `image_id`) matches what the validator expects (that is, that session is running the same code the validator wants to validate).

## Benefits of Continuations

As discussed with the EVM example, one of the most immediately apparent benefits of continuations is that you can now run a RISC Zero zkVM program for as long as you need to get the job done.
But continuations enable more capabilities than just this. I want to touch on three I mentioned in the introduction: parallelization, pausing, and memory.

### Parallelize
These small segments can be distributed to many computers to parallelize the workload and reduce latency.
The plan for where a zkVM program is split into segments and what work is done in each segment is computed ahead of proving by an "execution" phase.
This gives a plan for proving each segment that is independent of the contents of all of the other segments.
Thus, the hard work of proving the segments can be distributed amongst many systems, parallelizing the workload and reducing overall latency of the full proof.

### Pause-Resume
With continuations, a zkVM program can be paused and resumed.
When describing segments, I mentioned that they lasted until the user asked for them to stop.
This can be a traditional halt, indicating the end of computation.
But it can also be a pause, which lets the user say "I want to do some computation now, and then come back at some later time and pick up where I left off."

For example, imagine a zkML workload in which an ML model is constructed, its weights are loaded, input data is provided, and an output is computed.
In this example, the zkVM could be paused after the weights are loaded, just before input is provided.
This has a couple of advantages.
First, the model construction and weight loading could be performed prior to the user providing inputs, reducing the latency between inputs and outputs.
Moreover, this initial setup phase could be performed once and paused, and then resumed multiple times for different inputs, saving the work of re-executing the shared setup.

### Fixed Memory Requirements
Prior to continuations, when a zkVM program took twice as many cycles, it roughly doubled the runtime and _also_ roughly doubled the memory requirements.
With continuations, memory requirements depend on segment length rather than total program length, so that programs of arbitrary execution length require a fixed amount of memory to run.

## Continuations and Bonsai

Continuations are a powerful tool. We want to simplify the complexities of continuations, and zero-knowledge proofs more generally, as much as possible. To this end, we are working on Bonsai, and so here I want to mention a few of the complications we expect [Bonsai] to simplify.

For one, while there is the potential for substantial latency improvement through parallelization, orchestrating the distribution of a program's segments is not built into the zkVM code.

For another, we currently only support _flat continuations_ and not _rollup continuations_.
With flat continuations, each segment produces its own receipt.
Each of these receipts needs to be individually verified, and while we have helper functions to prove the overall program, the required verification work still increases with execution length.
Rollup continuations would use proofs of verification to recursively rollup these individual segment proofs.
The result would be that only a single receipt would be necessary for verification, and verification time would be constant regardless of execution length.

We are working on [Bonsai], which we believe addresses these challenges and more.
We expect [Bonsai] to make the power of zero-knowledge proofs and continuations easily accessible.
If you're interested, [sign up for the Bonsai waitlist][bonsai-waitlist]!
