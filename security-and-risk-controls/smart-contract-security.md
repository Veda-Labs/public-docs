# Smart Contract Security

Governs the core vault architecture and operational safeguards, defining how the vault functions at a technical level and constraining its capabilities to pre-approved actions. Veda's framework for smart contract security involves:&#x20;

### Minimal Surface Area

Veda vaults expose only two public functions:&#x20;

* **Deposit** to supply assets
* **Withdraw** to redeem assets

This deliberately reduced surface area minimizes potential attack vectors, and clearly defines the user interaction model making it possible to build robust safeguards.

### Merkle Verification System

* Every action a vault can perform - whether deploying liquidity, staking or rebalancing - is pre-registered, hashed, and embedded in a Merkle tree
* Before executing any action, the vault must prove its inclusion in this Merkle root, ensuring that only pre-approved actions are executable
* This system makes it impossible for arbitrary transactions or strategy changes to occur

### Transaction Safeguards

* **Share Lock Period:** Newly issued vault shares are locked for a brief period to neutralize flash loan manipulation risks
* **Delayed Withdrawals:** Withdrawals are subject to a time delay, creating a monitoring window for identifying and responding to irregularities

### Onchain Monitoring

* Veda  tracks both internal positions and external market dynamics.&#x20;
* Additionally, Veda utilizes third-party monitoring systems to detect malicious behaviour targeting Veda contracts as well as the underlying protocols that Veda vaults take exposure to.
* Veda vaults are also part of multiple ongoing bug bounty programs operated by our integration partners.

### Audits

Veda contracts are leveraged by many of the top protocols in DeFi including ether.fi, Plasma, Lombard, TAC, Rings, and TurtleClub making the BoringVault one of the most audited DeFi contracts in production.

Audit firms commissioned to evaluate the BoringVault include Spearbit, Macro, Secure3 & Hexens.
