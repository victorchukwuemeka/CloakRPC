# Notes on "Time Tells All: Deanonymization of Blockchain RPC Users"

This document provides a comprehensive summary of the research paper "Time Tells All: Deanonymization of Blockchain RPC Users with Zero Transaction Fee (Extended Version)" (arXiv:2508.21440v1).

## 1. Core Problem

-   **Primary Gateway, Primary Risk:** Remote Procedure Call (RPC) services (like Infura) are the main way users interact with blockchains (Ethereum, Bitcoin, Solana) via lightweight wallets (e.g., MetaMask).
-   **Anonymity Gap:** While these services offer significant convenience, they introduce critical privacy challenges that remain insufficiently examined. Existing deanonymization attacks either do not apply to blockchain RPC users or incur costs.
    -   **Address Clustering:** Some methods aim to cluster blockchain addresses belonging to the same user, but these cannot reveal users' real-world identities.
    -   **Transaction Source Node Identification:** Other methods link a transaction to the IP address of a blockchain node that first broadcasts it. However, RPC users' personal devices operate off-chain, with RPC service nodes acting as intermediaries. Thus, these methods cannot reveal the IP address of RPC users' personal devices.
    -   **Active Attacks:** Prior work on deanonymization of Ethereum RPC users assumed an active network adversary capable of injecting traffic, which incurs transaction fees and is more detectable.
-   **The Goal:** To link a user's real-world IP address to their blockchain pseudonym (wallet address) in a passive and zero-fee manner.

### 1.1 RPC-based Blockchain Communication

-   **Communication Paradigm:** Wallets on users' personal devices rely on RPC services to access blockchain networks. The RPC service provider operates full nodes for ledger synchronization and exposes standardized RPC APIs.
-   **Wallet Interaction:** Wallets send RPC requests and receive responses from these APIs.
-   **Protocol Stack:**
    -   **Application Layer:** JSON-RPC protocol, with structured JSON objects for requests (method, params, id) and responses (id, result).
    -   **Transport Layer:** HTTP or Websocket protocol, encapsulating the RPC JSON object.
    -   **Security Layer:** TLS protocol encrypts the TCP data, ensuring data integrity and confidentiality.
    -   **Network Layer:** Standard TCP/IP protocol.
-   **Attacker Visibility:** Due to TLS encryption, the actual content of RPC requests/responses (the RPC JSON object) is inaccessible to a passive attacker. However, metadata such as the quadruple `⟨source IP, source port, destination IP, destination port⟩`, timestamps, and packet sizes are always unencrypted and observable. This metadata is crucial for identifying RPC packets and inferring wallet-RPC service interactions.

## 2. The TRAP Attack (Timestamp Reveals Associated Pseudonym)

The paper proposes a novel, **passive**, and **zero-fee** deanonymization attack named TRAP.

### Threat Model

-   **Attacker:** A strong passive network adversary with access to network infrastructure, capable of monitoring traffic at key chokepoints such as border routers or Internet exchange points.
-   **Capability:** The attacker collects only the metadata of TCP packets, such as the quadruple of `⟨source IP, source port, destination IP, destination port⟩`, timestamps, and sizes, which are always unencrypted. The actual packet contents are encrypted by TLS and remain inaccessible to the attacker. The attacker also has access to public ledger data.
-   **Attack Cost:** This deanonymization attack incurs zero transaction fee.

### Core Vulnerability: Temporal Correlation

The attack's foundation is the time-based link between two key events:

1.  **`Tc` (Transaction Confirmation Timestamp):** The moment a transaction is officially confirmed and recorded in a block on the public ledger.
2.  **`Tq` (Transaction Query Timestamp):** The moment the user's wallet receives a response from the RPC service confirming that the transaction has succeeded.

The wallet, for good user experience, queries the transaction status shortly after it's confirmed on-chain. Therefore, `Tq` happens just after `Tc`, creating a predictable temporal relationship (`Tc ≈ Tq - Ic,q`).

## 3. Attack Methodology

The attack is a multi-step process designed to isolate a single user pseudonym from millions of transactions.

### Step 1: Identify Transaction Query Timestamp (`Tq`) from Traffic

-   **Challenge:** Traffic is encrypted. How to find the specific packet confirming a transaction?
-   **Solution:** Use Machine Learning to analyze TCP packet metadata.
    -   **Packet Size:** Different RPC API calls (e.g., `eth_getTransactionReceipt` for Ethereum, `blockchain.transaction.get_merkle` for Bitcoin, `getSignatureStatuses` for Solana) produce request/response objects of varying sizes, leading to distinct TCP packet sizes.
    -   **Packet Sequence:** Wallets have predictable logic. A transaction status query is often preceded and followed by other specific API calls (e.g., checking block number). This creates a unique "fingerprint" or sequence of packet sizes.
    -   **ML Model:** A Random Forest model (or similar basic ML models) can achieve high detection accuracy (e.g., 99.74% for Ethereum with `r=4`).
