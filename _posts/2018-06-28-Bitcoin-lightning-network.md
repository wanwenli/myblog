---
layout: post
title: A brief guide to Bitcoin lightning network with examples
excerpt: Technical details, source code and discussion about online resources
category: blockchain
---

Why another post about the lightning network (LN) of Bitcoin?

The primary reason is that
some of the resources online are _not_ up to date.
The technological details of LN are not intuitive enough
and not easy to grasp.
Without examples, confusion and misunderstanding may arise.

If you are a technician, researcher, engineer,
blockchain hobbyist or enthusiast
who already has some knowledge about LN,
this article is for you.
Do not worry if you know nothing about LN,
because this article leads you to the most useful resources
and tell you which piece of information is accurate in them.
Some understanding about
[segregated witness](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki)
is strongly recommended before
reading the remaining part of this post,
as current LN payments heavily rely on it.

In summary this post focuses on:

* what are the reliable resources to read/watch in order to understand LN;
* what info inside these articles and videos is accurate and what is outdated;
* how LN works, roughly;
* how to find LN transactions on-chain.

Unfortunately, this article does _not_ discuss
how lightning network will affect bitcoin ecosystem,
its advantages and drawbacks,
how much transaction rate will improve, or even coin price!

Good news: this article provides **concrete examples** of payments via LN
that have been permanently written on Bitcoin blockchain!
So far I have not seen any other articles that give out examples.

Last but not the least,
this post is my personal humble understanding on LN.
I am open for comments and critics.

### Articles and videos

