<pre>
  AIP: 81
  Title: CORE-VM module specifications
  Authors: Kristjan Kosic <kristjan@ark.io>
  Status: Draft, Rejected or Active
  Discussions-To: https://github.com/arkecosystem/AIPS/issues/81
  Address: Ark address used to collect votes for the specific AIP
  Type: Standards
  Category only required for Standards Track: <Core>
  Created: 2019-04-09
  Last Update: 2019-05-08
  Requires: AIP-11, AIP-18, AIP-29
  Replaces: AIP-12
</pre> 

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [-](#-)
- [Check items list/Questions/To Address](#check-items-listquestionsto-address)
- [Motivation](#motivation)
    - [Why?](#why)
    - [General overview](#general-overview)
        - [Deployment stage](#deployment-stage)
        - [Forger: Execution stage](#forger-execution-stage)
        - [Storage](#storage)
        - [Interfaces to other modules](#interfaces-to-other-modules)
        - [Rebuilding capability](#rebuilding-capability)
        - [General constraints](#general-constraints)
            - [Hardware limitations](#hardware-limitations)
            - [Audit function](#audit-function)
            - [Safety and security](#safety-and-security)
            - [Error handling and recovery](#error-handling-and-recovery)
- [Technical Specification(initial/will be further updated/aligned with reference implementation)](#technical-specificationinitialwill-be-further-updatedaligned-with-reference-implementation)
    - [Transaction Types](#transaction-types)
        - [TX: Deployment of script](#tx-deployment-of-script)
            - [Field descriptions](#field-descriptions)
        - [TX: Execute script method/A message call](#tx-execute-script-methoda-message-call)
            - [Field descriptions](#field-descriptions-1)
    - [Core-vm module mechanics](#core-vm-module-mechanics)
        - [General checkpoints](#general-checkpoints)
        - [Ensuring deterministic execution](#ensuring-deterministic-execution)
        - [Script Interface](#script-interface)
        - [Compilation of DApp](#compilation-of-dapp)
        - [Execution of script](#execution-of-script)
            - [Storage definition](#storage-definition)
            - [Inter-transaction execution](#inter-transaction-execution)
        - [Storage capabilities](#storage-capabilities)
- [Reference implementation](#reference-implementation)
    - [Copyright](#copyright)
- [References](#references)

<!-- markdown-toc end -->


## Abstract
The purpose of this document is to define specifications and expectations related to building ARK VM in terms of ARK’s technology stack, namely running as a core-plugin and enabling virtual machine execution inside a core-module.


## Check items list/Questions/To Address
  * [ ] AIP: define transaction outside of core-mode, e.g. inside our new module (store contract transaction)
  * [ ] Size, memory, execution stack limitations
  * [ ] Size of script
  * [ ] Number and size of storage options
  * [ ] Private smart-contracts (e.g. whitelisting addresses)
  * [ ] Rebuild from zero - saving data on the blockchain
  * [ ] Storage

## Motivation
The goal of this AIP is to enable script execution inside the `core` technology landscape and run this as a module, if enabled.

### Why?
In comparison to AIP-29 that provides custom application logic in the form of scripts/smart-contracts delivered via plugin. This introduces a difference between trust-less execution or internal transfers and state and "normal" transaction broadcast (transfer transaction validation and execution), such as:
- no special plugin installation and deployment is needed for ark-script to work
- modifications are possible via REST endpoints, no need for node operators to manage and update plugin logic 
- automated trust-less execution
- AIP-29 requires transactions to be signed/emitted to the network, thus making automated execution of secure functionalities (internal transfers) a bit more difficult as compared to the `internal calls` that change state while being executed (for example as with ETH smart contracts). This adds additional layer of security, as every transaction needs to be signed. By doing so we limit the automation capability of trust-less execution.
- state storage - custom transaction implementation via AIP-29 introduces options for additional storage, but is still limited within its implementation and replication capabilities (related to transaction type). By introducing a new VM Engine we need to introduce script state storage. Storage must be fully deterministic via blockchain replay logic.


| Pros                                                                                                                          | Cons                                     |
| ----------------------------------                                                                                            | ----                                     |
| javascript / typescript codebase                                                                                              | running VM environment not as big as ETH |
| isolated running environment                                                                                                  | rebuilding logic/protocol level          |
| adding custom behavior in our control (destroy contract, retire)                                                              | tooling?                                 |
| allowing more functionality inside script instead of introducing new tx types for each new functionality/or plugin deployment |                                          |
| no need to run a sidechain (less issues when searching for delegates)                                                         |                                          |
|                                                                                                                               |                                          |


## General overview
The goal of this proposal is to launch VM engine inside the `core` technology landscape and run it as a module, if enabled. Looking further at the virtual machine life-cycle and core execution lifecycle we have the following interaction points with our core.

A smart contract can be thought of as an script, as it is both, an Application and supports decentralized execution. SmartContracts can be invoked when a transaction is added to a blockchain (e.g. sending funds to a specifics smart contract address with parameters). Besides running in a decentralized way and supporting transfer of funds via internal message calls, smart contracts also support storing of state data related to script execution.

### Deployment stage
Deployment stage introduces new transaction types (see below). Deployment stage must validate, test, store new script. Introducing of new transaction types will be needed.

Smart contract/script deployment consists of several steps:

- compiling the contract
- validating the contract

### Forger: Execution stage
Script execution during block creation stage. A script function call is sent via script call transaction. 
script is executed inside the virtual-machine(`isolated-vm`) thus changing the state/storage options.

For the execution engine we looked at several options, but wanted to stay on the `javascript` code base. The following isolated environments where taken into account:
1. Isolated-VM - https://github.com/laverdet/isolated-vm (based on first research, looks like the most viable and secure option - using Google's V8 OSS JavaScript Engine)
2. VM2 - https://github.com/patriksimek/vm2 (used in desktop wallet)
3. NodeJs VM - https://nodejs.org/api/vm.html. (basic and limited functionality of nodejs)

VM execution engine was selected based on the security and memory management and overall execution in an isolated environment. State should be passed into the `vm` and used as
the current source of truth. Currently the best candidate is `isolated-vm`.

### Storage
Virtual Machine will introduce a new storage option for scripts to store state in a secure and distributed way. A Light key-value database can be used, as state can be reproduced via rebuild - from transactions (blockchain replay). All script calls need to be stored (calls and parameters) so we can replay and execute on each node (part of protocol).

### Interfaces to other modules
- core-blockchain
- core-state

### Rebuilding capability
script data history (any calls with all the details) must be saved on the blockchain. We could use our asset field where we store this. Add the execution/loading logic into SPV process, when chain is validated and rebuilt from 0.

### General constraints
#### Hardware limitations
- size limitations related to  overall script size
- memory usage limitations 
- isolated running environments per script
- memory limitations
- CPU time limitations (`while (true) {}` limit)

#### Audit function
- strict logging of outcomes
- rebuilding relevant data must be added on the blockchain
- a separate execution log of the VM engine

#### Safety and security
- sandboxed environment
- timeout 
- limited number of operations and execution scope
- private contracts (limited by white-listed senders)
- general class implementation / as guideline and access to blockchain (wallet-manager)

#### Error handling and recovery
General timeout for all execution points. Has to be “executed” in a predefined timeout. Confirmation and compilation stage will be done in the deployment phase.

# Technical Specification(initial/will be further updated/aligned with reference implementation)
The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations.

*THIS IS WIP, AND IT WILL CHANGE ALONG THE LINES AS REFERENCE IMPLEMENTATION CHANGES*

## Transaction Types
New transaction types need to be implemented in order to support interaction with the blockchain in terms of sending/loading script to the blockchain and giving possibilities to call the smart contract methods. This part references AIP-12 and continues on the definition of new transaction types related to dApp deployment and calling of their methods (execution).

### TX: Deployment of script
Transaction type scriptDeployment will send dApp payload to the chain `post/transactions` endpoint. 
script developer implements dApp following the predefined interface provided from `core-vm`. SmartContract interface defines the structure and the methods an ARK smart contract must implement, as well as access to super class (TBD/Link).
A return value is status of script deployment, and return address if successfully deployed. 

Size of the source code and duration of compilation(computing power needed) will be taken into account. Dynamic fee will be strongly affected by size.

| Description | Size | Sample |
| ----------  | ---- | ------ |
| type        |      |        |
| fee         |      |        |
| interface   | var  |        |
| source-code | var  |        |

#### Field descriptions
1. interface - descriptions and definition of  public script methods, that will be available via message call functionality
2. source code - javascript source code to be compiled and run

Also think about saving already compiled VM script - to save on blockchain size?

When script Deployment transaction enters the pool, we test it against internal compiled method from the virtual machine. If script is successfully compiled we accept it into the pool and it will be forged when its turn. If not we will return error code to the user sending the dApp.

### TX: Execute script method/A message call
When transaction is successfully deployed, it holds all the required information for anyone to execute the script (if allowed by the contract code). 

| Description      | Size | Sample |
| -----------      | ---- | -----  |
| type             |      |        |
| fee              |      |        |
| amount           |      |        |
| contract-address |      |        |
| method-name      |      |        |
| arguments        |      |        |

#### Field descriptions
1. contract-address
2. method-name (name of one of the methods specified via abi)
3. arguments (k/v pairs of arguments accepted by script call). Must be aligned with dApp abi specifications

## Core-vm module mechanics

  * [x] Integrate `isolated-vm` in the new plugin `core-vm`
  * [ ] Implement first sync mechanism to share state
  * [ ] Solve - return call logic (to provide a base for internal calls and execution)

### General checkpoints
  * [ ] Make basic code run
  * [ ] Sync Variables
  * [ ] Initialize a class
  * [ ] Run methods from class
  * [ ] Save data to common storage/outside of secure execution - just return
  * [ ] Prepare internal transaction calls - based on script output

### Ensuring deterministic execution
One of the decision to use `isolated-vm` was also to ensure deterministic execution on all nodes/hardware configurations. We will introduce the same logic as we have in the transaction validation mechanism:
- internal message call `internal-trans action` will be a function injected into the `virtual-machine` engine at bootstrap time
- store value call `save-value` will be injected during the bootstrap process

```ts
// Get a Reference{} to the global object within the context.
let jail = context.global;

// This make the global object available in the context as `global`. We use `derefInto()` here
// because otherwise `global` would actually be a Reference{} object in the new isolate.
jail.setSync('global', jail.derefInto());

// The entire ivm module is transferable! We transfer the module to the new isolate so that we
// have access to the library from within the isolate.
jail.setSync('_ivm', ivm);

// We will create a basic `log` function for the new isolate to use.
jail.setSync('_internalTransfer', new ivm.Reference(function(...args) {
    // WalletManager logic here
    console.log(...args);
}));

// We will create a basic `log` function for the new isolate to use.
jail.setSync('_storeValue', new ivm.Reference(function(...args) {
    // Storage logic here (also via wallet manager)
    console.log(...args);
}));


// This will bootstrap the context. 
let bootstrap = isolate.compileScriptSync('new '+ function() {
    // Grab a reference to the ivm module and delete it from global scope. Now this closure is the
    // only place in the context with a reference to the module. The `ivm` module is very powerful
    // so you should not put it in the hands of untrusted code.
    let ivm = _ivm;
    delete _ivm;

    // Now we create the other half of the `log` function in this isolate. We'll just take every
    // argument, create an external copy of it and pass it along to the log function above.
    let internalTransfer = _internalTransfer;
    delete _internalTransfer;
    global.internalTransfer = function(...args) {
        internalTransfer.applySync(undefined, args.map(arg => new ivm.ExternalCopy(arg).copyInto()));
    };

    let storeValue = _storeValue;
    delete _storeValue;
    global.storeValue = function(...args) {
        storeValue.applySync(undefined, args.map(arg => new ivm.ExternalCopy(arg).copyInto()));
    };
});

// we execute and share running environments
bootstrap.runSync(context);

```



### Script Interface

### Compilation of DApp
script will be deployed as a normal javascript(transpiled typescript) source-code to the core. Both options can be supported. This will give use the power to leverage the huge community of js/ts developers.

- [ ] Determine to be run locally or on the node/ currently node is preferred / possible errors can be returned?
- [ ] Additional tooling support

### Execution of script 
- Output results of script execution are storage updates or transaction execution - transfer from contract to another contract or normal address.
- We will be running already pre-compiled scripts (execution time)

#### Storage definition

#### Inter-transaction execution

### Storage capabilities

# Reference implementation
Based on research and some demo stuff, currently `isolated-vm` looks like most promising.
- https://github.com/kristjank/virtual-machine/
- https://github.com/kristjank/virtual-machine/packages/core-vm

## Copyright
MIT License

# References
1. Ethereum notes for outgoing transaction https://ethereum.stackexchange.com/questions/24031/how-ethereum-contracts-transfer-ether-without-a-blockchain-confirmation 
2. https://www.mobilefish.com/developer/blockchain/blockchain_quickguide_ethereum_related_tutorials.html
3. https://github.com/takenobu-hs/ethereum-evm-illustrated
4. https://ethereum.stackexchange.com/questions/20781/at-which-point-the-smart-contracts-get-executed
5. https://ethereum.stackexchange.com/questions/765/what-is-the-difference-between-a-transaction-and-a-call 
6. https://github.com/laverdet/isolated-vm
7. https://github.com/gianluca-venturini/isolated-vm-actors/blob/master/src/index.ts
8. https://v8.dev
9. https://v8docs.nodesource.com/

