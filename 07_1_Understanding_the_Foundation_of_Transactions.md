# 7.1: Understanding the Foundation of Transactions

> :information_source: **NOTE:** This is a draft in progress, so that I can get some feedback from early reviewers. It is not yet ready for learning.

The foundation of Bitcoin is the ability to protect transactions, something that's done with a simple scripting language.

## Know the Parts of the Cryptographic Puzzle

As described in [Chapter 1](01_0_Introducing_Bitcoin.md), the funds in each Bitcoin transaction are locked with a cryptographic puzzle. To be precise, we said that Bitcoin is made up of "a sequence of atomic transactions: each of which is enabled by the sender with the solution to a previous cryptographic puzzle that was stored as a script; each of which is locked for the recipient with a new cryptographic puzzle that is stored as a script". Those scripts, which lock and unlock transactions, are written in Bitcoin Script.

_What is Bitcoin Script?_ Bitcoin Script is a stack-based Forth-like language that purposefully avoids loops and so is not Turing-complete. It's made up of individual opcodes. Every single transaction in Bitcoin is locked with a Bitcoin Script; when the locking transaction for a UTXO is run with the correct inputs, that UTXO can then be spent.

The fact that transactions are locked with scripts means that they can be locked in a variety of different ways, requiring a variety of different keys. In fact, we've met a number of different locking mechanisms to date, each of which used different opcodes:

   * OP_CHECKSIG, which checks a public key against a signature, is the basis of a P2PKH address, as will be fully detailed in [§7.3: Scripting a P2PKH](07_3_Scripting_a_P2PKH.md).
   * OP_CHECKMULTISIG similarly checks multisigs, as will be fully detailed in [§8.4: Scripting a Multisig](08_4_Scripting_a_Multisig.md).
   * OP_CHECKLOCKTIMEVERIFY and OP_SEQUENCEVERIFY form the basis of more complex Timelocks, as will be fully detailed in [§9.2: Using CLTV in Scripts](09_2_Using_CLTV_in_Scripts) and [§9.3: Using CSV in Scripts](09_3_Using_CSV_in_Scripts.md).
   * OP_RETURN is the mark of an unspendable transaction, which is why it's used to carry data, as was alluded to in [§6.5: Sending a Transaction with Data](06_5_Sending_a_Transaction_with_Data.md).

## Access Scripts In Your Transactions

You may not realize it, but you've already seen these locking and unlocking scripts as part of the raw transactins you've been working with. The best way to look into these scripts in more depth is thus to create a raw transaction, then examine it.

### Create a Test Transaction

To examine real unlocking and locking scripts, create a quick raw transaction by grabbing the first unspent transaction sitting around, and resending it to a change address, minus a transaction fee:
```
$ utxo_txid=$(bitcoin-cli listunspent | jq -r '.[0] | .txid') 
$ utxo_vout=$(bitcoin-cli listunspent | jq -r '.[0] | .vout')
$ recipient=$(bitcoin-cli getrawchangeaddress)
$ rawtxhex=$(bitcoin-cli -named createrawtransaction inputs='''[ { "txid": "'$utxo_txid'", "vout": '$utxo_vout' } ]''' outputs='''{ "'$recipient'": 1.2985 }''')
$ signedtx=$(bitcoin-cli -named signrawtransaction hexstring=$rawtxhex | jq -r '.hex')
```
You don't actually need to send it: the goal is simply to produce a complete transaction that you can examine.

### Examine Your Test Transaction

You can now examine your transaction in depth by using `decoderawtransaction` on the `$signedtx`:
```
$ bitcoin-cli -named decoderawtransaction hexstring=$signedtx
{
  "txid": "7b5f161ff9f9275c94dca38348c91056fe6829a2b768ed82ede4bcf7c7384a68",
  "hash": "7b5f161ff9f9275c94dca38348c91056fe6829a2b768ed82ede4bcf7c7384a68",
  "size": 192,
  "vsize": 192,
  "version": 2,
  "locktime": 0,
  "vin": [
    {
      "txid": "99d2b5717fed8875a1ed3b2827dd60ae3089f9caa7c7c23d47635f6f5b397c04",
      "vout": 0,
      "scriptSig": {
        "asm": "3045022100c4ef5b531061a184404e84ab46beee94e51e8ae15ce98d2f3e10ae7774772ffd02203c546c399c4dc1d6eea692f73bb3fff490ea2e98fe300ac6a11840c7d52b6166[ALL] 0319cd3f2485e3d47552617b03c693b7f92916ac374644e22b07420c8812501cfb",
        "hex": "483045022100c4ef5b531061a184404e84ab46beee94e51e8ae15ce98d2f3e10ae7774772ffd02203c546c399c4dc1d6eea692f73bb3fff490ea2e98fe300ac6a11840c7d52b616601210319cd3f2485e3d47552617b03c693b7f92916ac374644e22b07420c8812501cfb"
      },
      "sequence": 4294967295
    }
  ],
  "vout": [
    {
      "value": 1.29850000,
      "n": 0,
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 371c20fb2e9899338ce5e99908e64fd30b789313 OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a914371c20fb2e9899338ce5e99908e64fd30b78931388ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "mkYMA2i2vzEXjoDn4GfvPoFr3MJ62uUtMx"
        ]
      }
    }
  ]
}
```
The two scripts are found in the two different parts of the transaction.

