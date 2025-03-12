# Integrate New Blockchain Into Rosen Bridge
This document states the design of Bitcoin Runes integration into Rosen Bridge and and describes the selected approach. Please note that this represents the overall design, which may be subject to changes as the implementation progresses.


## Contents
- [Challenge](#challenge)
- [Method](#method)
- [Integration](#integration)


## Challenge
As the Bitcoin chain is already integrated into Rosen Bridge, most of requirements are addressed and only one of them differs in case of Runes. Rosen uses the `OP_RETURN` protocol to write bridge data on the transaction in Bitcoin, while the Runes protocol also uses the `OP_RETURN` to write protocol messages. The Runes encoding in `OP_RETURN` box is strict and no extra data is allowd. Also `OP_RETURN` has limit of ~80Bytes and it is not possible to combine these two. On the other side, transactions with multiple `OP_RETURN` boxes are not considered 'standard' and may raise unexpected issues (e.g. since it won't be propagated through the network, it may take several hours to be mined).

Therfore, a new solution for writing data on the chain is required in order to integrate Runes into Rosen Bridge. Different solutions are investigated, such as:

1. Write data in two different transactions on two different chains, one in Bitcoin containing Runes and on in Ergo containing bridging data
2. Write data in two different transactions on Bitcoin, one containing Runes data and one containing bridging data
3. Write data in sigle transaction on Bitcoin, use `OP_RETURN` for Runes and use taproot witness section to write bridging data

And a 4th method is decided to utilize the benefits of each one.


## Method
This method utilizes address script bytes and embeds bridging data in UTxO address and use the `OP_RETURN` for Runes. It is possible to convert bridging data to multiple Bitcoin addresses. Data is at most 80 bytes, where Bitcoin native-segwit addresses contain 20 bytes, representing hash of owner's public key. Thus, 4 more outputs are generated where their addresses are chunk of bridging data. A minimum 294 satoshis are required in these UTxOs (native-segwit addresses have lowest min UTxO value between addresses).

Although the transaction seems large (7 UTxOs are required, 1 containing bridged assets, 1 for `OP_RETURN`, 4 for bridging data and 1 as the change box), but it is still better than two Bitcoin transactions. The bridging data is splitted into 4 chunks of 20 bytes, generating one address for each one by prepending `'0014'` to it. The amount of satoshi for each address is increased by chunk index.

For example, serialized bridgning data is `'00000000007554fc820000000000962f582103f999da8e6e42660e4464d17d29e63bc006734a6710a24eb489b466323d3a9339'` (refer to [the `@rosen-bridge/rosen-extractor` package](https://github.com/rosen-bridge/utils/blob/dev/packages/rosen-extractor/lib/getRosenData/bitcoin/utils.ts) to see how data is serialized), it is splitted into 3 chunks (3 is enough in this case), representing:
- script `001400000000007554fc820000000000962f582103f9`, which is `bc1qqqqqqqqqw420eqsqqqqqqqyk9avzzqlewpctgk`
- script `001499da8e6e42660e4464d17d29e63bc006734a6710`, which is `bc1qn8dgumjzvc8ygex30557vw7qqee55ecsvavkfg`
- script `0014a24eb489b466323d3a9339000000000000000000`, which is `bc1q5f8tfzd5vcer6w5n8yqqqqqqqqqqqqqqmcs2uu`

Then 3 UTxOs are generated:
- address `bc1qqqqqqqqqw420eqsqqqqqqqyk9avzzqlewpctgk`, value: `294` satoshi
- address `bc1qn8dgumjzvc8ygex30557vw7qqee55ecsvavkfg`, value: `295` satoshi
- address `bc1q5f8tfzd5vcer6w5n8yqqqqqqqqqqqqqqmcs2uu`, value: `296` satoshi


## Integration

### Rosen Bridge’s Approach to Runes  
Within the Rosen Bridge, Runes is treated as a distinct blockchain. While its underlying architecture closely resembles Bitcoin, this separation allows for entirely independent codebases between Bitcoin and Runes. Maintenance is streamlined, and flexibility in integration and usage is enhanced. For example, taproot addresses are utilized for Runes lock transactions during bridging, whereas BTC transfers continue to rely on native-segwit addresses.

Performance is further optimized by isolating Runes-specific APIs, which are often slower and resource-intensive, from BTC transaction workflows.

### Address Format Configuration
Bitcoin Runes and BTC are managed as two independent chains within Rosen Bridge. BTC bridging remains supported by native-segwit addresses, while Runes transactions are exclusively handled through taproot addresses.

### Data Sources and Verification  
Rosen Bridge operates through two entities: **Watchers** and **Guards**, each requiring distinct API integrations.  

**Watchers**  
- Runes transfers are verified through a two-step process.  
- Transaction formats are first validated using existing Bitcoin module APIs.  
- If validated, Runes transfers are confirmed via secondary API calls.  
- This approach maintains scanning speeds comparable to Bitcoin’s native scanner and minimizes Runes API rate-limiting risks.  

**Guards**  
- The same verification process as Watchers is replicated for event validation.  
- Additionally, Runes balances and UTxOs associated with addresses must be retrieved.  
- Every transaction is rigorously checked for accurate Runes transfers and encoding compliance.  

### API Providers
Two primary Bitcoin Runes explorers, Ordiscan and Unisat, provide critical APIs:  

- **Ordiscan**  
  - [`Runes in tx` API](https://ordiscan.com/docs/api#runes-in-tx): Verifies Runes transfers in lock transactions for both Watchers and Guards.  

- **Unisat**  
  - [Address Runes Balance List](https://docs.unisat.io/dev/unisat-developer-center/bitcoin/runes/get-address-runes-balance-list): Fetches Runes balances for Guard services.
  - [Address Runes UTxOs](https://docs.unisat.io/dev/unisat-developer-center/bitcoin/runes/get-address-runes-utxo): Retrieves UTxO data for generating the transaction.

Although the Ordiscan API performs well for Watcher functions, concerns may arise regarding the rate limit imposed on Unisat APIs in the guard service. The actual impact of these limits can vary significantly based on the implementation, making it challenging to provide an accurate estimation of their effects.


### Library Implementation  
No existing API fully supports Runes transfer encoding/verification for unsigned transactions. After evaluation, [Runelib](https://www.npmjs.com/package/runelib) has been selected to handle encoding and decoding tasks. While verification logic must still be custom-developed, this library provides foundational functionality for Rosen Bridge’s requirements.
