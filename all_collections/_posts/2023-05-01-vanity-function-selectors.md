---
layout: post
title: Vanity Function Selectors
date: 2023-05-01
categories: []
---

Recently while getting some work done integrating the Etherscan API to determine the deployer of a contract I noticed something strange which sent me down a rabbit hole for a couple hours: Vanity function selectors.  

## Function Selectors in the EVM

If you've ever browsed contract transactions on Etherscan you probably have seen a "Method Name" for some transactions:
![Tx1](/images/vanity-function-selectors/1.png)
![Tx2](/images/vanity-function-selectors/2.png)
![Tx3](/images/vanity-function-selectors/3.png)
  
But you might wonder why other txs you might see don't have these fancy names:  
![Tx4](/images/vanity-function-selectors/4.png)

Well the answer is simply: The contract isn't verified on Etherscan.  
But if you're a bit more curious, you might wonder how we got from A to B, how to get from the ugly transaction input data:  
![InputData1](/images/vanity-function-selectors/5.png)  
To something pretty. If you look closer, you might notice that the first 8 characters of the input data are the same as the Method Name in the transaction.  
This is actually how the EVM (Ethereum Virtual Machine) translates the input data into which function to call.  

But the question still remains of how to get from the function name to the "function selector" (starting 8 characters) or vice versa.  
Enter: The cryptographic hash function Keccak-256: [Introduction to hash functions](https://www.tutorialspoint.com/cryptography/cryptography_hash_functions.htm)  
The solidity compiler simply takes the function name and hashes it with Keccak-256 to get the function selector.  
Lets look at an example:  
![InputData2](/images/vanity-function-selectors/6.png)  
This is a simple USDT Transfer, you can see the method ID is 0xa9059cbb, if we look at the raw input data:  
`0xa9059cbb000000000000000000000000029e71595d4c3ed71f7b9c69258a5fb25f9c9e8500000000000000000000000000000000000000000000000000000002540be400`  
We can see that the first 8 characters are the same as the method ID.  
We can take the function name: `transfer(address,uint256)` (Removing any whitespace and variable names for consistency), hash it with keccak256 and get the same result: `a9059cbb2ab09eb219583...`, note that we just use the first 8 characters of the hash.  

With this knowledge, you can understand how the EVM knows which function to call, and how Etherscan knows which function to display, given a verified contract.

## Vanity Function Selectors

Now that we know how function selectors work, we can look at vanity function selectors.
This is something I completely stumbled upon when working with contract deployment transactions, since they are initiating a contract they don't require a function selector, however a quirk of the solidity compiler is that contract deployments always begin with `0x60806040` ([Technical explanation](https://ethereum.stackexchange.com/questions/129737/new-contract-deployment-pattern-0x60c06040)).
I made a call to the Etherscan API to check the first transactions to a contract, to find the deployment transaction, and noticed something strange:
![Vanity1](/images/vanity-function-selectors/7.png)  
Why is there a function name for a contract deployment??  
Well, for vanity of course!
As it turns out, 8 hex characters isn't actually all that much entropy, only 4 billion possibilities, meaning someone with some time on their hands could find a function name that hashes to a specific function selector.  
Now why Etherscan decided to display this specific function name over any others that exist (or display one at all!) is a mystery to me.. But you can look up other signatures that hash to `0x60806040` and see some [other interesting ones](https://www.4byte.directory/signatures/?bytes4_signature=0x60806040).  

But mum said its my turn with the vanity function selectors! So lets make one ourselves.

## Making a Vanity Function Selector

All we have to do is find a function name that hashes to the function selector we want.  
Luckily for us, there's actually a lot of types in solidity, hundreds, even! So we have a lot of options to choose from.  
I built a small piece of code in Rust, that uses parallelization and 100% of my CPU to find a function signature that hashes to a specific function selector.  
It took less than a half hour to find my selector, but it could be faster or slower depending on your luck, CPU and implementation.  
You can definitely find one with fewer types, but I was able to find one with 5 types: `sanleo(bytes3,bytes30,bytes26,uint8[],uint48[])`  
You can also see that I published a [contract](https://etherscan.io/address/0xf28da06d4b2a6bc6cdc4905fb19aed3df64225ad) and immortalized my selector on Etheruem forever!  

If you're wondering why I didn't post my code, its because you should make your own! Its a fun little project, and you can find a selector that means something to you.

## Is any of this even useful?
Surprisingly, yes!  
You might have noticed if you've been trading NFTs on Seaport v1.4 recently, that your trades appear on Etherscan with the Method `0x00000000`, now this is no mistake!
![Seaport](/images/vanity-function-selectors/8.png)
Opensea actually spent some time mining their own vanity function name: `fulfillBasicOrder_efficient_6GL6yc(tuple parameters)` that hashes to `0x00000000`.  
You can note that they went for the method of changing the function name instead of the parameters, which is why the function name is so long.  
But why? It's actually a gas optimization trick! Generally in the EVM, more zeros = less gas cost (since zeros are cheaper to store than other values), so by making the function name all zeros, they can save a bit of gas on every transaction.  