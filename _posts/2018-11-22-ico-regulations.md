---
layout: post
title: 'ICO Regulations — Securities Depository on a Blockchain'
author: jordan
categories: [blockchain, finance]
image: /assets/images/securities.png
featured: true
hidden: false
---

While the world is turning towards ICOs, governments struggle to come up with regulations.

This post outlines a proof of concept implementation of a ICO / securities depository base contract that could be instantiated and owned by any government in order to lay down fundamental ICO regulations in the blockchain contracts.

This base contract essentially becomes a securities depository on a blockchain, **where every security is a token**. We now have a solution which is capable of operating both as securities depository for ICOs and all of the previous types of securities (ownership rights, banknotes, bonds, swaps etc.).

Quick recap on what is a securities depository:

>A central securities depository (CSD) is a specialist financial organization holding securities such as shares either in certificated or uncertificated (dematerialized) form so that ownership can be easily transferred through a book entry rather than the transfer of physical certificates. (Wikipedia)

## Motivation
* A possible way to have regulated ICOs with minimal overhead
* Regulations are up to date in smart contracts
* Availability to hold any type of securities (stocks, options..) in blockchain
* Have a common blockchain interface and development platform for securities depository (instead of APIs that for some depositories is legacy)
* Distributed infrastructure, openness and transparency
* Independence — after a blockchain identity is approved by the regulator, it can operate independently.

## Let’s get started
Using Solidity and Truffle test suite.

**Full working code is available at [Github](https://github.com/Producement/SecuritiesDepositoryBlockchain)**

Outlined code is not optimised for performance or security.

## Domain
We will start with an issuer.
```
struct Issuer {
    bytes32 registryCode;
    address identifier;
    bytes32 name;
    bytes32 countryCode;
}
```
Where identifier is the key which identifies the issuer on the blockchain. We will later use this to make sure only issuer of a given security can emit them.

Let’s continue with securities. For this we need to pick a set of security types, which we can later use to build business logic (out of the scope of this POC).

```
enum SecurityType { DebtInstrument, FundUnit, Rights, Unit }
```

Now we define what a security looks like

```
struct Security {
    bytes32 isin;
    Issuer issuer;
    SecurityType securityType;
    uint nominalValue;
}
```

Next, let’s take a look at the beneficiary

```
struct Shareholder {
    bytes32 registryCode;
    address identifier;
}
```

As you can see beneficiary at the moment has a registry code attached to it. This could be company number, social number, id code etc. An important question would be if we want the real world identity of the beneficiary to be visible on the blockchain? Luckily we can implement it both ways — and store the real world identity off the chain if needed.

Now we need a way to record movements of securities within a given domain. Let’s use a standard [double-entry bookkeeping](https://en.wikipedia.org/wiki/Double-entry_bookkeeping_system) with credit and debit entries. We’ll do it with Transaction entity.

```
enum TransactionType { Debit, Credit }
```
```
struct Transaction {
    Security security;
    Shareholder shareholder;
    uint amount;
    TransactionType transactionType;
}
```

Voilà — simplified version of a securities / ICO depository domain is ready.

Each of those entities have CRUD functionality to be persisted in the blockchain.

## Calculating a balance
To calculate a balance for a particular shareholder and a security. We’ll go over all of the credit and debit transactions to end up with a final balance. In a real world use case this algorithm would likely look different as optimised for performance.

```
for (uint i = 0; i < transactions.length; i ++) {
    var transaction = transactions[i];

    if (isTransactionFor(transaction,
        shareholderRegistryCode, securityIsin)) {
        if (transaction.transactionType == TransactionType.Credit) {
            balance += transaction.amount;
        } else if (transaction.transactionType ==
          TransactionType.Debit) {
            balance -= transaction.amount;
        }
    }
}
```

## Emitting and transferring securities
**Before emitting the securities / ICO tokens, the smart contract then could run through a set of business logic, which defines if current emission corresponds to regulations and is approved.**

To emit a security, we need to make sure that the given function caller is the Issuer of a given security and then credit the shareholder.

```
function emit(bytes32 securityIsin, bytes32 shareholderRegistryCode, uint amount) {
    var security = securities[securityIsin];
    var shareholder = shareholders[shareholderRegistryCode];

    require(isValidEmission(security, shareholder));

    credit(security, shareholder, amount);
}

function isValidEmission(Security security, Shareholder shareholder) internal returns (bool) {
    // regulations and emission rules go here
    // eg. is valid shareholder PEP, AML, .. checks
    // eg. is valid security nominal values
    // ...

    require(msg.sender == security.issuer.identifier);

    return true;
}
```

**The same applies: The blockchain contract before transfer, can make sure that the transfer is legal and approved.**

To transfer a security from one shareholder to another we simply debit the sender and credit the receiver. We also need to check that the sender has enough funds.

```
function transfer(
    bytes32 senderRegistryCode, bytes32 receiverRegistryCode,
    bytes32 securityIsin, uint amount) {
    var security = securities[securityIsin];
    var sender = shareholders[senderRegistryCode];
    var receiver = shareholders[receiverRegistryCode];

    require(isValidTransfer(sender, receiver, security, amount));

    debit(security, sender, amount);
    credit(security, receiver, amount);
}

function isValidTransfer(
    Shareholder sender, Shareholder receiver, Security security, uint amount) internal returns (bool) {
    // regulations and transfer rules go here
    // eg. is valid sender and receiver, amounts PEP, AML, .. checks
    // eg. is valid security
    // ...

    uint senderBalance =
        getBalance(sender.registryCode, security.isin);
    require(senderBalance >= amount);
    require(sender.identifier == msg.sender);

    return true;
}
```

## Summary
Thanks for thinking along, thoughts and feedback is welcome!

Full working code is available at [Github](https://github.com/Producement/SecuritiesDepositoryBlockchain).

![A few transactions while running the test suite.](/assets/images/securities2.png)

_(This article was originally posted to Jordans personal [blog](https://medium.com/@JordanValdma/ico-regulations-securities-depository-on-a-blockchain-26a65d54495).)_