-   **Result:** The ML model can accurately detect the specific response packet for a transaction status query and extract its timestamp, `Tq`.

### Step 2: Estimate Confirmation Window (`k` blocks)

-   **Challenge:** The exact interval between `Tc` and `Tq` (`Ic,q`) is unknown.
-   **Solution:** Exploit wallet design. Wallets are built to be responsive. They query for status automatically and promptly.
    -   **Query Methods:** Wallets typically use either **periodic queries** (e.g., Ethereum/Solana wallets) or **notification subscriptions** (e.g., Bitcoin wallets).
    -   **Estimation:** The interval `Ic,q` is estimated based on the wallet's query cycle, network RTT, and blockchain block time (`z`).
-   **Result:** The attacker can reliably estimate an upper bound for this interval, which corresponds to a small number of blockchain blocks (e.g., `k=3` blocks for Ethereum, `k=60` for Solana, `k=2` for Bitcoin). This means the target transaction must be within the `k` blocks confirmed immediately before `Tq`.

### Step 3: Uniquely Identify the Pseudonym via Intersection

-   **Challenge:** The `k` blocks may contain hundreds or thousands of transactions from many different users.
-   **Solution:** Use multiple rounds of analysis and set intersection.
    1.  **Round 1:** The attacker observes a transaction from a victim IP. They identify `Tq`, determine the `k`-block window, and retrieve all pseudonyms from the transactions in those blocks. This forms the first candidate set, `S1`. The target's pseudonym, `st`, is in this set.
    2.  **Round 2+:** The attacker waits for the same victim IP to make another transaction. They repeat the process, generating a new candidate set, `S2`.
    3.  **Intersection:** The attacker computes the intersection of the sets (`Sm = S1 ∩ S2 ∩ ...`).
-   **Result:** Most blockchain users ("normal users") transact infrequently. The probability of a random user appearing in multiple, temporally separate candidate sets is extremely low. After just 3-4 rounds, the intersection `Sm` is highly likely to contain only one pseudonym: the target's.

### Optimization: Filtering Active Users

-   **Problem:** Highly active users (e.g., bots, exchanges) might appear in multiple sets by chance, leading to false positives.
-   **Solution:** The attacker calculates the transaction rate for every pseudonym in the final intersection set. If a pseudonym's rate exceeds a certain threshold, it's classified as an "active user" and filtered out. This significantly improves accuracy for normal users.

## 4. Validation and Results

-   **Blockchains Tested:** Ethereum, Bitcoin, and Solana.
-   **Wallets & RPC Services Tested:**
    -   Ethereum: MetaMask (Infura)
    -   Bitcoin: Electrum (Blockstream)
    -   Solana: Torus (QuickNode)
-   **Success Rate:** Over **95%** success in uniquely identifying a normal user's pseudonym after observing 3-4 transactions.
-   **Robustness:** The attack is effective regardless of the attacker's geographic location (e.g., on a router in the same city or across the globe) because the packet size/sequence features remain consistent. The method is also robust to normal network latency and jitters.

## 5. Impact and Responsible Disclosure

-   **Severity:** The findings demonstrate a widespread and severe privacy flaw in the fundamental architecture of how modern wallets interact with blockchains. It exposes millions of users to potential deanonymization by powerful adversaries.
-   **Responsible Disclosure:** The authors reported the vulnerability to 12 affected vendors.
    -   **CVE Assigned:** Electrum assigned **CVE-2025-43968**.
    -   **Bug Bounty:** Electrum awarded a $5,000 bounty.
    -   **Grants:** The Ethereum Foundation awarded an academic grant to research countermeasures.
    -   **Collaboration:** Several vendors are actively working with the researchers to implement mitigations.
-   **Ethical Considerations:** The research adhered to ethical guidelines, using passive monitoring, informed consent for data collection, and emulating transaction patterns to avoid network load.

## 6. Limitations and Countermeasures

-   **Countermeasures:**
    -   **Packet Padding:** Adding random data to packets to obscure size-based fingerprints.
    -   **Random Delays:** Introducing artificial, random delays before querying for transaction status to break the temporal correlation.
    -   **Trade-offs:** These countermeasures can be hard to implement effectively and may degrade user experience.
-   **IP-Masking:** Using tools like Tor or VPNs can be an effective defense, as they obscure the user's true IP address from the network observer. However, deanonymizing users behind Tor/VPNs remains an open problem.
-   **Experimental Limitations:** Real-world conditions might differ (e.g., network jitters, dynamic IPs, NAT, partial traffic visibility), potentially reducing success rates if insufficient transaction observations are made.