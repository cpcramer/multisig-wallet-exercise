﻿Testing... 

This walkthrough is based on [this project](./Multisig_wallet_info.md) by Nate Rush.


The solution is based on [this MultiSignature Wallet](https://github.com/ConsenSys/MultiSigWallet) found in the ConsenSys github repository.


# What is a Multisignature wallet? 


A multisignature wallet is an account that requires some m-of-n quorum of approved private keys to approve a transaction before it is executed. 


Ethereum implements multisignature wallets slightly differently than Bitcoin does. In Ethereum, multisignature wallets are implemented as a smart contract, that each of the approved external accounts sends a transaction to in order to "sign" a group transaction. 


Following this project spec designed by the UPenn Blockchain Club, you will now create your own multisignature wallet contract. 


**Note: It is not suggested that you use this multisignature wallet with any real funds, but rather use a far more deeply audited one such as the [Gnosis multisignature wallet.](https://wallet.gnosis.pm/)**


**Objectives:**
1. To learn how to handle complex interactions between multiple users on one contract
2. Learn how to avoid loops and implement voting
3. [To learn to assess possible attacks on contracts.](https://github.com/ConsenSys/smart-contract-best-practices)

Table of contents
=================
<!--ts-->
   * [Project Setup](#project-setup)
   * [Implementing the Contract](#implementing-the-contract)
      * [Constructor](#constructor)
      * [Submit Transaction](#submit-transaction)
      * [Add Transaction](#add-transaction)
      * [Confirm Transaction](#confirm-transaction)
      * [Execute Transaction](#execute-transaction)
      * [Additional Functions](#additional-functions)
   * [Interacting with the Contract](#interacting-with-the-contract)
   * [Further-Reading](#further-reading)
<!--te-->

Project Setup
============

Clone this GitHub repository. The [MultiSignatureWallet.sol](./contracts/MultiSignatureWallet.sol) file in the contracts directory has the structure of a multisignature wallet that you will be implementing.


Implementing the Contract
============

Let’s work on this contract in Remix to get more familiar with the Remix IDE. Navigate to [Remix](https://remix.ethereum.org/) in your browser.


In the upper left corner click the “+” icon to create a new file. Call it MultiSignatureWallet.sol.


Copy the contents of the MultiSignatureWallet.sol file found in the project directory into Remix.

![](./gifs/remix.gif)

Let’s review what this contract needs to be able to do before we start writing the code.


The contract will have multiple owners that will determine which transactions the contract is allowed to execute. Contract owners need to be able to propose transactions that other owners can either confirm or revoke. If a proposed transaction receives enough support, it will be executed.


Keeping these requirements in mind, let’s start going through the contract stub and start implementing this functionality. 


Remix is a browser based IDE that has code parsing built in, so it will show us any syntax or compilation errors directly in our environment. Notice the yellow triangles along the left side of the screen. Hovering over the triangle with your cursor will display the warning message. Yellow triangles are warning messages, whereas red circles are error messages and will prevent your contract from compiling.


Constructor
=====

Starting with the constructor, you can see that with the latest solidity compiler version, using the contract name as the constructor name has been deprecated, so let’s change it to constructor.

```javascript
    constructor(address[] memory _owners, uint _required)
```

We are going to want to check the user inputs to the constructor to make sure that a user does not require more confirmations than there are owners, that the contract requires at least one confirmation before sending a transaction and that the owner array contains at least one address.


We can create a modifier that checks these conditions

```javascript
    modifier validRequirement(uint ownerCount, uint _required) {
        if (_required > ownerCount || _required == 0 || ownerCount == 0)
            revert();
        _;
    }  
```



And call it when the constructor runs.

```javascript
    constructor(address[] memory _owners, uint _required) public 
            validRequirement(_owners.length, _required)
        {...}
```

We are going to want to keep the `_owners` and `_required` values for the life of the contract, so we need to declare the variables in storage in order to save them.

```javascript
    address[] public owners;
    uint public required;
    mapping (address => bool) public isOwner;
```

We also added a mapping of owner addresses to booleans so that we can quickly reference (without having to loop over the owners array) whether a specific address is an owner or not.


All of these variables will be set in the constructor.

```javascript
    constructor(address[] memory _owners, uint _required) public 
        validRequirement(_owners.length, _required)
    {
        for (uint i=0; i<_owners.length; i++) {
            isOwner[_owners[i]] = true;
        }
        owners = _owners;
        required = _required;
    } 
```

Submit Transaction
=====

The `submitTransaction` function allows an owner to submit and confirm a transaction.


First we need to restrict this function to only be callable by an owner.

```javascript
    require(isOwner[msg.sender]);
```

Looking at the rest of the contract stub, you will notice that there are two other functions in the contract that can help you implement this function, one is called `addTransaction` that takes the same inputs as `submitTransaction` and returns a `uint transactionId`. The other is called `confirmTransaction` that takes a `uint transactionId`.


We can easily implement `submitTransaction` with the help of these other functions:

```javascript
    function submitTransaction(address destination, uint value, bytes memory data) 
        public 
        returns (uint transactionId) 
    {
        require(isOwner[msg.sender]);
        transactionId = addTransaction(destination, value, data);
        confirmTransaction(transactionId);
    }
```

Add Transaction
=====

Let’s jump to the `addTransaction` function and implement that. This function adds a new transaction to the transaction mapping (which we are about to create), if the transaction does not exist yet.

```javascript
    function addTransaction(address destination, uint value, bytes memory data) internal returns (uint transactionId);
```

A transaction is a data structure that is defined in the contract stub.

```javascript
    struct Transaction {
        address destination;
        uint value;
        bytes data;
        bool executed;
    }
```

We need to store the inputs to the `addTransaction` function in a Transaction struct and create a transaction id for the transaction. Let’s create two more storage variables to keep track of the transaction ids and transaction mapping.

```javascript
    uint public transactionCount;
    mapping (uint => Transaction) public transactions;
```

In the `addTransaction` function we can get the transaction count, store the transaction in the mapping and increment the count. This function modifies the state so it is a good practice to emit an event. 


We will emit a `Submission` event that takes a `transactionId`. Let’s define the event first. Events are usually defined at the top of a Solidity contract, so that is what we will do. Add this line just below the contract declaration.

```javascript
    event Submission(uint indexed transactionId);
```

The ***indexed*** keyword in the event declaration makes the event easily searchable and is useful when building user interfaces that need to parse lots of events.


In the function body we can call the event.

```javascript
    function addTransaction(address destination, uint value, bytes memory data)
        internal
        returns (uint transactionId)
    {
        transactionId = transactionCount;
        transactions[transactionId] = Transaction({
            destination: destination,
            value: value,
            data: data,
            executed: false
        });
        transactionCount += 1;
        emit Submission(transactionId);
    }
```

The `uint transactionId` is returned for the `submitTransaction` function to hand over to the `confirmTransaction` function.


Confirm Transaction
=====

```javascript
    function confirmTransaction(uint transactionId) public {}
```

The confirm transaction function allows an owner to confirm an added transaction.


This requires another storage variable, a confirmations mapping that stores a mapping of boolean values at owner addresses. This variable keeps track of which owner addresses have confirmed which transactions.

```javascript
    mapping (uint => mapping (address => bool)) public confirmations;
```

There are several checks that we will want to verify before we execute this transaction. First, only wallet owners should be able to call this function. Second, we will want to verify that a transaction exists at the specified `transactionId`. Last, we want to verify that the `msg.sender` has not already confirmed this transaction.

```javascript
    require(isOwner[msg.sender]);
    require(transactions[transactionId].destination != address(0));
    require(confirmations[transactionId][msg.sender] == false);
```

Once the transaction receives the required number of confirmations, the transaction should execute, so once the appropriate boolean is set to true

```javascript
    confirmations[transactionId][msg.sender] = true;
```

Since we are modifying the state in this function, it is a good practice to log an event. First, we define the event called `Confirmation` that logs the confirmers address as well as the `transactionId` of the transaction that they are confirming.

```javascript
    event Confirmation(address indexed sender, uint indexed transactionId);
```

Both of these event parameters are indexed to make the event more easily searchable. Now we can call the event in the function.


After logging the event we can attempt to execute the transaction

```javascript
    executeTransaction(transactionId);
```

So the entire function should look like this:

```javascript
    function confirmTransaction(uint transactionId)
        public
    {
        require(isOwner[msg.sender]);
        require(transactions[transactionId].destination != address(0));
        require(confirmations[transactionId][msg.sender] == false);
        confirmations[transactionId][msg.sender] = true;
        emit Confirmation(msg.sender, transactionId);
        executeTransaction(transactionId);
    }
```

Execute Transaction
=====

The execute transaction function takes a single parameter, the `transactionId`.


First, we want to make sure that the Transaction at the specified id has not already been executed.

```javascript
    require(transactions[trandsactionId].executed == false);
```

Then we want to verify that the transaction has at least the required number of confirmations.


To do this we will loop over the owners array and count how many of the owners have confirmed the transaction. If the count reaches the required amount, we can stop counting (save gas) and just say the the requirement has been reached.


I define the helper function isConfirmed, which we can call from the `executeTransaction` function.

```javascript
    function isConfirmed(uint transactionId)
        public
        view
        returns (bool)
    {
        uint count = 0;
        for (uint i=0; i<owners.length; i++) {
            if (confirmations[transactionId][owners[i]])
                count += 1;
            if (count == required)
                return true;
        }
    }
```

It will return true if the transaction is confirmed, so in the `executeTransaction` function, we can execute the transaction at the specified if it is confirmed, otherwise do not execute it and then update the Transaction struct to reflect the state.


We are updating the state, so we should log an event reflecting the change.

```javascript
    event Execution(uint indexed transactionId);
    event ExecutionFailure(uint indexed transactionId);
```

We have two possible outcomes of this function -- the transaction is not guaranteed to successfully execute. We will want to catalog whether the send transaction successfully executes or fails. Notice how the function body handles a failed transaction and tracks it in the contract state.

```javascript
    function executeTransaction(uint transactionId)
        public
    {
        require(transactions[transactionId].executed == false);
        if (isConfirmed(transactionId)) {
            Transaction storage t = transactions[transactionId];
            t.executed = true;
            (bool success, bytes memory rdata) = t.destination.call.value(t.value)(t.data);
            if (success)
                emit Execution(transactionId);
            else {
                emit ExecutionFailure(transactionId);
                t.executed = false;
            }
        }
    }
```

Additional Functions
=====

So far, we have only covered the core functionality of this MultiSignature Wallet found in the ConsenSys github repository.


I will leave it to you to continue the exercise and explore the rest of the contract. The code is well commented and you should be able to determine and explain the purpose of each function in the contract.


If you would like a further challenge, continue on to the bottom of the Solidity file and investigate the [MultiSigWalletWithDailyLimit contract](https://github.com/ConsenSys/MultiSigWallet/blob/master/MultiSigWalletWithDailyLimit.sol).


Interacting with the Contract
============

Now that we have a basic MultiSignature Wallet, let’s interact with the Multisig Wallet and see how it works.


Copy the contract that we developed in Remix into the truffle project directory provided. 


You can see that the project directory comes with a SimpleStorage.sol contract. This is the contract that we are going to be calling from the Multisig contract.


If you look in the migrations directory you will see the deployment script that truffle will use to deploy the SimpleStorage contract as well as the MultiSig Wallet.


The truffle deployer allows us to access accounts, which is useful given that the MultiSig contract constructor requires an array of owner addresses as as well as the number of required confirmations to execute a transaction.


The owners array and the deployment scripts are already in the file.

```javascript
    const owners = [accounts[0], accounts[1]]
    deployer.deploy(MultiSig, owners, 2)
```

We are only going to require 2 confirmations for the sake of simplicity.


To deploy the contracts, start the development environment by running `truffle develop` in a terminal window at the project directory. The truffle command line will appear

```
truffle(develop)> 
```

Deploy the contracts
```
truffle(develop)> migrate 
```
If `migrate` does not work, try `migrate --reset`.

And then get the deployed instances of the SimpleStorage.sol and MultiSignatureWallet.sol contracts.

```
truffle(develop)> var ss = await SimpleStorage.at(SimpleStorage.address)
truffle(develop)> var ms = await MultiSignatureWallet.at(MultiSignatureWallet.address)
```

Check the state of the the SimpleStorage contract

```
truffle(develop)> ss.storedData.call()
<BN: 0>
```

This means that it is 0. You can verify by waiting for the promise to resolve and converting the answer to a string. Try it with:

```
ss.storedData.call().then(res => { console.log( res.toString(10) )} )
0
```

Let’s submit a transaction to update the state of the SimpleStorage contract to the MultiSig contract. `SumbitTransaction` takes the address of the destination contract, the value to send with the transaction and the transaction data, which includes the encoded function signature and input parameters.


If we want to update the SimpleStorage contract data to be 5, the encoded function signature and input parameters would look like this:

```javascript
var encoded = '0x60fe47b10000000000000000000000000000000000000000000000000000000000000005'
```

Let's get the available accounts and then make a call to the MultiSig contract:

```
truffle(develop)> var accounts = await web3.eth.getAccounts()
truffle(develop)> ms.submitTransaction(ss.address, 0, encoded, {from: accounts[0]})
```

And we see the transaction information printed in the terminal window. In the logs, we can see that a “Submission” event was fired, as well as a “Confirmation” event, which is what we expect. 


The current state of the MultiSig has one transaction that has not been executed and has one confirmation (from the address that submitted it). One more confirmation should cause the transaction to execute. Let’s use the second account to confirm it. The `confirmTransaction` function takes one input, the index of the Transaction to confirm.

```
truffle(develop)> ms.confirmTransaction(0, {from: accounts[1]})
```

The transaction information is printed in the terminal. You should see two log events this time as well. A “Confirmation” event as well as an “Execution” event. This indicates that the call to SimpleStorage executed successfully. If it didn’t execute successfully, we would see an “ExecutionFailure” event there instead.


We can verify that the state of the contract was updated by running

```
truffle(develop)> ss.storedData.call()
<BN: 5>
```

The `storedData` is now 5. And we can check that the address that updated the SimpleStorage contract was the MultiSig Wallet.

```
truffle(develop)> ss.caller.call()
'0x855d1c79ad3fb086d516554dc7187e3fdfc1c79a'
truffle(develop)> ms.address
'0x855d1c79ad3fb086d516554dc7187e3fdfc1c79a'
```

The two addresses are the same!

Further Reading
============
- [ConsenSys MultiSig Repo](https://medium.com/hellogold/ethereum-multi-signature-wallets-77ab926ab63b)
- [List of MultiSig Contracts on Mainnet](https://medium.com/talo-protocol/list-of-multisig-wallet-smart-contracts-on-ethereum-3824d528b95e)
- [What is a MultiSig Wallet?](https://medium.com/hellogold/ethereum-multi-signature-wallets-77ab926ab63b)
- [Unchained Capital's Multi-Sig](https://blog.unchained-capital.com/a-simple-safe-multisig-ethereum-smart-contract-for-hardware-wallets-a107bd90bb52)
- [Grid+: Toward an Ethereum MultiSig Standard](https://blog.gridplus.io/toward-an-ethereum-multisig-standard-c566c7b7a3f6)
