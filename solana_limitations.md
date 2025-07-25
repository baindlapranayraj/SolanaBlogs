<img 
  width="1000px"
 height="430px"
 src="./images/shinchan .jpg"
/>

# 🦀 Deep Dive: Solana Limitatons (Still writing)

GM GM everyone 😁,

In this blog, we will go through the various types of Solana resource limitations you might encounter while developing smart contracts. You may run into errors containing words like "limit" or "exceed." These errors represent boundaries predefined by Solana programs to maintain fairness and performance on the blockchain. If you’d like to get more information on these topics, you’re in the right place!

Hear what u can expect from this blog:-

- CU Limitations
- Transaction size limitations
- Stack size

# Limitations of Solana:

Solana is a high-performance public blockchain that stands apart from traditional blockchains like Bitcoin and Ethereum due its unique approach to transaction processing separates logic and state into different accounts, which allows Solana to process many transactions simultaneously.

However, programs running on Solana are subject to several types of resource limitations. These limitations ensure that programs use system resources fairly while maintaining high performance.

Knowing these boundaries is very helpful, and as the program designer, you must be mindful of these limitations to create programs that are affordable, fast, safe, and functional.

## 1. Compute Unite Limitations:

### What is Compute Units ?

CU (Compute Unit) the name itself suggests that CU is the fundamental measurement of the computational work (CPU cycles and memory usage) performed by a transaction or instruction on Solana. It's similar to "Gas" fees in Ethereum but is much more predictable and low-latency.

Every instruction your smart contract executes on-chain, such as reading or writing to accounts, performing cryptographic operations (like zk-ElGamal), or verifying signatures and serailization and deserialization consumes a certain number of compute units (CUs). This is roughly proportional to the amount of work done by the nodes.

If you perform simple transactions, then nodes can process those smart contracts efficiently, resulting in lower CU consumption. However, if you perform complicated mathematical operations or heavy loops, nodes consume a large amount of memory and CPU, and it takes more time to run the program (smart contract), resulting in higher CU consumption

_For example, for an simple transaction like sending a SOL form wallet A to wallet B it takes around 3000 Compute Units_

```rust

let transfer_amount_accounts = Transfer {
      from: ctx.accounts.signer.to_account_info(),
      to: ctx.accounts.recipient.to_account_info(),
    };

let ctx = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        transfer_amount_accounts,
      );

transfer(ctx, amount * LAMPORTS_PER_SOL)?;

 // Takes around 3000 CU
```

The code in the above was written in Anchor ⚓️, It performs an simple transfer of SOL, and for that it taking 3000 CUs.

### Compute Unit Budget

- As we know, heavy mathematical operations or loops consume a large amount of compute units. However, there is a default budget of 200,000 CUs for every transaction or instruction.

- If a transaction or instruction exhausts the 200,000 CU limit, the transaction or instruction is simply reverted, all state changes are undone, and the fees are not refunded to the signer who invoked the transaction. (This mechanism prevents attackers from running never-ending or computationally intensive programs on nodes, which could slow down or halt the chain.)

```rust
Error: exceeded maximum number of instructions allowed (200000) compute units
```

