# Poly Network Hack Analysis

## Overview of the protocal
Poly Network is a **cross-chain interoperability protocol** that enables the transfer of assets and data across different blockchain networks. It connects blockchains like Ethereum, Binance Smart Chain, Polygon, and others, making it possible for users and applications to interact seamlessly between ecosystems.


### Key Features of Poly Network
1. **Cross-Chain Compatibility**:
   - Supports major blockchains like Ethereum, Binance Smart Chain, Polygon, and more.
   - Provides seamless asset transfers across diverse ecosystems.

2. **Lock and Release Mechanism**:
   - Assets are locked on the source chain and equivalent tokens are minted or released on the destination chain.
   - Ensures the total supply of tokens remains consistent across chains.

3. **Merkle Tree Authentication**:
   - Uses Merkle Trees to securely track and authenticate cross-chain messages.
   - Ensures data integrity and tamper-proof messaging between chains.

4. **Modular Architecture**:
   - Composed of `Home Contracts` and `LockProxy` contracts that manage asset transfers and cross-chain communication efficiently.

5. **Wide Adoption**:
   - Integrated with numerous DeFi projects and widely used for bridging liquidity in decentralized applications.

## Technical Workflow of Poly Network
Poly Network facilitates cross-chain asset transfers through a modular framework involving smart contracts and cryptographic verification. Below is a detailed step-by-step explanation of its workflow:

#### **1. Asset Locking on the Source Chain**
   - Users initiate a transaction on the source chain by interacting with the **LockProxy** contract.
   - The `LockProxy` contract locks the specified asset (e.g., ETH, BNB) on the source chain.
   - A transaction record is created and forwarded to the **EthCrossChainManager** contract for further processing.

#### **2. Cross-Chain Message Generation**
   - The `EthCrossChainManager` generates a **cross-chain message**, including details like:
     - Sender's address.
     - Recipient's address.
     - Asset type and amount.
     - Destination chain ID.
   - This message is hashed and added to a **Merkle Tree**, with the resulting Merkle Root stored in the manager contract.

#### **3. Message Relay**
   - Off-chain relayers monitor the source chain for new Merkle Roots.
   - Once a new root is detected, the relayer submits it to the destination chain's **EthCrossChainManager** contract.

#### **4. Message Verification on the Destination Chain**
   - The `EthCrossChainManager` on the destination chain verifies the submitted Merkle Root against the stored record.
   - If the Merkle Root is valid, the cross-chain message is unpacked and processed.

#### **5. Asset Release on the Destination Chain**
   - The **LockProxy** contract on the destination chain mints or releases an equivalent amount of the locked asset to the recipient's address.
   - For example, if ETH is locked on Ethereum, an equivalent amount of wrapped ETH (wETH) is released on the Binance Smart Chain.

#### **6. Finalization**
   - The transaction is finalized, and both chains update their state to reflect the completed cross-chain transfer.

---

## Root Cause



### Steps of the Exploit
1. **computing the 4byte signature**:
   - The attacker computed the 32-bit ID for putCurEpochConPubKeyBytes: ethers.utils.id ('putCurEpochConPubKeyBytes(bytes)').slice(0, 10)'0x41973cd9' 
   
   - The function putCurEpochConPubKeyBytes changes the ConKeepersPkBytes variable which contains publickey which has control the over funds of the protocal in that network.
   - ```
    // Store Consensus book Keepers Public Key Bytes
    function putCurEpochConPubKeyBytes(bytes memory curEpochPkBytes) public whenNotPaused onlyOwner returns (bool) {
        ConKeepersPkBytes = curEpochPkBytes;
        return true;
    }
   ````
2. **Brute forcing similar 4byte signature**:
   - The attacker brute-forced a string that, if set as _method in the code snippet above, gives the same 32-bit value. 
   - In this case the attacker used the string “f1121318093”: ethers.utils.id ('f1121318093(bytes,bytes,uint64)').slice(0, 10)'0x41973cd9' 

3. **Making a  cross-chain transaction**:
    - The attacker called a cross-chain transaction from the Ethereum network to the Poly network by triggering EthCrossChainManager and targeting EthCrossChainData, and passing the string f1121318093 as _method, and the public key of his Ethereum wallet as a parameter. 
    - [transaction hash](https://etherscan.io/tx/0xb1f70464bd95b774c6ce60fc706eb5f9e35cb5f06e6cfe7c17dcda46ffd59581#eventlog)

4. **Transfer of Funds**:
    - Once the transaction was executed and the attacker was granted the status of Keeper for the Ethereum blockchain, the attacker proceeded into using the corresponding secret key in their possession to funnel tokens out of Poly’s Ethereum wallet into their own wallet. 
    - The attacker repeated the above for other Poly liquidity wallets: Binance, Neo, Tether, etc. 
    - [ethereum funds transfer tx hash](https://etherscan.io/tx/0xad7a2c70c958fcd3effbf374d0acf3774a9257577625ae4c838e24b0de17602a)




## Impact
- **Stolen Assets**:
  - $273 million on Ethereum.
  - $253 million on Binance Smart Chain.
  - $85 million on Polygon.
- The total exploit amounted to **$610 million** in various cryptocurrencies, making it one of the largest DeFi hacks ever recorded.

---

## Recovery and Response
1. **Public Communication**:
   - Poly Network publicly appealed to the hacker, requesting the return of funds.
2. **Funds Recovery**:
   - The attacker, identifying themselves as a "white-hat," began returning the funds within days of the attack.
   - By August 25, 2021, almost all funds were returned, except for frozen Tether (USDT).
3. **Protocol Patches**:
   - Poly Network implemented fixes to address the vulnerabilities and conducted a comprehensive review of its smart contracts.


---

## Recommendations to Prevent Similar Hacks
1. **Robust Cryptographic Authentication**:
   - Use tamper-proof cryptographic signatures for all inter-chain messages.
2. **Enhanced Governance**:
   - Implement decentralized multi-sig governance for sensitive operations.
3. **Continuous Monitoring**:
   - Employ real-time monitoring tools to detect anomalies and respond swiftly to exploits.
5. **Emergency Mechanisms**:
   - Introduce circuit breakers to halt operations during suspicious activities.
4. **Bug Bounty Programs**:
   - Encourage white-hat hackers to identify vulnerabilities through incentivized bug bounty programs.




