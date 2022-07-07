# Ethereum Virtual Machine

*The following are explanations and notes made by following the [Jordan McKinney](https://www.youtube.com/watch?v=kCswGz9naZg) EVM video along with some other resources.*

Ethereum has a **WORLD STATE** and the world state contains all the important things in the system. All the tokens, NFTs, account state etc are stored in this world state. At any given time, Ethereum has just `1` world state. There is a history of the world state but most of the time we only care about the current world state. This world state only changes when a new `block` is mined and a block consists of bunch of `transactions` in it. 

![Screenshot 2022-07-06 at 11 51 55 AM](https://user-images.githubusercontent.com/35381035/177482614-ed68ab1a-9579-47ab-bc43-6982b979ab2a.png)

`Each transaction represents a change in the world state.`

So when a block is mined, all the transactional changes are done to the world state and we get a new world state. This ability of the system to change its state when new transaction happen is the core idea behind Ethereum. Everything else that constitutes the system i.e POW, POS, smart contracts are there to help the system function in this way. Now what does these state changes is the `EVM`.

**EVM** takes up these transactions, and make changes to the starting state and computes what the new state would look like. There can be several views of the world state but what's important is that the world state simply consists of accounts. 

![Screenshot 2022-07-06 at 8 11 29 PM](https://user-images.githubusercontent.com/35381035/177577227-b634c2c4-6069-47dd-8585-dfc22ee4622e.png)

There are `2` types of accounts - **Externally Owned Account**(EOA) and **Contract Account**. Contract accounts also consist of storage and code. Code is immutable but storage can change. 

![Screenshot 2022-07-06 at 8 14 01 PM](https://user-images.githubusercontent.com/35381035/177577793-a6c01616-bf00-4223-958d-409d3de706cc.png)

All these account objects are stored in a data structure called `Merkle Patricia Tree`. Why and how these account addresses are stored inside the Patricia Tree can be skipped for now. What's important for the momment is the *EOA* only consists of little data i.e the nonce and the balance. But the *Contract account* also consists of a `code hash` and a `storage root`. Now this storage root itself is made up of another Patricia Tree. Broadly viewing the World State now - `the Ethereum World State simply consists of some # of Merkle Patricia Trees connected together.`

<img width="670" alt="Screenshot 2022-07-06 at 8 19 03 PM" src="https://user-images.githubusercontent.com/35381035/177578849-a9b5c6bb-56cd-4781-91c7-b12c995f1aa7.png">

The EVM makes changes to these giant data structure. A merkle patricia tree contains a root i.e the `State Root` or the `Storage Root`. These roots are formed by taking hashes of the data present in the nodes of the tree. So any change in the tree would change the root of the tree. Hence everytime a new block is mined - i.e some transactions or even just one transaction occurs, that means a change in at least one account and so the root of the tree changes and transitions to a new world state. The below diagram represents this.

![Screenshot 2022-07-06 at 8 45 51 PM](https://user-images.githubusercontent.com/35381035/177584894-eef82559-cd34-4b92-a147-24930c6074e9.png)

The EVM is a `virtual machine`. Every node in the Ethereum system must get the **exact** same results given an identical starting state and transaction. Virtual machines are used for this because they run exactly the same regardless of underlying hardware, OS, etc. 

![Screenshot 2022-07-06 at 9 20 52 PM](https://user-images.githubusercontent.com/35381035/177592025-3c7687d1-324a-4358-82d4-7730b0c22285.png)

EVM layer node consists of a physical computer running any OS - windows, linux, mac. On top of that, there's an Ethereum Node - `Geth` or `Parity`. On top of that is the Ethereum Virtual Machine. The following is a simple EVM architectue. All the implementations of EVM maintain this virtual machine.

![Screenshot 2022-07-06 at 10 55 30 PM](https://user-images.githubusercontent.com/35381035/177608493-eabf1731-7bfd-4d37-ac6f-c9f6ddfb4865.png)

The Virtual ROM pulls up the EVM code of the contract that the EVM would be working on. The Machine State is volatile. So whenever the EVM interact with a contract - it pulls up the contract code and the contract storage from the world state. 

![Screenshot 2022-07-06 at 11 05 59 PM](https://user-images.githubusercontent.com/35381035/177610144-35cd4b93-f33c-4bcd-bddd-124bdc56ce24.png)

The EVM - the `CPU` of the World Computer is single threaded. For each transaction: code of contract being called is loaded, program counter set to zero, contract storage loaded, memory is set to all zeros, all block and environment variables are set. if gas limit is hit(out of gas exception): no changes to Ethereum state are applied, except sender's nonce incrementing and their ETH balance going down to pay for wasting the EVM's time. 

EVM understands the bytecode compiled from the solidity/vyper language. EVM translated the bytecode into opcodes and each opcode has a gas associated with it. That's the real world cost needed to execute that particular opcode by the EVM. 
   
