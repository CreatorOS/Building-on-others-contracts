# Building on others’ contracts
On Ethereum we recently saw a project called LootProject.com launch

It is a very simple project in which NFTs are created. These NFTs are a collection of some attributes - white text on a black background.

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/7f9d6ec5-50ad-427d-971e-1811763f6124.jpg)

It is so simple yet powerful, because over 200 games and projects have been built on top of these NFTs. 

When projects like this launch, many more projects suddenly get enabled. This is called composability of smart contracts. All the functions in a smart contract are, by default, accessible and can be called by other smart contracts. 

In this quest, we’ll create a simple contract that calls functions in another contract. You can then use the same mechanisms to interact with practically every contract that has ever been deployed to near!

This quest is written in Rust and requires that you have done the previous quests on creating an NFT - because that covers a lot of basics on how to write, compile and deploy rust smart contracts.
## Redeploying NFT contract
For the sake of simplicity, we will have to make it really simple. 

We’ve already created an NFT contract in the previous quest. We will build on top of this.

We’ve already deployed it. We’ll need the account id of the deployed contract. If you don’t remember it, now should be a good time to re-deploy it and copy the account id. 

Every contract that you deploy will always be deployed to a unique account. As we had seen earlier any account can have utmost one contract inside it. That means that every contract that you deploy must be deployed to a new account. 

```

$ cargo build --target wasm32-unknown-unknown --release 

$ near dev-deploy --wasmFile target/wasm32-unknown-unknown/release/rust.wasm

```

Copy the new account ID that has been generated for you by `near dev-deploy` and use this in place of CONTRACT_ACCOUNT_ID below

```

$ near call CONTRACT_ACCOUNT_ID get_token_owner ‘{“token_id”: 1}’ --accountId yourname.testnet

```

Now that we’ve called the function from the commandline, we’ll now call the same function from rust using another contract we’ll deploy.
## External Functions
Create a new project (refer to the first near quest on rust from [https://questbook.app](https://questbook.app) for how to). 

```

use near_sdk::borsh::{self, BorshDeserialize, BorshSerialize};

use near_sdk::{env, near_bindgen};

near_sdk::setup_alloc!();

\#\[near_bindgen\]

\#\[derive(BorshDeserialize, BorshSerialize)\]

struct ExternalCaller {

}

impl ExternalCaller {

    pub fn caller() {

        env::log(b"Hello");

    }

}

```

To call functions in other contracts, we must first define the interface of the functions we’d want to call. We’ll also mark it as an external contract interface using `ext_contract` decorator.

```

\#\[ext_contract(ext_ft)\]

trait NonFungibleTokenBasic {

    fn get_token_owner(&self, account_id: String) -> String;

}

```

We will be calling only one function from this contract, so we’ll only define this particular function in our trait. 

The signature of the function is the same as the signature of the function in the lib.rs of the nearnft project from the previous quest. 

Then we will define the callback function. This is the function the other contract (the nft contract) will call after processing the function. The return of the called function will become the parameter of this callback function. 

```

\#\[ext_contract(ext_self)\]

trait SelfContract {

    fn my_callback(&self, account_id: String) -> String;

}

impl SelfContract {

  pub fn my_callback(&self, account_id: String) {

  }

}

```

There’s a very specific reason why the way to call functions is using callbacks. Let’s dive deeper into that for a moment.
## Sharding in Near
A blockchain is a world computer. Anybody can connect their laptop to the world computer and provide their computation. WHenever someone calls a function, makes a money transfer etc, all the computers process these transactions. By processing, they are validating if the transaction is legal - i.e. if the user has as much money she’s trying to send, the function called on a contract actually exists. Then they process the transaction. Processing a transaction means actually running the function when a contract function is called, or updating the account balances of the users involved in a money transfer. 

Once the computers do the computation, they participate in a voting. They vote on “yes, these are the updated balances of the accounts after this money transfer takes place.” or “Yes the return of this function call is 42”. This voting is what gives blockchains the security. No one computer can cheat and say “No, the updated balances of the accounts of the money transfer is actually ths…”.

A simplistic view of the blockchain is to say that every computer verifies every transaction. But this makes the system slow. So near has implemented what is called as sharding. 

The entire set of computers (known as validators) are divided into smaller sets called shards.

Each shard will compute only some transactions. Each shard manages a particular set of accounts. Every validator in that shard will verify the transactions that involve an account in that shard. 

So, if a function is called on a contract that lives in account 1, it will be processed by all the validators in shard A. If a function is called on a contract that lives in account 2, it will be processed by all the validators in shard B and so on. 

The distribution of accounts into shards happens almost randomly. So when we’re calling a function on a particular contract from another contract, we can’t be sure that the two function calls will be executed on the same machine - because the two contracts might reside on different shards. 

So all external function calls assume by default that contracts will always exist on different shards. And hence, all the external contract calls are network calls. Network calls always follow a callback pattern, because there’s usually latency involved in data transfer over the wire. 

It  is hence, we’ll use a callback rather than simply calling the function and using it’s return statement.
## Defining the calling function
```

pub fn call_external(&self, u128 &token_id) -> Promise {

    ext_ft::get_token_owner(

        token_id, 

        &"dev-account-id", //acc id of the deployed nft contract

        0, //amount

        5_000_000_000_000,         //gas

    )

 

    .then(ext_self::my_callback(

        &env::current_account_id(),

        0,                

        5_000_000_000_000,

    ))

}

```

So when we call `call_external`, it will call `get_token_owner` and that in turn will call `my_callback` with the return value as the parameter to the function.

There are 2 more parameters in both `ext_ft` and `ext_self` that is `0, // amount` and `5_000_000_000_000 //gas`

In a blockchain smart contract function call, not only can you call functions with parameters, but you can also send money. The parameter `amount` defines how much money in Near tokens we want to send to the function. We will, in a later quest, see how to accept near tokens in a function call. 

The last parameter is called `gas`. We won’t get too deep into it, but all that you need to understand is for running any function on the blockchain, you need to pay a transaction fees. This transaction fees is paid to the validators who are running your functions. A major engineering challenge while writing smart contracts is to write them in an optimized way such that the transaction fees for running functions is kept to a minimum. But that is a topic in itself for a later date :)
## What next?
Now that you have called functions from other contracts, can you write a contract that accepts a token id, checks if an nft with that token id has already been minted - and if not, mint the token by calling the `mint` function.