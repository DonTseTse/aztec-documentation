# Aztec protocol (v1) documentation

> Work in progress, protocol authors contacted with questions around use of non-mintable zk assets, documentation will be updated once answered. 

The Aztec protocol is a powerful zero-knowledge-based secret transaction solution for Ethereum. Its developers provide a lot of material but there's no good guide on how to use it and many of the examples are rather confusing for newcomers. 

## Introduction
Aztec works using *notes*, where each note is a zero-knowledge (zk) representation of a certain amount of value. Notes have a lot in common with Bitcoin's unspent outputs: each note has an owner (an address); a note can only be spent by the owner or parties it approved and a note can only be spent completely.

Every Aztec deployment consists of:
- one *Aztec cryptographic engine (ACE)* smart-contract including additional contracts for proof validation
- one or more *zk asset* contracts relying on the ACE

The ACE provides and maintains a *note registry* for every linked zk asset.

A zk asset has 2 essential and mutually exclusive properties, it may be
- *convertible*: it's linked to an ERC20 and note value corresponds to ERC20 value (with a scaling factor)
- *adjustable*: value is handled internally through note minting and burning

Zk value transfer is achieved using so-called *join-split*s which describes the fact that the protocol is able to handle operations involving multiple input notes ("join") and multiple output notes ("split"). While the zk proof signer must obviously either own or have the spending permission on all input notes, output notes can have any owner and the protocol guarantees that the total value on both sides is balanced. A nice feature is that for convertible zk assets, join-splits may be combined with ERC20 transactions to balance inputs and outputs. It's hence possible f.ex. to do a transaction which splits a note of 0 value into notes with value and to "pay it" with ERC20s. In such a scenario, the ACE escrows the ERC20s of all outstanding note values. 

Beside the join-split, there are 6 additional operations: mint, burn, swap, dividend, private range and public range. While mint and burn will obviously be used in the case of adjustable zk assets, all others represent special operation types which are not covered in this documentation. 

The next sections show the details for a convertible zk and an adjustable zk asset, with smart-contract deployment details and the use on client side. 

## Convertible zero-knowledge asset
### Smart-contract deployment
See [smart-contract-deployments.md > convertible zk asset](./smart-contract-deployments.md)

### Transfering ERC20 value to a Aztec note
Transfer of 10 ERC20s into an Aztec note (supposing a zk asset scaling factor of 1). The 10 ERC20s must be owned by the account of course, otherwise the `approve()` fails. 
```js
const secp256k1 = require("@aztec/secp256k1");
const aztec = require("aztec.js");
// erc20, ace and zkAsset are the @truffle/contract instances of the smart-contracts

let myAccount = secp256k1.accountFromPrivateKey(MY_PRIVATE_KEY);    
// myAccount: other ways to handle wallet internals may be used - signature construction expects an object with the attributes privateKey, publicKey and address
let inputNote = await aztec.note.createZeroValueNote();
let outputNote = await aztec.note.create(myAccount.publicKey, 10);
// the 3rd and 4th parameters of JoinSplitProof() are the ERC20 token owner and the amount (< 0 means tokens go from myAccount.address to the ACE)
let proof = new aztec.JoinSplitProof([inputNote], [outputNote], myAccount.address, -10, myAccount.address);
let proofEncoding = proof.encodeABI(zkAsset.address);
let proofSignatures = proof.constructSignatures(zkAsset.address, [myAccount]);
// ERC20: allow the ACE to use 10 of myAccount's tokens (fails if myAccount's balance is not >= 10)
await erc20.contract.methods.approve(ace.address, 10).send({from: myAccount.address});
// ACE: confirm that this transaction is allowed to retrieve 10 ERC20s from the linked contract
await ace.contract.methods.publicApprove(zkAsset.address, proof.hash, 10).send({from: myAccount.address});
// Do the magic - expensive in gas
await zkAsset.contract.methods.confidentialTransfer(proofEncoding, proofSignatures).send({from: myAccount.address, gasLimit: 250000});
```
The `outputNote` may then be used in subsequent join-split transactions. The easiest way is to call `outputNote.getView()` to retrieve the note's view key and to store it. Once the note shall be used, it may be reconstructed using `aztec.note.fromViewKey()`. 

