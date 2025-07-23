# Bitcoin Runes Modules in Rosen Bridge

This document states the chain-specific modules of the Rosen Bridge and describes their approach for Bitcoin Runes.


## Contents
- [Chain Specific Modules](#chain-specific-modules)
- [Bitcoin Runes Implementation](#bitcoin-runes-implementation)
    - [Scanner](#scanner)
    - [Observation Extractor](#observation-extractor)
    - [Rosen Extractor](#rosen-extractor)
    - [Abstract Chain](#abstract-chain)


## Chain Specific Modules
Rosen Bridge core modules are designed on an abstract level, meaning that the supported chain and its traits do not affect it. Therefore, the requirements are split into common and chain-specific modules. For example, the `TxAgreement` module handles the consensus of transactions before sending them to the signing process. This module takes the transaction in the `PaymentTransaction` format and the chain of the transaction doesn't matter at all. For verifying the transaction, this module depends on the `ChainHandler` which contains the chain-specific actions of each chain.

The chain-specific modules of Rosen Bridge mainly are:

1. Scanner: For each blockchain, the watchers should process all the transactions and extract the required data. The scanner module handles fetching transactions and passing them to the extractors.
2. Observation Extractor: It extracts the transfer requests (the full data of the lock transaction with requested Rosen data) from scanner transactions.
3. Rosen Extractor: The core module of the Observation Extractor, which extracts data from the lock transaction. The structure of the lock transaction directly depends on the implementation of this module. There are two versions of this module:
    - Network-based, which is used on the watcher side and the transaction format is based on the network, e.g. RPC or explorer.
    - Universal, which is used on the guard side and the transaction format is the same as the format used on the chain-specific modules of the guard-service.

    This approach removes data conversion on the watcher side which accelerates the scanning process since all of the transactions are processed.
4. Address Codec: The module handling encoding and decoding of addresses for each chain which is used inside Rosen Extractor and UI.
5. Abstract Chain: The main chain-specific module of the guard service which handles all actions in a blockchain. The requirements and implementation of this module for a new blockchain is described both by [it's repository document](https://github.com/rosen-bridge/rosen-chains/blob/dev/packages/abstract-chain/README.md) and [Bridge Expansion Kit document](https://github.com/rosen-bridge/rcs/tree/master/rcs-003#abstract-chain-bases).


## Bitcoin Runes Implementation
Rosen uses the `bitcoinjs-lib` library for almost every action in the implementation of Bitcoin Runes chains same as Bitcoin chain since most of actions are identical. For the Runes protocol specific actions, [`@magiceden-oss/runestone-lib`](https://github.com/me-foundation/runestone-lib) is used. In the following, the implementation of each module is described alongside references to the codes and release packages.

> Note: In the previous documents, the `runelib` library was used. However, the library was not suitable as it alters the Runestone while decoding it, which hides the flaws and results in unexpected states and vulnerabilities.

> Note: In the previous documents, also stated that a Taproot address will be used for the lock address. However, it is decided to stick to the native-segwit, as its already supported in Rosen and removes the complexities of adding a new sign package.

### Scanner
Since Bitcoin chain is already supported and Runes is also processing the block transactions, no new scanner is required and the `BitcoinRpcScanner` will be used for Bitcoin Runes too.

Reference:
- Implementation: [GitHub](https://github.com/rosen-bridge/scanner/tree/dev/packages/scanners/bitcoin-rpc-scanner)

### Observation Extractor
The main part of the Observation Extractor is extracting data from the transaction using the Rosen Extractor module. For other supported chains, all of the data is written on the transaction itself and this module is implemented completely abstractly. However, in Bitcoin Runes lock transactions, the locked amount of Runes cannot be extracted directly from the transaction bytes. It is because of assigning unallocated Runes to the output specified by the `pointer` field and the first non-OPRETURN output in case of null `pointer`. To address this problem, The Runes observation extractor gets the core data (every observation data other than the locked token with its amount) from the Rosen Extractor module. If extraction was successful, the Rune transfer of the transaction is fetched by [Ordiscan API](https://ordiscan.com/) and the id and amount of the locked Rune is extracted and verified. 

Reference:
- Runes Observation Extractor Class: [GitHub](https://github.com/rosen-bridge/scanner/tree/dev/packages/observation-extractors/runes-observation-extractor)
- Released: [NpmJs](https://www.npmjs.com/package/@rosen-bridge/runes-observation-extractor)

### Rosen Extractor
In Rosen Bridge, the some fields are required to be specified while locking the asset. This data is serialized and wrote on the transaction in various ways. For Bitcoin Runes, this data is serialized into 3-4 UTxOs where its script (which is in native-segwit type) represents the data. Please refer to previous documents for exact specification.

Bitcoin Runes Rosen Extractor module is responsible to validate this structure and extract the Rosen Data. Unlike other chains, Rosen Extactor cannot verify the transferring token, not its ID nor its amount, as its function is ran synchronously while extracting and validating the token requires fething data from the network and should be run asynchronously. As previously explained, Rosen Extractor only extracts the core data and in case of successful extraction, the Observation Extractor continues, fetches and validates the transferring Runes.

Reference:
- Implementation: [GitHub](https://github.com/rosen-bridge/utils/tree/dev/packages/rosen-extractor/lib/getRosenData/runes)
- Rosen-Extractor Package: [NpmJs](https://www.npmjs.com/package/@rosen-bridge/rosen-extractor)

### Address Codec
Every new blockchain should have encoding, decoding and validation script in the `@rosen-bridge/address-codec` library. For Bitcoin Runes, the `bitcoinjs-lib` library which is also used for Bitcoin chain, is used, with a slight change in the validation function, where it does only accept native-segwit and taproot addresses. 

Reference:
- Implementation: [GitHub](https://github.com/rosen-bridge/utils/tree/dev/packages/address-codec)
- Rosen-Extractor Package: [NpmJs](https://www.npmjs.com/package/@rosen-bridge/address-codec)

### Abstract Chain
The functionalities of Bitcoin Runes in Abstract Chain are under four categories:

- **Transaction Utility**, such as generating, signing and sending transactions.

    Every Runes in the input boxes should be edicted in the transactions. This strict condition facilitates some verifications and enables some actions such as the `extractTransactionOrder`.

    To verify previous condition, the amount of Rune in each input boxes should be fetched from network, which requires all inputs boxes to be confirmed on the blockchain, which inhibits chaining the payment transactions.

    In order to keep boxes small, multiple change boxes will be created where each one contains up to 5 Runes.

    The structure of the generated transaction are as follow:
        1. User payment
        2. OP_RETURN, containing Rune transfer data
        3. change box(es)

    Since a native-segwit address will be used for the lock address, the signing will be done by TSS using `@rosen-bridge/tss` package, same as Bitcoin chain.

- **Verification**, which consists of multiple functions to verify a transaction generated by other guards and a function to verify an event. The conditions are:

    - Transaction fees cannot be much different from the expected value. The expected value is what the guard would use based on the current network status, and the difference is measured using the slippage config (e.g., if slippage is 10 percent and the guard expected fee is 100, the transaction is verified if the fee is between 90 and 110). This slippage handle fee fluctuation, as it's already done for Bitcoin and EVM chains.
    - Transaction may not have Runestone, but they cannot have a Cenotaph. 
    - In case of having Runestone, it should have `pointer` field, targeted to the change box.
    - As previously mentioned, all input Runes should be edicted. No more, no less.
    - No redundant edict should be present (i.E., there should be no two edict with same Rune ID and output index).

    Verifying an event is [implemented abstractly](https://github.com/rosen-bridge/rosen-chains/blob/63af639067321e01d66991868b746618bb4d04aa/packages/abstract-chain/lib/AbstractChain.ts#L162) and no extra condition is required for Bitcoin Runes.

- **Serialization**, which is trivial and completely similar to Bitcoin chain. It converts PSBT object of the `bitcoinjs-lib` library format to the `PaymentTransaction` structure used in the guard service, alongside input boxes, which includes previous output transaction id, index, amount of BTC and Runes in it.

- **Validation**. The only condition that makes a transaction invalid, is having one of it's inputs spent.

Reference:
- Implementation: [GitHub](https://github.com/rosen-bridge/rosen-chains/tree/dev/packages/chains/bitcoin-runes)
