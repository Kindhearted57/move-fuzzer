
## TL;DR

This project enhances Sui-fuzzer to support Move smart contracts on Aptos and introduces gas-aware fuzzing.

Relevant code can be found in the master branch of [this](https://github.com/Kindhearted57/sui-fuzzer) repo, I built on top of the current Sui-fuzzer codebase.

1) **Gas-aware feedback**: Added gas usage as a feedback metric to guide stateful fuzzing with the Aptos backend. 

[Gas info collection - commit # bdcfb55](https://github.com/Kindhearted57/sui-fuzzer/commit/bdcfb5579240ed474a0a5f21267387160b07bcb0)

2) **Aptos compatibility**: Adapted Sui-fuzzer to work with Move smart contracts on Aptos.

[commit # 949864d](https://github.com/Kindhearted57/sui-fuzzer/commit/949864d1b1c9d9e943abee9e7bf1946cefe4b8c6)

[commit # dde8691](https://github.com/Kindhearted57/sui-fuzzer/commit/dde869173bac74279ae5a8fcac6981b4f21ca854)

In the future, I want to explore bytecode level fuzzer for Move smart contracts, optimize input corpus generation and integrate concolic fuzzing assisted by symbolic execution.

## Setup Instructions

Build Sui - `./build-sui.sh`

Build Aptos - `./build-aptos.sh`

Run stateless Sui fuzzer - 

```
cargo run --release -- \
  --config-path ./config_stateless.json \
  --target-module fuzzinglabs_module \
  --target-function fuzzinglabs`
```

Run stateful Aptos fuzzer - 

```
cargo run --release -- \
  --config-path ./config_aptos_stateful.json \
  --target-module amm_factory \
  --functions "create_pair,set_admin"
```



## Move Smart Contract Fuzzing

### Sui-fuzzer

Sui-fuzzer (FuzzingLabs) is a specialized tool for discovering bugs and vulnerabilities in Sui Move smart contracts. It carries out both stateless and stateful fuzz testing for smart contracts at the source-code level, supporting contracts executed on the Sui Virtual Machine.



### Different Modes

Sui-fuzzer provides two fuzzing modes:

1) **Stateless fuzzing**: Runs individual Move module functions in isolation.
2) **Stateful fuzzing**: Goes beyond isolated tests by generating sequences of Move calls, testing smart contracts across multiple state transitions.

### Fuzzing Framework

**Coverage based fuzzing**: 

Currently, coverage based fuzzing is only applied to stateless fuzzing in Sui-fuzzer. Stateful fuzzing lacks a meaningful feedback mechanism because transactions can span multiple contracts, depend on complex sequences of object state changes, and interact with shared resources. Unlike stateless fuzzing, where each input is isolated and coverage directly reflects which execution paths have been explored. In stateful fuzzing, the effect of a single input is highly context-dependent.


**Sui-Fuzzer Workflow**:

1. **Input Generation**: Start with random or structured inputs for Move smart contract functions
2. **Execution**: Inputs are executed on the Sui Move VM
3. **Feedback**:
Stateless - In each step, inputs that explore new paths are saved and mutated to generate new test cases
Stateful - There is no feedback mechanism as described in the coveraged based fuzzing section.
4. **Mutation**:
There are two levels of mutation for stateful fuzzing, sequence level and parameter level. For stateless fuzzing there is only parameter level.
Sequence level: Generates random call sequences by combining `target_functions` and `fuzz_functions`, randomly duplicates functions from existing sequence to create longer sequences. Shuffle entire sequence to randomize execution order.
Parameter level: each function in sequence gets its parameters mutated.
4. **Crash & Bug Detection**: Fuzzer monitors for crashes, aborts or invariant violations. Detected issues are recorded along with the input sequence that triggered them.
5. **Output**: Logs of interesting inputs, sequences, coverage statistics, and potential vulnerabilities.


## My Approach & Contribution

### Goal

#### Adapt the Sui-fuzzer to the Aptos environment

This project extends Sui-fuzzer to support Aptos, enabling comprehensive fuzzing of Aptos Move smart contracts while leveraging the proven fuzzing techniques developed for Sui.

##### Key Technical Adaptations

The adaptation from Sui to Aptos required addressing fundamental architectural differences between the two blockchain platforms:

**1) Dependency management & Build System**

**Sui**: Uses sui-specific Move runtime and VM components

\- `sui-move-build`, `sui-json-rpc-types`, `sui-types`, etc.

**Aptos**: Uses aptos-specific Move dependencies

\- `aptos-language-e2e-tests`, `aptos-types`, `aptos-vm`, etc.

Due to conflicting move library versions, I separate `Cargo.toml` file into `Cargo-sui.toml` and `Cargo-aptos.toml`, so that in each ecosystem the respective move version will be used.

**2) Transaction execution architecture**

**Sui**: Object-centric model 

\- Uses `ProgrammableTransactionBuilder`

\- Transactions operate on objects with unique ObjectIDs

\- Features object versioning and ownership semantics

**Aptos**: Account-centric model

\- Uses `EntryFunction` + `TransactionPayload`

\- Transactions are account-based operations

\- Uses `FakeExecutor` for testing


**3) Transaction argument conversion**

Implemented conversion logic to handle the different argument passing mechanisms between Sui's object-based parameters and Aptos's account-based function parameters.

**4) VM integration approach**

Sui: Uses `send_and_confirm_transaction()` with a test executor infrastructure

Aptos: Uses `FakeExecutor::execute_transaction()` with e2e test infrastructure

Overall, this adaptation enables cross platform fuzzing capabilities for move smart contracts, reuses core fuzzing component, unifies fuzzing framework.


#### Optimizations for Sui-fuzzer 


There are aspects in the fuzzing component of Sui-fuzzer that can be optimized, specifically, in this project, I optimized the feedback part.
Currently, Sui-fuzzer uses coverage as the primary feedback mechanism (and this is only for stateless fuzzing). Only inputs that increase code coverage are deemed "interesting" and retained for further mutation.

Coverage based fuzzing has a long history in traditional software testing, the main goal is to explore as many code paths as possible in large programs (e.g., operating systems, browsers, etc.). In these contexts:

1) Execution paths are numerous and complex, and triggering a crash or memory corruption is the primary concern.

2) Resource constraints are mostly ignored because the program is assumed to run in a general-purpose environment, not a constrained virtual machine.

However, smart contract fuzzing is fundamentally different, for at least the following reasons:

**1) Resource-constrained execution**

\- Every operation consumes gas within strict limits

\- Valid execution paths that approach gas limits represent critical edge cases

\- Coverage alone cannot identify gas inefficient but functionally correct paths


**2) Economic consequences**
Coverage metrics don’t account for these risks:

\- Bugs directly translate to finacial losses (token drainage, 
economic exploits)
\- DoS through gas exhaustion is also a concern



**3) Deterministic & transactional context**
\- Smart contracts run deterministically on-chain, often interacting with shared state. This differs from traditional applications, where crashes or undefined behavior are the primary target.
\- State-dependent vulnerabilities may not manifest through coverage alone


Therefore, in my opinion, in the context of smart contracts, reaching "novel paths" is not sufficient, the fuzzer must also consider gas limit.

I implemented in this project gas-aware feedback, where I track gas consumption patterns along with code coverage. In the mutation step, I prioritize inputs that approach gas limits or show unusual consumption.

### Future Work

#### Other potential feedback metrics

We can also consider using contract state changes as feedback signals, track state transition uniqueness beyond code coverage.


#### ByteCode level fuzzer

Sui-fuzzer is a source-code level fuzzer. The adaptation I made also works on source-code level.

The advantages of a source code level fuzzer include:

\- **Clearer input-behavior mapping**: Direct correlation between fuzzer inputs and source code execution paths, more developer friendly.

\- **Better error reporting**: Stack traces and error messages reference original source code, not bytecode.

\- **Developer-friendly output**: Discovered vulnerabilities are reported in a way that developers can better understsand.


However, in the Solidity ecosystem. We can see lots of fuzzers that target bytecode level. Part of this can be attributed to its early dominance on Ethereum, Solidity has a more developed ecosystem. Solidity benefits from a rich ecosystem of bytecode-level analysis tools, for example:

\- Symbolic execution engines (Mythril, Manticore)

\- Static analysis frameworks (Slither, Securify)

\- Bytcode decompilers and analyzers

\- Extenstive EVM tooling and libraries

This maturity makes bytecode-level fuzzing more feasible for Ethereum, while in contrast, the Move ecosystem currently lacks these relavant tools. Therefore, source code level fuzzing is more practical.

However, bytecode-level fuzzing offers advantages that source-code level fuzzing cannot achieve:


**\- Language agnostic:** Fuzzer can analyze contracts written in different Move dialects

**\- Compilation agnostic:** Fuzzer does not need to be concerned about compiler versions. Issues introduced during compilation optimizations can also be detected

**\- Source code agnostic** Not all on chain contracts have public available source code. It is important to work on bytecode level contracts.

**\- ABI-level fuzzing:** Direct testing of contract interfaces as they appear to external callers

Therefore, in my opinion, it is important to build fundamental toolkit to improve the ecosystem, and work towards bytecode level fuzzing.
### Reference

FuzzingLabs/Sui-Fuzzer: https://github.com/FuzzingLabs/sui-fuzzer/tree/master/src

Fuzzland/ityfuzz: https://github.com/fuzzland/ityfuzz

meridian_amm contracts: https://explorer.movementnetwork.xyz/account/0xfbdb3da73efcfa742d542f152d65fc6da7b55dee864cd66475213e4be18c9d54/modules/packages/MeridianAmm?network=mainnet

razor_amm contracts: https://explorer.movementnetwork.xyz/account/0xc4e68f29fa608d2630d11513c8de731b09a975f2f75ea945160491b9bfd36992/modules/code/amm_router?network=mainnet