---
layout: post
title: How do Bitcoin mining pools pay their miners?
description: Insights of a data analytics project with onchain examples
categories: blockchain
tags:
- blockchain
- bitcoin
---

Since the price of Bitcoin surged in the 2017 Q4,
many mining pools have surfaced.
Over the globe there are over 20 active and recognizable mining pools.
You can find more pools and their addresses from
[this GitHub repo](https://github.com/btccom/Blockchain-Known-Pools/blob/master/pools.json).
This article explains a particular type of routine job in a Bitcoin mining pool:
how does it pay Bitcoins to its miners?
Actually you will find out that it might not be the same as you thought.

### Directly from coinbase addresses

This is the simplest way.
SlushPool has been using this method for some time.
You can easily find payout transactions miners from address
[1CK6KHY6MHgYvmRQ4PAafKYDrg1ejbH1cE](https://btc.com/1CK6KHY6MHgYvmRQ4PAafKYDrg1ejbH1cE)
which is the publicly known coinbase address of SlushPool.
On a typical day you can observe more than five transactions from this address
each of which is with a huge number of outputs to various addresses -
it is the typical way of identifying payments from a mining pool to its miners.

Other pools that also use this method are
BTC.TOP and Huobi Pool.
However, you can find that BTC.TOP also uses intermediate addresses to pay its miners.

### Pay via intermediate wallets

This is probably the most common practice of pools.
Take [BTC.com](https://btc.com) for example,
you can find its coinbase address is
[bc1qjl8uwezzlech723lpnyuza0h2cdkvxvh54v3dn](https://btc.com/bc1qjl8uwezzlech723lpnyuza0h2cdkvxvh54v3dn).
There are many transactions associated with this address,
most of which are coinbase transactions.
However there are some Bitcoins sent by it to other addresses,
for example,
[3FxUA8godrRmxgUaPv71b3XCUxcoCLtUx2](https://btc.com/3FxUA8godrRmxgUaPv71b3XCUxcoCLtUx2)
in transaction
[d094eca893253c12bb53f55ed834281349eda9e59505fa91d07e114fd616aa02](https://btc.com/d094eca893253c12bb53f55ed834281349eda9e59505fa91d07e114fd616aa02).

Therefore,
[3FxUA8godrRmxgUaPv71b3XCUxcoCLtUx2](https://btc.com/3FxUA8godrRmxgUaPv71b3XCUxcoCLtUx2)
is one of the intermediate addresses used by BTC.com.
Every day there are a few transactions from it to many addresses,
probably thousands of outputs in a single transaction,
and this is how BTC.com pays its miners.

### AntPool: payments chain

AntPool is a very special case and
its payment pattern is the most difficult to trace.
The whole process of payments is like a chain or loop.
Almost every day it starts from in the morning
and last until late afternoon.

* Generate a new wallet
* Use the intermediate address to pay 101 addresses
* Among the 101 recipients 100 are miners,
while the one newly generated wallet receives _all_ the change
* Use the balance in the change wallet to pay another 100 miners + 1 new wallet
* Repeat the process above until all miner addresses are paid

The end result is that all the intermediate change wallets are empty and
they will _not_ be reused.
Only the very last change wallet has some Bitcoins in it.
Personally I like this process because it gives the miners highest level of
privacy.
If you wish to find out all miners' addresses under AntPool by hands,
you have to traverse many transactions.
But for other pools, you only need to checkout less than ten transactions,
probably even less than five.

### F2Pool: pay without transaction fees

Everyday F2Pool broadcasts a transaction with zero transaction fee
to pay its miners.
After that, F2Pool tries to mine it because
no other miner is willing to mine a transaction without any fee.
This approach saves money, apparently.
A major disadvantage of it is there are huge
uncertainties in the time when its miners receive Bitcoins.
