## Scope

### Move Smart Contract Fuzzing

#### Existing Fuzzer

Sui-fuzzer (FuzzingLabs) is a specialized fuzzing tool for discovering bugs and vulnerabilities in Sui Move smart contracts. It carries out both stateless and stateful fuzz testing for smart contracts written in Move and will be executed in Sui virtual machine.

##### Different Modes

Sui-fuzzer has two modes:

1) Stateless fuzzing: Runs individual Move module functions
2) Stateful fuzzing: Goes beyond isolated tests by generating sequences of Move calls, enabling it to test smart contracts across multiple state transitions

##### Fuzzing Framework

**Coverage based fuzzing**: Currently, coverage based fuzzing only applies to stateless fuzzing, stateful fuzzing does not have any feedback mechanism. In stateful fuzzing, it is difficult to use coverage as a meaningful feedback signal because transactions can span multiple contracts, depend on complex sequences of object state changes, and interact with shared resources. Unlike stateless fuzzing, where each input can be treated in isolation and coverage directly indicates which execution paths have been explored, in stateful fuzzing the effect of a single input is highly context-dependent.
**Property-based fuzzing**: xxx
**Sui-Fuzzer Workflow**:
1. Input Generation: Start with random or structured inputs for Move smart contract functions
2. Execution: Inputs are executed on the Sui Move VM
3. Mutation: Inputs that explore new paths are saved and mutated to generate new test cases
4. Coverage Feedback: VM tracks program counter, inputs that increase coverage are considered "interesting" and prioritized for further mutation
5. Crash & Bug Detection: Fuzzer monitors for crashes, aborts or invariant violations. Detected issues are recorded along with the input sequence that triggered them.
6. Output: Logs of interesting inputs, sequences, coverage statistics, and potential vulnerabilities.





### My Approach & Contribution

#### Goal

##### Adapt the Sui smart contract fuzzer idea to the Aptos environment

* Changing the execution environment from Sui's API to Aptos

##### Optimizations for Sui-fuzzer

There are aspects in the fuzzing component of Sui-fuzzer that can be optimized:

* Feedback

Currently, Sui-fuzzer relies coverage to feedback fuzzing. Only inputs that increase coverage are considered "interesting"

Coverage based fuzzing has a long history in traditional software testing, the main goal is to explore as many code paths as possible in large programs (e.g., operating systems, browsers, network daemons). In these contexts:

1) Execution paths are numerous and complex, and triggering a crash or memory corruption is the primary concern.

2) Resource constraints are mostly ignored because the program is assumed to run in a general-purpose environment, not a constrained virtual machine.

However, smart contract fuzzing is fundamentally different:

Resource-limited execution – Each contract executes within a gas-constrained VM. A path that is valid but consumes excessive gas is critical to detect. Coverage alone cannot capture this.

Economic consequences – Bugs can have direct financial impact (e.g., draining tokens, DoS attacks). Coverage metrics don’t account for these risks.

Deterministic & transactional context – Smart contracts run deterministically on-chain, often interacting with shared state. This differs from traditional applications, where crashes or undefined behavior are the primary target.

Therefore, in my opinion, in the context of smart contracts, reaching "novel paths" is not sufficient, the fuzzer must also consider gas limit.


### VM Fuzzer

### Reference

FuzzingLabs/Sui-Fuzzer: https://github.com/FuzzingLabs/sui-fuzzer/tree/master/src