- Personally, I encountered this error while building my Chaubet project. When a bettor buys shares,[this function](https://github.com/baindlapranayraj/Chaubet/blob/86b5c91de727dd173a0593a7378a0acb3eb25b2a/programs/chaubet/src/utils/helper.rs#L89C5-L132C6) performs some heavy mathematical computations and checks. Due to this, the instruction exceeded the 200,000 CU limit, and the entire instruction was reverted.

- Well we can increase our computation limit using `SetComputeUnitLimit` we can request a specific calculation unit limit by adding an instruction to our transaction. But 🍑 we can only increase CUs Budget only upto 1.4 Million Units.

```rust
const computeLimitIx = ComputeBudgetProgram.setComputeUnitLimit({
  units: 500000,  // Increased from 200k to 500k CUs.
});
```

### Why do we have a limited Compute Unit Budget?

In short and simple terms, Solana has CU limitations to ensure fair resource allocation. But what does that mean?

Solana validators are individual computers (or nodes) that process blockchain transactions and maintain the network’s state (these are the basic fundamentals you must know 😒). Each validator has limited CPU power and memory, just like any regular computer. When programs run inside these nodes, they allocate some memory to process all the instructions.

Now, if a malicious user sends a transaction that contains infinite loops, it can use a huge amount of memory and may slow or even crash the system. Since the blockchain is a network of shared computers/nodes, if one user performs a huge number of CPU tasks and uses a large amount of CPU/memory, it hogs the system, starving other users.

Due to this limitation in the CU Budget, it helps the Solana network to prevent denial-of-service (DoS) attacks and resource exhaustion on validators.

## 2. Transaction Size Limit.

Before getting into Transaction Size Limit, lets peak a little into what are transactions what do they contain.
A transaction on Solana is a request sent by users to interact with the network, typically to mutate data such as transferring lamports (the native token unit) between accounts.

Each transaction contains one or more instructions that specify the operations to perform on the blockchain. This execution logic is stored in raw bytes in program state.

<img
 width="1000px"
 height="550px"
 src="./images/trx_arch.png"
/>

### The Transaction Contains:

- **Array of signatures**:- A sequence of signatures included in the transaction.
- **Message**:- This is the core part of the transaction. It contains everything needed to describe what the transaction intends to do.

  As you are already seeing the above Image, The Message is the core part of the transaction which contains the :-

- Header:- This one indicates the number of signers and read-only accounts **(3 bytes)**
- Array of Accounts:- It contains all the accounts required to perfrom the transaction, provided from the client (Each Pubkey 32 bytes)
- Recent Blockhash:- A recent blockhash was attached to this transaction (32 bytes)
- Array of Instruction:- Contains all instructions invoked by the signer(The size is depended on complexity of Instruction code).

### Transacion Size Limitaions:-

| **Component**    | **Size**               | **Description**                                                     |
| ---------------- | ---------------------- | ------------------------------------------------------------------- |
| Signature        | 64 bytes               | Each signature included in the transaction                          |
| Message Header   | 3 bytes                | Indicates number of signers and read-only accounts                  |
| Account Pubkey   | 32 bytes (per account) | Public key for each account involved in the transaction             |
| Recent Blockhash | 32 bytes               | Recent blockhash attached to the transaction                        |
| Instructions     | Varies                 | Depends on number and complexity of instructions in the transaction |

<br/>

Every Solana transaction has a size limit of **1,232 bytes**. The total transaction size is the sum of the bytes for all signatures, accounts involved, the blockhash, and the instructions (including their accounts and data), and must be less than 1,232 bytes. Because of this limit, developers need to fit all signers, account addresses, and required data within the 1,232-byte constraint

You would have encounter the `transaction size limit exceeded` error when you try to send too many accounts from client side or too many instructions in single transaction.

**you might wonder why only limited to 1,232 bytes ?** <br/>
In short :- The transaction size limit of 1,232 bytes in Solana is closely tied to how data is transmitted over the internet using IPv6 (Internet Protocol version 6). But lets break it down more.

**Hears Why ?:**

#### Let’s go back to some fundamentals of computer networking briefly.

So, what is computer networking?

In simple terms, it's a group of computers or nodes connected together to share data 📦 among them. This can involve communication between two local nodes/computers (LAN) or between computers across the globe (Internet).

For In order to communicate with nodes effectively without sending data 📦 or information to the wrong destination node, we need a set of rules or protocols to follow. **IP (Internet Protocol)** is the universal addressing system for nodes on a network. IP provides unique address for every node or computer and uses these addresses to route and deliver data packets 📦 to the correct destination. Data sent over a network isn’t sent all at once. It’s broken into small pieces called packets.

But solana also uses UDP(User Datagram Protocol) on top of IP, to send the packets/data 📦 more quickly (unlike in etheareum uses TCP which do lots of checks and hinders the speed if packet is too large). UDP just fires packets 📦, as fast as possible, with no guarantee they’ll arrive. But 🍑 if packet is large(if greater then MTU) then might get fragmented(splitted) which we may loose some data which reduces the reliablity.

<img
 width="1000px"
 height="550px"
 src="./images/trx_size_udp.png"
/>

OK now why are we learning all these stuff ? how are they related to Transaction Size limit ?
Hold your horses we are getting there lets connet some dots 😌.

Solana nodes, like other networked computers, communicate using the Internet Protocol (IP) often IPv6, the latest version. They must follow IP's rules for addressing and routing data. Solana nodes send serialized transactions to each other using the UDP transport protocol, which is fast and efficient.

However, if a transaction (UDP packet) is larger than the network’s Maximum Transmission Unit (MTU), it is fragmented (split) by the IP layer, not by UDP itself. UDP simply sends the packets it does not handle fragmentation If any fragment is lost in transit, the entire transaction is lost, since UDP provides no error correction or retransmission.

So this fragmentation is handled by Solana,In order to avoid fragmentation the packt/transaction size should be less then MTU(Maximum Transmission Unit) which is typically **1280 bytes**.After Removing the headers(IP and UDP header = 48 bytes),
the remaining 1,232 bytes are allocated for transaction size.

### Solana’s New Update: Increasing Transaction Size Limit to 4000 bytes

With Solana’s adoption of modern transport protocols like QUIC (which doesn’t have the same strict MTU constraints as UDP, QUIC is more like cross-breed between TCP(of fragmentation handle) and UDP(fast as f\*ck)), larger messages can be more reliably delivered.

In the older version, where the transaction size was limited to 1,232 bytes, it was difficult to create complex transactions for ZK and DeFi applications. Increasing the number of accounts on the client side (with each account’s public key being 32 bytes) and listing all those public keys could quickly hit the transaction size limit.

Because of this limitations developers should write optmize code where all accounts,user signatures should be squeezed into a tiny 1,232-byte space.

<img
 width="1000px"
 height="550px"
 src="./images/meme_trx_size_limit.png"
/>

With this new [SIMD-296 proposes](https://github.com/solana-foundation/solana-improvement-documents/pull/296/commits/bbc29c909085589989ca5f258550ce4447e68a89) Transaction size limit incresed to **4k bytes**.This change allows for more instructions, larger data payloads, and smoother execution of complex dApps especially those involving multiple interactions or requiring rich metadata.

## 3. Stack Size Limitations

1. Explain about general programming language memory allocation.
2. Write an example and explain all the things realted to it ex:- Frame, Stack, Heap and Data/Globl persistend Data.
3. Explain how solana manages the memory
4. Connect the Dots with Solana BPF Virtual Machine.
5. Explain how can we optmize the Stack Size Limitations.

In Solana programs (smart contracts), there's a limitation on the stack size - the amount of memory allocated for local variables and function calls. The current stack frame size limit is **4KB per frame**.

### What is Stack Memory?

Stack memory is used for:

- Local variables in functions
- Function parameters
- Return addresses
- Temporary data

### Stack Size Constraints

```rust
    // This function / stack frame contains more the 4KB size
    pub fn handle(_ctx: Context<SendAmount>) -> Result<()> {
        // Allocate a 1MB buffer on the stack

        let mut buffer = [0u8; 50000]; // 50,000 bytes (each u8 is 1 byte, so 50,000 * 1 = 50,000 bytes)

       // Fill the buffer with some pattern
        for i in 0..buffer.len() {
            buffer[i] = (i % 256) as u8; // Fill with repeating 0..255
        }

        // Find the sum of all bytes (just for fun)
        let sum: u64 = buffer.iter().map(|&b| b as u64).sum();
        println!("Sum of all bytes: {}", sum);

        Ok(())
    }
```

> ⚠️ **Warning Example:**
>
> When you allocate a large buffer or array on the stack, you may encounter an error like:
>
> ```
> Error: Function _ZN16cu_optimizations16cu_optimizations6handle17h60f6d64a7e5552edE Stack offset of 1048576 exceeded max offset of 4096 by 1044480 bytes, please minimize large stack variables. Estimated function frame size: 1048576 bytes. Exceeding the maximum stack offset may cause undefined behavior during execution.
> ```
>
> This warning indicates your function's stack frame exceeds Solana's 4KB stack limit. To resolve this, avoid allocating large arrays or buffers on the stack—use heap allocation or refactor your code to reduce stack usage.

```
error Access violation in program section at address 0x1fff02ff8 of size 8
```

### Solana BPF Memory Regions

| **Region**       | **Start Address** | **Purpose**                                    | **Access Rules**                 |
| ---------------- | ----------------- | ---------------------------------------------- | -------------------------------- |
| Program Code     | 0x100000000       | Executable bytecode (SBF instructions)         | Read-only, execute-only          |
| Stack Data       | 0x200000000       | Function call frames and local variables       | Read/write (4KB per stack frame) |
| Heap Data        | 0x300000000       | Dynamic memory allocation (default 32KB)       | Bump allocator (no deallocation) |
| Input Parameters | 0x400000000       | Serialized instruction data & account metadata | Read-only during execution       |

To work around stack limitations:

1. Use heap allocation when necessary
2. Break large functions into smaller ones
3. Avoid large stack-allocated arrays
4. Use references instead of moving large data structures

[solana_optimization_github](https://github.com/solana-developers/cu_optimizations) <br/>
[rare_skill_blog_post](https://www.rareskills.io/post/solana-compute-unit-price) <br/>
[solana github discussion SIMD-0296](https://github.com/solana-foundation/solana-improvement-documents/pull/296/commits/bbc29c909085589989ca5f258550ce4447e68a89)<br/>
[frank_castle tweet on solana transaction size limit](https://x.com/0xcastle_chain)<br/>
[a great video to undertand about memory mangment in programming](https://youtu.be/vc79sJ9VOqk?si=hTqpylYjBO88hvJ9)

<img
 width="1000px"
 height="430px"
 src="./images/solana_limitation_bye.gif"
/>
