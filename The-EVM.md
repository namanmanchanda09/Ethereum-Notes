# Ethereum Virtual Machine

*The following are explanations and notes made by following the [Jordan McKinney](https://www.youtube.com/watch?v=kCswGz9naZg) EVM video along with some other resources.*

Ethereum has a **WORLD STATE** and the world state contains all the important things in the system. All the tokens, NFTs, account state etc are stored in this world state. At any given time, Ethereum has just `1` world state. There is a history of the world state but most of the time we only care about the current world state. This world state only changes when a new `block` is mined and a block consists of bunch of `transactions` in it. 

![Screenshot 2022-07-06 at 11 51 55 AM](https://user-images.githubusercontent.com/35381035/177482614-ed68ab1a-9579-47ab-bc43-6982b979ab2a.png)

`Each transaction represents a change in the world state.`

So when a block is mined, all the transactional changes are done to the world state and we get a new world state. This ability of the system to change its state when new transaction happen is the core idea behind Ethereum. Everything else that constitutes the system i.e POW, POS, smart contracts are there to help the system function in this way. Now what does these state changes is the `EVM`.

**EVM** takes up these transactions, and make changes to the starting state and computes what the new state would look like. 