### "Paying" a secret transfer with ERC20
Let's split a value of 20 ERC20s into two Aztec notes of 10 for recipientA and recipientB, of which we need to know the public keys. The zk asset scaling factor is 1. 
```js
// same imports than in the 1st example for aztec and secp256k1
// erc20, ace and zkAsset are the @truffle/contract instances of the smart-contracts

let myAccount = secp256k1.accountFromPrivateKey(MY_PRIVATE_KEY);    
// myAccount: other ways to handle wallet internals may be used - signature construction expects an object with the attributes privateKey, publicKey and address
let inputNote = await aztec.note.createZeroValueNote();
let noteForA = await aztec.note.create(recipientA.publicKey, 10);
let noteForB = await aztec.note.create(recipientB.publicKey, 10);
// the 3rd and 4th parameters of JoinSplitProof() are the ERC20 token owner and the amount (< 0 means tokens go from myAccount.address to the ACE)
let proof = new aztec.JoinSplitProof([inputNote], [noteForA, noteForB], myAccount.address, -20, myAccount.address);
let proofEncoding = proof.encodeABI(zkAsset.address);
let proofSignatures = proof.constructSignatures(zkAsset.address, [myAccount]);
// ERC20, ACE, zkAsset calls: see comments in 1st example
await erc20.contract.methods.approve(ace.address, 20).send({from: myAccount.address});
await ace.contract.methods.publicApprove(zkAsset.address, proof.hash, 20).send({from: myAccount.address});
await zkAsset.contract.methods.confidentialTransfer(proofEncoding, proofSignatures).send({from: myAccount.address, gasLimit: 250000});
```
Recipient A and B may then spend or redeem their notes. The easiest is to provide them the view key (retrieved using `noteFor{A,B}.getView()`) which has to be transmitted over a secure channel (f.ex. a properly encrypted web-service). 

### Recovering ERC20 value from a Aztec note
Let's imagine the receipient was sent two notes of value 10 each and wants to withdraw the 20 ERC20s (zk asset scaling factor = 1). The recipient was provided the view keys of the notes as explained in the previous examples. 
```js
// same imports than in the 1st example for aztec and secp256k1
// zkAsset is the @truffle/contract instance of the smart-contract

let myAccount = secp256k1.accountFromPrivateKey(MY_PRIVATE_KEY);    
// myAccount: other ways to handle wallet internals may be used - signature construction expects an object with the attributes privateKey, publicKey and address
let inputNote1 = await aztec.note.fromViewKey(note1ViewKey);
let inputNote2 = await aztec.note.fromViewKey(note2ViewKey);
let outputNote = await aztec.note.createZeroValueNote();
// the 3rd and 4th parameters of JoinSplitProof() are the ERC20 token owner and the amount (> 0 means tokens go from the ACE to myAccount.address)
let proof = new aztec.JoinSplitProof([inputNote1, inputNote2], [outputNote], myAccount.address, 20, myAccount.address);
let proofEncoding = proof.encodeABI(zkAsset.address);
let proofSignatures = proof.constructSignatures(zkAsset.address, [myAccount]);
await zkAsset.contract.methods.confidentialTransfer(proofEncoding, proofSignatures).send({from: myAccount.address, gasLimit: 250000});
```
Once executed, myAccount's ERC20 balance has increased by 20. 

## Adjustable zk asset
### Smart-contract deployment
See [smart-contract-deployments.md > adjustable zk asset](./smart-contract-deployments.md)

### Mint a note

### Transfer value
Let's imagine a user owns a note of value 20 (minted or received) of which it has the view key and wants to send two notes of value 10 each to recipient A and recipient B, of which it knows the public keys.
```js
// same imports than in the 1st example for aztec and secp256k1
// zkAsset is the @truffle/contract instance of the smart-contract

let myAccount = secp256k1.accountFromPrivateKey(MY_PRIVATE_KEY);    
// myAccount: other ways to handle wallet internals may be used - signature construction expects an object with the attributes privateKey, publicKey and address
let inputNote1 = await aztec.note.fromViewKey(noteViewKey);
let noteForA = await aztec.note.create(recipientA.publicKey, 10);
let noteForB = await aztec.note.create(recipientB.publicKey, 10);
// 3rd and 4th parameter set to defaults because there's no public value involved
let proof = new aztec.JoinSplitProof([inputNote], [noteForA, noteForB], 0x0, 0, myAccount.address);
let proofEncoding = proof.encodeABI(zkAsset.address);
let proofSignatures = proof.constructSignatures(zkAsset.address, [myAccount]);
await zkAsset.contract.methods.confidentialTransfer(proofEncoding, proofSignatures).send({from: myAccount.address, gasLimit: 250000});
```

### Burn a note

ToDO: check out factory, check out asset type calc



several example repositories, a SDK and there are medium articles to explain the features of the protocol, but there's nevertheless a big confusion for newcomers about how to use it because the example only cover very small parts of the usage of the protocol.