The `scriptSig` is located in the `vin`. This is the _unlocking_ script. It's what's run to access the UTXO being used to fund this transaction. There will be one `scriptSig` per UTXO in a transaction.

The `scriptPubKey` is located in the `vout`. This is the _locking_ script. It's what locks the new output from the transaction. There will be one `scriptPubKey` per output in a transaction.

_How do the scriptSig and scriptPubKey interact?_ The `scriptSig` of a transaction unlocks the previous UTXO; this new transaction's output will then be locked with a `scriptPubKey`, which can in turn be unlocked by the `scriptSig` of the transaction that reuses that UTXO.

### Read The Scripts in Your Transaction

Look at the two scripts and you'll see that each includes two different representations: the `hex` is what actually gets stored, but the more readable assembly language (`asm`) can sort of show you what's going on.

Take a look at the `asm` of the unlocking script and you'll get your first look at what Bitcoin Scripting looks like:
```
"3045022100c4ef5b531061a184404e84ab46beee94e51e8ae15ce98d2f3e10ae7774772ffd02203c546c399c4dc1d6eea692f73bb3fff490ea2e98fe300ac6a11840c7d52b6166[ALL] 0319cd3f2485e3d47552617b03c693b7f92916ac374644e22b07420c8812501cfb"
```
As it happens, that mess of numbers is a private-key signature followed by the associated public key. Or at least that's hopefully what it is, because that's what's required to unlock the P2PKH UTXO that this transaction is using.

Read the locking script and you'll see it's a lot more obvious:
```
OP_DUP OP_HASH160 371c20fb2e9899338ce5e99908e64fd30b789313 OP_EQUALVERIFY OP_CHECKSIG
```
That is the standard method in Bitcoin Script for locking a P2PKH transaction.

[§7.3](07_3_Scripting_a_P2PKH.md) will explain how these two scripts go together, but first you will need to know how Bitcoin Scripts are evaluated.

## Examine a Different Sort of Transaction

Before we leave this foundation behind, we're going to look at a different type of locking script. Here's the `scriptPubKey` from the multisig transaction that you created in [§6.1: Sending a Transaction with a Multisig](06_1_Sending_a_Transaction_to_a_Multisig.md).
```
      "scriptPubKey": {
        "asm": "OP_HASH160 babf9063cee8ab6e9334f95f6d4e9148d0e551c2 OP_EQUAL",
        "hex": "a914babf9063cee8ab6e9334f95f6d4e9148d0e551c287",
        "reqSigs": 1,
        "type": "scripthash",
        "addresses": [
          "2NAGfA4nW6nrZkD5je8tSiAcYB9xL2xYMCz"
        ]
      }
```

Compare that to the `scriptPubKey` from your new P2PKH transaction:
```
      "scriptPubKey": {
        "asm": "OP_DUP OP_HASH160 371c20fb2e9899338ce5e99908e64fd30b789313 OP_EQUALVERIFY OP_CHECKSIG",
        "hex": "76a914371c20fb2e9899338ce5e99908e64fd30b78931388ac",
        "reqSigs": 1,
        "type": "pubkeyhash",
        "addresses": [
          "mkYMA2i2vzEXjoDn4GfvPoFr3MJ62uUtMx"
        ]
      }
```

These two transactions are _definitely_ locked in different ways. Bitcoin recognises the first as `scripthash` (P2SH) and the second as `pubkeyhash` (P2PKH), but you should also be able to see the difference in the different `asm` code: `OP_HASH160 babf9063cee8ab6e9334f95f6d4e9148d0e551c2 OP_EQUAL` versus `OP_DUP OP_HASH160 371c20fb2e9899338ce5e99908e64fd30b789313 OP_EQUALVERIFY OP_CHECKSIG`. This is the power of scripting: it can very simply produce some of the dramatically different sorts of transactions that you learned about in the previous chapters.

## Summary: Understanding the Foundation of Transactions

Every Bitcoin transaction includes at least one unlocking script (`scriptSig`), which solves a previous cryptographic puzzle, and at least one locking script (`scriptPubKey`), which creates a new cryptographic puzzle. There's one `scriptSig` per input and one `scriptPubKey` per output. Each of these scripts is written in Bitcoin Script, a Forth-like language that further empowers Bitcoin.

_What is the power of scripts?_ Scripts unlock the full power of Smart Contracts. With the appropriate opcodes, you can make very precise decisions about who can redeem funds, when they can redeem funds, and how they can redeem funds. More intricate rules for corporate spending, partnership spending, proxy spending, and other methodologies can also be encoded within a Script. It even empowers more complex Bitcoin services such as Lightning and sidechains.

## What's Next?

Continue "Introducing Bitcoin Scripts" with [§7.2: Running a Bitcoin Script](07_2_Running_a_Bitcoin_Script.md).
