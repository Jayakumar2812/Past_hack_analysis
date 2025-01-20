# Nomad Hack: Expanded Overview of the Protocol

**Nomad** is a cross-chain messaging protocol designed to facilitate interoperability between blockchains. It enables the transfer of tokens and arbitrary data across various blockchain ecosystems, making it a key player in the decentralized finance (DeFi) ecosystem, where cross-chain interactions are critical.

## Key Features of Nomad

### Cross-Chain Messaging:
- Nomad relies on a message-passing mechanism to relay data and transactions between blockchains.
- It uses a set of smart contracts deployed on different blockchains, allowing users to send tokens or instructions across chains.

### Token Bridges:
- Nomad is commonly used as a token bridge, enabling users to transfer assets like Ether (ETH), stablecoins, and other tokens between Ethereum, Avalanche, and other chains.
- For example, users lock tokens on Chain A, and equivalent tokens are minted or released on Chain B.

### Optimistic Verification:
- Nomad uses an optimistic model for message verification. In this model:
  - Messages are presumed valid unless proven otherwise.
  - This allows for lower gas fees and faster cross-chain transactions compared to models that rely on immediate cryptographic verification.
- Messages are verified by a set of validators, who can dispute invalid transactions during a predefined challenge period.

### Modularity:
- Nomad is designed to work with different blockchains through a modular architecture.
- It uses "home" and "replica" contracts:
  - **Home Contract**: Deployed on the source chain, managing the submission of cross-chain messages.
  - **Replica Contract**: Deployed on the destination chain, responsible for processing messages.

### Tech 
- The protocol comprises on-chain smart contracts and off-chain agents:
  - On-chain contracts implement Nomad's messaging API, enabling message queuing and state replication across different chains.
  - Off-chain agents secure and relay state across chains, forming the backbone of the messaging layer.


## Workflow of the Nomad Protocol

### **1. User Sends Tokens or Messages on the Source Chain**

1. **Locking Tokens**:
   - A user interacts with the **Home Contract** on the source chain to lock their tokens (e.g., ETH, USDC).
   - The Home Contract generates a **message** representing the token transfer request. This message includes:
     - Sender's address
     - Recipient's address on the destination chain
     - Token amount
     - Destination chain ID

2. **Message Commitment**:
   - The Home Contract bundles the transaction data into a **Merkle Tree**.
   - A **Merkle Root** is computed and stored in the Home Contract to track all pending cross-chain messages.

---

### **2. Message Relayed Off-Chain**

1. **Message Relayer**:
   - Off-chain agents (known as relayers) observe the state of the Home Contract and fetch the latest Merkle Root.
   - The relayer submits the Merkle Root to the **Replica Contract** on the destination chain.

2. **Optimistic Verification**:
   - The Replica Contract processes the submitted Merkle Root but does not immediately treat it as valid.
   - Instead, the root enters a **challenge period** where validators or other parties can dispute its validity by submitting fraud proofs.

---

### **3. Message Processing on the Destination Chain**

1. **Finalizing the Message**:
   - Once the challenge period expires without disputes, the Merkle Root is marked as valid in the Replica Contract.

2. **Unlocking Tokens**:
   - The user’s original message (locked in the Merkle Tree) is unpacked and verified.
   - The Replica Contract releases the equivalent tokens (e.g., wrapped assets) to the recipient on the destination chain.

3. **Disputes (if any)**:
   - If a fraudulent Merkle Root is detected during the challenge period, validators submit fraud proofs.
   - Fraudulent transactions are rejected, and penalties may be applied to dishonest relayers.


## Root Cause Analysis

### 1. **The Initialization Flaw**

- During an upgrade on **April 21, 2022**, a new version of the `Replica` contract was deployed. However, during the deployment, the contract's `_committedRoot` variable was set to `0x00` (the zero hash). 
- The `Replica` contract uses the `_committedRoot` to determine the validity of incoming cross-chain messages. This hash acts as a reference point for verifying whether a given message root is authorized for processing.
- By setting `_committedRoot` to `0x00`, the `confirmAt` mapping in the contract was updated such that the zero hash (`0x00`) was considered a valid and trusted root.

#### **Impact of the Initialization Flaw**
- The mapping `confirmAt[0x00]` was assigned a timestamp of `1`, which allowed any message with a root of `0x00` to bypass normal authentication and validation checks.
- This misconfiguration essentially rendered the zero hash (`0x00`) as a universal bypass key, as messages associated with this root were automatically deemed valid without further verification.

---

### 2. **Message Verification Bypass**

- The `Replica` contract is responsible for authenticating messages sent across chains. This is achieved through the `process()` function, which checks if the root of a given message is part of the `confirmAt` mapping.
- Normally, this mapping would only include trusted roots that had been validated through a rigorous process. However, due to the initialization flaw, the zero hash (`0x00`) was already included in the mapping with an early timestamp (`1`), meaning it was always valid.

#### **The Exploit Mechanism**
- Attackers identified this flaw and began crafting messages with the root value set to `0x00`.
- When these malicious messages were submitted to the `process()` function, the verification logic trusted the root due to the faulty `confirmAt` entry.
- As a result, the attackers were able to call the `process()` function multiple times, authorizing withdrawals of funds from the bridge’s reserves without proper validation.

---

### 3. **How the Exploit Unfolded**

- The simplicity of the exploit meant that multiple attackers, including opportunistic hackers and white-hat participants, were able to exploit the vulnerability simultaneously.
- Unlike sophisticated attacks that require significant technical expertise, this exploit could be carried out by copying and pasting the original attacker's transaction data and replacing the target addresses.
- This “free-for-all” nature of the exploit led to the rapid draining of funds, with over $190 million stolen within hours.

---

## Conclusion

The Nomad Bridge hack was a direct result of a **misconfiguration during initialization** and the lack of robust safeguards in the `Replica` contract. Specifically:
1. The `_committedRoot` being set to `0x00` allowed the zero hash to bypass authentication checks.
2. The `confirmAt` mapping, improperly initialized, allowed attackers to exploit the system by submitting unauthorized messages with the zero root.

This incident highlights the importance of:
- **Rigorous testing and validation** during smart contract deployments and upgrades.
- Ensuring initialization logic does not inadvertently introduce vulnerabilities.
- Implementing fallback mechanisms to pause or mitigate damage when critical issues arise.

The hack is a stark reminder of how even small oversights in contract initialization can lead to catastrophic consequences in decentralized systems.
