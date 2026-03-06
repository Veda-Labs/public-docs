---
description: The DeFi engine for financial applications.
---

# Introduction

## Overview

Veda is a DeFi vault primitive, which is a mechanism for pricing, accounting, securing, optimizing, and automating capital. Veda sits as layer above DeFi protocols (lending, DEXs, staking), packaging the best of DeFi into non-custodial, trust-minimized, and composable products. Veda empowers fintechs, asset issuers, exchanges, chains, wallets and applications to build enterprise-grade DeFi yield products without reinventing complex smart contract and offchain infrastructure.

Veda is the only vault infrastructure that is truly multi-chain and multi-protocol, allowing partners to access the best yields across chains without being locked into any single protocol. By abstracting away the intricacies of cross-chain operations, yield optimization, and risk mitigation, Veda significantly lowers the barriers to entry for onchain finance for both consumer and institutional participants. Protocols integrating Veda can seamlessly onboard users with transparent safeguards in real time.

Veda has demonstrated consistent market leadership in the vault and curation category. Notable achievements include:

* Recognized as the largest vault provider in DeFi, operating at billion-dollar scale for years with zero security incidents.
* Longest-running and largest general-purpose vault company in DeFi history.
* Developed the BoringVault, the most widely used vault standard across DeFi.
* Onboarded the first vault token, eBTC, onto Aave's main market.
* Powering Kraken DeFi Earn, EtherFi Liquid Vaults, Lido Earn, and distributed through Binance Web3 wallet and Bybit Web3 wallet.



## The Problem: DeFi is Complex, Fragmented, and Inefficient

### Friction in Accessing Yield

DeFi offers powerful financial tools, but accessing and optimizing yield remains unnecessarily complex:

* The same asset exists in multiple forms (staked, restaked, wrapped), each with different risks, liquidity, and utilities.
* Users must navigate multiple chains, manually bridging assets and acquiring the correct gas tokens.
* Every protocol has different interfaces, terminologies, and operational risks, creating an overwhelming experience.
* The optimal strategy to earn on your assets varies with market conditions and is increasing in complexity.
* Each business has its own compliance framework for offering DeFi products to its userbase.

### Unclear and Evolving Risk

Users are forced to assess smart contract and market risks manually, without clear, standardized disclosures:

* Risks evolve as protocols upgrade, making continuous risk assessment impractical.
* There are no automated protections, exposing users to systemic failures and exploits.
* Market conditions are unpredictable requiring vigilant monitoring on factors like liquidity, peg stability, price, protocol health, etc.

### Bootstrapping New Ecosystems

New ecosystems, whether that ecosystem revolves around a chain or an asset, require a critical mass of capital to function efficiently:

* Without deep liquidity, new ecosystems lack efficient markets, composability, and sustainable network effects.
* Vault-based liquidity solutions have proven effective, but teams shouldn't be forced to develop their own vault solution to kickstart this flywheel. Instead, they should be able to leverage generalized vault infrastructure that is both flexible and secure.

### Capital Retention and Liquidity Efficiency

Crypto applications and chains don’t just struggle to attract liquidity. They also struggle to retain it and make it productive:

* **Liquidity Mining is Unsustainable:** Incentive programs attract short-term capital, but capital leaves once rewards dry up and there is no longer an easy way to earn.
* **Liquidity Sits Idle:** Staked or bridged liquidity is often underutilized instead of being natively productive within the ecosystem. This is also true for DeFi applications where lack of time or too much complexity relegates users to adopt lazy positions that reduce a protocol’s capital efficiency.
* **Capital Fragmentation:** For asset issuers and multichain protocols, assets are spread across multiple chains and protocols, making it difficult to aggregate liquidity and sustain deep markets.

## Veda's Approach: A DeFi Vault Primitive

<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

### What Is a DeFi Vault Primitive?

A DeFi vault primitive is foundational infrastructure that allows developers to easily onboard capital, mitigate risk, and optimize ecosystem liquidity. Veda’s BoringVault anchors this process by providing:

* **Standardized Pricing & Accounting:** Tracks user balances and vault share valuations in real time.
* **Adaptive Allocation:** Dynamically allocates capital across different yield strategies.
* **Automated Optimization:** Continuously rebalances for optimal risk-adjusted profile.
* **Rapid Iteration:** Launch new vaults or update strategies in as little as 48 hours.

### Core Differentiators

* **Protocol-,Asset-, and Chain-Agnostic:** Works seamlessly across multiple blockchains including EVM, SVM and even MoveVM and integrates with various DeFi protocols.
* **Arbitrarily Complex Strategies:** Offchain logic combined with Veda's modular architecture allows vault curators/strategists to run any onchain yield strategy and even multiple strategies within a single vault
* **Verifiable Constraints:** Vault operations are constrained to a whitelisted set of actions that are viewable onchain.
* **Non-Custodial:** User funds reside in audited smart contracts, eliminating the need for centralized custody. Unlike centralized yield products, all Veda vault assets are fully on-chain and verifiable. Users can see their assets at any time through block explorers—even if the distribution partner goes offline.
* **Composable:** Easily integrates with other DeFi primitives like lending markets, DEXs, and yield trading protocols to build powerful new financial applications. In fact, Veda vault shares are the first vault tokens to ever be onboarded as collateral on Aave’s main market.
