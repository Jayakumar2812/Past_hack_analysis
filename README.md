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
```
/**
     * @notice Initialize the replica
     * @dev Performs the following action:
     *      - initializes inherited contracts
     *      - initializes re-entrancy guard
     *      - sets remote domain
     *      - sets a trusted root, and pre-approves messages under it
     *      - sets the optimistic timer
     * @param _remoteDomain The domain of the Home contract this follows
     * @param _updater The EVM id of the updater
     * @param _committedRoot A trusted root from which to start the Replica
     * @param _optimisticSeconds The time a new root must wait to be confirmed
     */
    function initialize(
        uint32 _remoteDomain,
        address _updater,
        bytes32 _committedRoot,
        uint256 _optimisticSeconds
    ) public initializer {
        __NomadBase_initialize(_updater);
        // set storage variables
        entered = 1;
        remoteDomain = _remoteDomain;
        committedRoot = _committedRoot;
        // pre-approve the committed root.
        if (_committedRoot != bytes32(0)) confirmAt[_committedRoot] = 1;
        _setOptimisticTimeout(_optimisticSeconds);
    }
```

#### **Impact of the Initialization Flaw**
- The mapping `confirmAt[0x00]` was assigned a timestamp of `1`, which allowed any message with a root of `0x00` to bypass normal authentication and validation checks.
- This misconfiguration essentially rendered the zero hash (`0x00`) as a universal bypass key, as messages associated with this root were automatically deemed valid without further verification.

---

### 2. **Message Verification Bypass**

- The `Replica` contract is responsible for authenticating messages sent across chains. This is achieved through the `process()` function, which checks if the root of a given message is part of the `confirmAt` mapping.
- Normally, this mapping would only include trusted roots that had been validated through a rigorous process. However, due to the initialization flaw, the zero hash (`0x00`) was already included in the mapping with an early timestamp (`1`), meaning it was always valid.


```
/**
     * @notice Given formatted message, attempts to dispatch
     * message payload to end recipient.
     * @dev Recipient must implement a `handle` method (refer to IMessageRecipient.sol)
     * Reverts if formatted message's destination domain is not the Replica's domain,
     * if message has not been proven,
     * or if not enough gas is provided for the dispatch transaction.
     * @param _message Formatted message
     * @return _success TRUE iff dispatch transaction succeeded
     */
    function process(bytes memory _message) public returns (bool _success) {
        // ensure message was meant for this domain
        bytes29 _m = _message.ref(0);
        require(_m.destination() == localDomain, "!destination");
        // ensure message has been proven
        bytes32 _messageHash = _m.keccak();
        require(acceptableRoot(messages[_messageHash]), "!proven");
        // check re-entrancy guard
        require(entered == 1, "!reentrant");
        entered = 0;
        // update message status as processed
        messages[_messageHash] = LEGACY_STATUS_PROCESSED;
        // call handle function
        IMessageRecipient(_m.recipientAddress()).handle(
            _m.origin(),
            _m.nonce(),
            _m.sender(),
            _m.body().clone()
        );
        // emit process results
        emit Process(_messageHash, true, "");
        // reset re-entrancy guard
        entered = 1;
        // return true
        return true;
    }
```

#### **The Exploit Mechanism**
- Attackers identified this flaw and began crafting messages with the root value set to `0x00`.
- When these malicious messages were submitted to the `process()` function, the verification logic trusted the root due to the faulty `confirmAt` entry.
- As a result, the attackers were able to call the `process()` function multiple times, authorizing withdrawals of funds from the bridge’s reserves without proper validation.

---

### 3. **How the Exploit Unfolded**

- The simplicity of the exploit meant that multiple attackers, including opportunistic hackers and white-hat participants, were able to exploit the vulnerability simultaneously.
- Unlike sophisticated attacks that require significant technical expertise, this exploit could be carried out by copying and pasting the original attacker's transaction data and replacing the target addresses.
- This “free-for-all” nature of the exploit led to the rapid draining of funds, with over $190 million stolen within hours.

[Attack Transaction in Tenderly](https://dashboard.tenderly.co/tx/mainnet/0xa5fe9d044e4f3e5aa5bc4c0709333cd2190cba0f4e7f16bcf73f49f83e4a5460)
---

## Recommendations to Prevent Nomad Hacks

1. **Strengthen Optimistic Verification**:
   - Add stricter validation during the challenge period to detect and reject fraudulent messages proactively.
   - Implement cryptographic signatures on message roots to ensure they cannot be forged or tampered with.

2. **Improve Contract Initialization**:
   - Ensure critical variables like `_committedRoot` are set correctly during deployment to prevent trust bypass vulnerabilities.

3. **Enhance Safeguards**:
   - Add fallback mechanisms to pause operations during anomalies.

4. **Conduct Regular Audits and Fuzzing**:
   - Perform detailed audits after every deployment or upgrade.
   - Use fuzz testing to identify unexpected edge cases and vulnerabilities.