The GitHub repo
[awesome-lightning-network](https://github.com/bcongdon/awesome-lightning-network)
lists out many resources for LN.
It is a good starting point.
However in my point of view,
there is only one that is sufficiently comprehensive: the
[lightning network paper](https://lightning.network/lightning-network-paper.pdf).
I strongly recommend you to read through it.

You may have come across
[this YouTube video](https://www.youtube.com/watch?v=8zVzw912wPo&t=2317s)
on LN.
It appears as the 2<sup>nd</sup> search result of
"lightning network" on YouTube.
Its weakness is that for bi-directional payments,
how to penalise an old commitment transaction and
revoke all the funds is not clear.
To understand the details of penalty
which is an essential part to ensure the safety and trustworthiness of LN,
please read the LN paper.
Penalty on HTLC transactions follows a very similar mechanism.
Later you will see both successfully claimed and penalised fund on-chain.

Besides, the articles and videos share another fatal issue:
the scripts illustrated in them _cannot_ be found on the most recent blocks.
For example, I am _unable_ to find an input script
that follow the format below which is on page 31 of the LN paper
and implemented in a JavaScript LN protocol,
[yours-channels](https://github.com/yoursnetwork/yours-channels).
However if you do find one, please leave your comment.

```
OP IF
  OP HASH160 <Hash160 (R)> OP EQUALVERIFY
  2 <Ali c e 2> <Bob2> OP CHECKMULTISIG
OP ELSE
  2 <Ali c e 1> <Bob1> OP CHECKMULTISIG
OP ENDIF
```

I suppose it is because
the specifications of LN have been updated and redefined in
[this repo](https://github.com/lightningnetwork/lightning-rfc)
called Basis of Lightning Technology (BOLT).
To be more specific, transactions and scripts are documented in
[BOLT \#3](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md).

### Source code

There is a handful of implementations for LN.
The most popular two are the
[LND](https://github.com/lightningnetwork/lnd) written in Go
and
[lightning](https://github.com/ElementsProject/lightning) in C.
I have found these two repos after encountering the
[mainnet LN explorer](https://lnmainnet.gaben.win/)
which gets readings from them.  

Both of them strictly follow the
[BOLT \#3](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md)
to build transactions and scripts.
Read more detailed source code here:

* [script_utils.go](https://github.com/lightningnetwork/lnd/blob/master/lnwallet/script_utils.go)
in LND
* [script.c](https://github.com/ElementsProject/lightning/blob/master/bitcoin/script.c) in lightning

As you will see soon,
BOLT \#3 is the dominantly (probably the only) observable form on-chain.

### How to find a LN transaction

There are two major forms of transactions that are surely from LN:

* 2-of-2-MULTISIG embedded in P2WSH followed by
a conditional time locked input (for bi-directional channels)
* 2-of-2-MULTISIG embedded in P2WSH followed by at least one HTLC input

The first one is the dominant type of input script in LN and
takes up more than 80% of transaction inputs broadcast by LN.
Later you will see that the second form has two versions,
one for sender and the other for receiver.

### Examples

Here are some example hashes of LN transactions.

##### to_local

It is used to close a bi-directional channel and is
the most commonly seen form, as mentioned before.

* succeeded:
[0191535bfda21f5dfec1c904775c5e2fbee8a985815c88d77258a0b42dba3526](https://btc.com/0191535bfda21f5dfec1c904775c5e2fbee8a985815c88d77258a0b42dba3526#in_0)
* penalised:
[0da5e5dba5e793d50820c2275dab74912b121c8b7e34ce32a9dbfd4567a9bf8e](https://btc.com/0da5e5dba5e793d50820c2275dab74912b121c8b7e34ce32a9dbfd4567a9bf8e#in_0)

Let us examine the successful case:

* Its upstream input is a 2-of-2 MULTISIG embedded in P2WSH,
the funding transaction.
* The funding transaction has two outputs (commitment transactions)
and the other is a P2WPKH.
The P2WPKH is not time locked and
has been sent to the counterparty of the channel.
* Its input witness starts with `<sig> 0` form
therefore its fund has already been successfully taken.
According to the LN paper, the LN channel is _fully closed_.

Why the second input has been penalised?
Because there is a `1` right after the signature and
it executes the if-branch which revokes all the funds in the channel.
Both inputs were from the two outputs of
the previous funding transaction and there is only one output.
Therefore we can conclude that it is in a penalty transaction.

##### sender script

Here are two examples:

* timeout:
[a16f6d78a58d31fe7459887adf5bd6b4dd95277ea375d250c700cde9fa908bdb](https://btc.com/a16f6d78a58d31fe7459887adf5bd6b4dd95277ea375d250c700cde9fa908bdb)
* claimed by preimage:
[89c744f0806a57a9b4634c320703cc941aaf272f290296373b709499064335e5](https://btc.com/89c744f0806a57a9b4634c320703cc941aaf272f290296373b709499064335e5)

The expenditures by timeout are easy to identify, as its format is simply
`0 <sig> <sig> 0`.
How to identify if a HTLC transaction has used the preimage?
Here I would like to recommend the
[python bitcoin library](https://github.com/petertodd/python-bitcoinlib).
I like it a lot because python provides power interactive terminal
and it is convenient to analyse bitcoin blockchain.

Let us see the transaction input whose hash starts with "89c744".
Its witness is in the form of `sig [unknown] scriptPubKey`.
Later we will find out that the unknown part is
actually the hash160 of a secret, the hash that locks the contract.
A revocation script will appear in the exactly same form
but we shall see that the script does not execute the revocation path.

The scriptPubKey part, according to BOLT \#3, is decoded as

```python
In [1]: from bitcoin.core import CScript

In [2]: CScript(bytearray.fromhex('76a9149d908ebcc9e1913b808eb7b4ba47bc4d1b35ebd38763ac672102aa52226cbb5aaef23f175575c06feb16aa303e76f288be28c4760ef768c865977c820120876475527c210362c9ec1c0c7bb399037469b376e9e19d6aa0fbb58df811786b7425dea94b519a52ae67a9146e3bef3f86aed6d6f825f13d1fa070039c866c5c88ac6868'))
Out[2]: CScript([OP_DUP, OP_HASH160, x('9d908ebcc9e1913b808eb7b4ba47bc4d1b35ebd3'), OP_EQUAL, OP_IF, OP_CHECKSIG, OP_ELSE, x('02aa52226cbb5aaef23f175575c06feb16aa303e76f288be28c4760ef768c86597'), OP_SWAP, OP_SIZE, x('20'), OP_EQUAL, OP_NOTIF, OP_DROP, 2, OP_SWAP, x('0362c9ec1c0c7bb399037469b376e9e19d6aa0fbb58df811786b7425dea94b519a'), 2, OP_CHECKMULTISIG, OP_ELSE, OP_HASH160, x('6e3bef3f86aed6d6f825f13d1fa070039c866c5c'), OP_EQUALVERIFY, OP_CHECKSIG, OP_ENDIF, OP_ENDIF])
```

Next, let us see the hash value of the unknown part,
```python

In [1]: import bitcoin

In [2]: CScript([bitcoin.core.Hash160(bytearray.fromhex('ae626cc4d6c208bdb3179b9d3efc7ae61779a9924b3852f01d0024afa84a4bbb'))])
Out[2]: CScript([x('6e3bef3f86aed6d6f825f13d1fa070039c866c5c')])
```

See? The hash160 of the previously unknown field is
equal to the value right after the last `OP_HASH160`
and before the `OP_EQUALVERIFY`.
Therefore the fund is indeed taken by providing the preimage,
according to the HTLC script for senders defined in BOLT \#3,
instead of being revoked.

Here I am not going to show how the script should be executed
step by step on a stack.
You may do this yourself and keep in mind that
the exection path in a if-else branch is determined by a digit
whose value is either 1 or 0.

##### receiver script

Similarly, here are another two examples for receiver script:

* claimed by preimage:
[36b1aff2ad0076be95b1ee1dc4036374998760c80c6583a6478a699e86658ac0](https://btc.com/36b1aff2ad0076be95b1ee1dc4036374998760c80c6583a6478a699e86658ac0)
* timeout:
[f9af9b93d66c7e5ee7dcbe0b53faa3d17aa6b9f4cc5b19f0985917b57d82c59a](https://btc.com/f9af9b93d66c7e5ee7dcbe0b53faa3d17aa6b9f4cc5b19f0985917b57d82c59a#in_0)

Again, if you find HTLC inputs that are penalised,
be sure to leave a comment.

### Summary

With all the resources presented by this article
and on-chain examples,
I hope you can better understand transactions in LN.

Long live blockchain.
