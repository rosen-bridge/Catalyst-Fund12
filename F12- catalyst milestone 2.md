# Integrate Runes Into Rosen Bridge
This document states the in-depth explanation of Runes interaction with Rosen Bridge.

## Contents
- [Transfer](#transfer)
  - [Data Serialization Across Chains](#data-serialization-across-chains)
  - [Rosen Data Structure](#rosen-dat-structure)
  - [Example](#example)
  - [Notable Steps to Generate the Lock Transaction](#notable-steps-to-generate-the-lock-transaction)
- [Extract](#extract)
  - [Steps for Extracting Rosen Data](#steps-for-extracting-rosen-data)

## Transfer
On previous document, the overall design of Runes and its integration into Rosen Bridge is described. A single transaction on Bitcoin includes both the Rune and Rosen data. To enable this, the serialization process for Rosen data is first explained.

### Data Serialization Across Chains
The serialization process for Rosen data is designed to be flexible across chains, allowing variations depending on the specific requirements of each blockchain. For instance:

- On Ergo Chain, the required fields are serialized as an array of strings and stored in a single register of the UTxO (specifically `R4`).
- On Cardano, the data is serialized as JSON and stored in the `metadata` field of the transaction.
- On Bitcoin and EVM Chains, the required fields are serialized as a single hex string. This hex string is stored in the `OP_RETURN` field of the UTxO on Bitcoin and in the `data` field of the transaction on EVM chains.

The serialization process for Runes is intended to follow the same approach as Bitcoin and EVM chains.

### Rosen Data Structure
The structure of Rosen data is defined as follows:

- Target chain: 1 byte
- Bridge fee: 8 bytes
- Network fee: 8 bytes
- Encoded destination address length: 1 byte
- Destination address: Variable length, restricted by the `@rosen-bridge/address-encoder` library, which is used to encode and decode addresses.

> The parser of this data can be found [here](https://github.com/rosen-bridge/utils/blob/5210808d2f8dadb788db83637bc51b90f00e011e/packages/rosen-extractor/lib/getRosenData/evm/utils.ts#L15).

### Example
An example serialization is provided for a transfer to a Cardano address with the following details:

```
010000000000001388000000000000089839011b2eff6f19da9c786a5be52986e4fa888754f48fdc82639073e4b9827cba6224d854f00de94428ff0024e297ad5ab773a19ed6fac80c40de
```

- `01` represents the index of Cardano in the supported chains list of Rosen Bridge
- `0000000000001388` represents **5000** unit, the bridge fee
- `0000000000000898` represents **2200** unit, the newtork fee
- `39` represents length of the target address, which is 57 bytes
- `011b2eff6f19da9c786a5be52986e4fa888754f48fdc82639073e4b9827cba6224d854f00de94428ff0024e297ad5ab773a19ed6fac80c40de` represents the encoded Cardano address (Refer to [`@rosen-bridge/address-coded`](https://github.com/rosen-bridge/utils/blob/5210808d2f8dadb788db83637bc51b90f00e011e/packages/address-codec/lib/encoder.ts#L29) for encoding implementation)

### Notable Steps to Generate the Lock Transaction
The process to generate the lock transaction involves the following steps:

1. **Generate the Runes Transfer Script**:
  The transfer script, which will be placed in the `OP_RETURN` UTxO, is generated as follows:

  ```ts
  // generate rune data
  const ROSEN_POC_RUNE = {
    id: '880887:3052',
    block: 880887,
    index: 3052
  }
  const runeId = new runelib.RuneId(ROSEN_POC_RUNE.block, ROSEN_POC_RUNE.index)
  const edicts = [
    new runelib.Edict(runeId, 250000n, 0),
  ]

  const stone = new runelib.Runestone(
    edicts,
    runelib.none(),
    runelib.none(),
    runelib.some(1),
  );
  const opReturnData = stone.encipher();
  ```

2. **Generate and Split the Lock Data**:
  The lock data is generated and split into chunks of **20 bytes** for storage.

  ```ts
  // generate lock data
  const minUtxoValue = 294n;
  const targetChain = 'cardano'
  const cardanoPandoraAddress = 'addr1qydjalm0r8dfc7r2t0jjnphyl2ygw4853lwgycusw0jtnqnuhf3zfkz57qx7j3pgluqzfc5h44dtwuapnmt04jqvgr0qwd9mqk'
  const bridgeFee = '5000'
  const networkFee = '2200'
  const lockData = generateEventData(
    targetChain,
    cardanoPandoraAddress,
    bridgeFee,
    networkFee
  )
  const lockDataChunks = lockData.match(/.{1,40}/g);
  if (!lockDataChunks) throw Error(`Failed to split lock data [${lockData}] into chunks`)
  ```

  > **Note**: Depending on the size of the destination address, the data may be split into either 3 or 4 chunks.

  The function `generateEventData` responsible for this step appears as follows:

  ```ts
  export const generateEventData = (
    toChain: string,
    toAddress: string,
    bridgeFee: string,
    networkFee: string
  ): string => {
    // parse toChain
    const toChainCode = ['ergo', 'cardano', 'bitcoin', 'ethereum', 'binance', 'doge', 'runes'].indexOf(
      toChain
    );
    const toChainHex = toChainCode.toString(16).padStart(2, '0');

    // parse bridgeFee
    const bridgeFeeHex = BigInt(bridgeFee).toString(16).padStart(16, '0');

    // parse networkFee
    const networkFeeHex = BigInt(networkFee).toString(16).padStart(16, '0');

    // parse toAddress
    const addressHex = encodeAddress(toChain, toAddress);
    const addressLengthCode = (addressHex.length / 2)
      .toString(16)
      .padStart(2, '0');

    return (
      toChainHex + bridgeFeeHex + networkFeeHex + addressLengthCode + addressHex
    );
  };
  ```

3. **Calculate Change Amount and Fee**:
  The change amount and transaction fee are calculated, and the change box is created. For simplicity, a fixed `vSize` is used in this example. However, in a real scenario, the `vSize` of the transaction can be calculated accurately.

  ```ts
  // calculate fee and change
  const estimatedVsize = 302;
  const fee = BigInt(Math.ceil((await getFeeRatio(network)) * estimatedVsize));

  const requiredSatoshiForLockData = 
    (Number(minUtxoValue) * lockDataChunks.length) + 
    Math.ceil(lockDataChunks.length * (lockDataChunks.length - 1) / 2);
  ```

4. **Generate, Sign, and Finalize the Transaction**:  
  The transaction is completed by adding UTxOs, signing, and finalizing the transaction. Outputs are added during this step, as shown in the code snippet below:  

  ```ts
  //-- add outputs
  //---- add lock address
  const receiverValue = 500n;
  const rosenLockAddress = 'bc1px0ad45qrfwc20yfd9wljeytrvfa6tmrcxv6pgxze2svvx00tp7mstj5rpk'
  psbt.addOutput({ script: bitcoinJs.address.toOutputScript(rosenLockAddress), value: Number(receiverValue) });
  //---- add change utxo
  psbt.addOutput({ script: taprootPayment.output, value: Number(inSatoshi - receiverValue - BigInt(requiredSatoshiForLockData) - fee) });
  //---- add Rune transfer (OP_RETURN utxo)
  psbt.addOutput({
    script: opReturnData,
    value: 0,
  });
  //---- split lock data into multiple utxos
  lockDataChunks.forEach((chunk, index) => {
    psbt.addOutput({
      script: Buffer.from(`0014${chunk.padEnd(40, '0')}`, 'hex'),
      value: Number(minUtxoValue) + index,
    });
  } )
  ```

> [**View The Proof of Concept transaction**](https://mempool.space/tx/ac16759cc66ad1f4b9fe49e068d979728302ed6fb566d94665c76a654a93eeb2)

> [Full script](https://gist.github.com/RaaCT0R/695694a32b475e44bf50264086d8be61)

## Extract
The extraction of Rosen data is performed by two entities: **Watchers** and **Guards**. While both extract the same data, the APIs they utilize for this purpose differ slightly.  

- **Watchers**: These entities scan blockchain networks, process all the transactions within a block, and attempt to extract Rosen data from them.  
- **Guards**: In contrast, guards only extract Rosen data from specific transactions reported by watchers (commonly referred to as "lock transactions").  

It is important to note that the process of extracting Rosen data itself remains unchanged between watchers and guards. For the sake of simplicity, this document focuses on the **watchers’ extraction process**. The only difference for guards is that the transaction data originates from a different API, while the remaining steps of the process are identical.  

### Steps for Extracting Rosen Data
1. **Fetch and Process All Transactions in the Block**:
  All transactions within the block are fetched and processed as follows:

  ```ts
  export const processBlock = async () => {
    const targetBlockId = '00000000000000000000a34330cefa5f08c714713b84a0595969c7e6b1326123'

    // get all transactions of the block
    const blockTxs = await getBlockTxs(targetBlockId);

    for (const tx of blockTxs) {
      // for the sake of simplicity, we ignore other txs
      if (tx.txid !== 'ac16759cc66ad1f4b9fe49e068d979728302ed6fb566d94665c76a654a93eeb2') continue;

      // try to extract data from the transaction
      await extractRuneTransferPoc(tx);
    }
  }
  ```

  The function designed to fetch block transactions appears as follows:

  ```ts
  /**
   * returns transactions in a block with specified hash
   * @param blockHash
   */
  const getBlockTxs = async (
    blockHash: string
  ): Promise<Array<BitcoinRpcTransaction>> => {
    const randomId = randomBytes(32).toString('hex');
    const blockHashResponse = await Axios.post<JsonRpcResult>(bitcoinRpcUrl, {
      method: 'getblock',
      id: randomId,
      params: [blockHash, 2], // verbosity should be 2 in order to retrieve full transaction info
    });
    if (blockHashResponse.data.id !== randomId)
      throw Error(`UnexpectedBehavior: Request and response id are different`);
    const blockTxs = blockHashResponse.data.result.tx;

    return blockTxs;
  };
  ```

2. **Validation 1: Check for Lock UTxO Presence**:
  The first validation ensures that a lock UTxO is present in the transaction:

  ```ts
  // validate lock conditions
  for (let i = 0; i < outputs.length; i++) {
    const output = outputs[i];
    if (output.scriptPubKey.hex === lockScriptPubKey) {
      validLock = true;
      break;
    }
  }
  if (!validLock) {
    console.debug(
      baseError + `: Failed to find rosen lock utxo`
    );
    return undefined;
  }
  ```

3. **Validation 2: Verify Valid Lock Data**:
  The second validation checks whether valid lock data can be extracted from the transaction’s UTxOs.

  ```ts
  // validate data conditions
  const minUtxoValue = 294;
  const lockDataChunks: Array<{ index: number, data: string }> = []
  for (let i = 0; i < 4; i++) {
    for (let boxIndex = 0; boxIndex < outputs.length; boxIndex++) {
      const output = outputs[boxIndex];
      if (output.value * 10**8 !== minUtxoValue + i) continue; // wrong data index
      if (output.scriptPubKey.hex.slice(0, 4) !== '0014') continue; // not a native-segwit utxo
      lockDataChunks.push({
        index: i,
        data: output.scriptPubKey.hex.slice(4)
      })
    }
  }
  let lockData: OnChainLockData | undefined;
  try {
    lockData = parseRosenData(lockDataChunks.map(chunk => chunk.data).join(''));
    validData = true;
  } catch (e) {
    console.debug(
      `Failed to extract data from chunks [${JsonBigInt.stringify(lockDataChunks)}]: ${e}`
    );
  }
  if (!validData || !lockData) {
    console.debug(
      baseError + `: Failed to extract rosen data from utxos`
    );
    return undefined;
  }
  ```

  As previously described, the data is ordered by the amount of satoshis in the UTxOs. This ensures that the order of UTxOs in the transaction does not affect the extraction process, facilitating accurate data parsing.

  The data parsing function appears as follows:

  ```ts
  /**
   * extracts rosen data from raw hex data
   * @param scriptPubKeyHex
   */
  export const parseRosenData = (scriptPubKeyHex: string): OnChainLockData => {
    // parse toChain
    const toChainHex = scriptPubKeyHex.slice(0, 2);
    const toChainCode = parseInt(toChainHex, 16);
    if (toChainCode >= SUPPORTED_CHAINS.length)
      throw Error(
        `invalid toChain code, found [${toChainCode}] but only [${SUPPORTED_CHAINS.length}] chains are supported`
      );
    const toChain = SUPPORTED_CHAINS[toChainCode];

    // parse bridgeFee
    const bridgeFeeHex = scriptPubKeyHex.slice(2, 18);
    const bridgeFee = BigInt('0x' + bridgeFeeHex).toString();

    // parse networkFee
    const networkFeeHex = scriptPubKeyHex.slice(18, 34);
    const networkFee = BigInt('0x' + networkFeeHex).toString();

    // parse toAddress
    const addressLengthCode = scriptPubKeyHex.slice(34, 36);
    const addressHex = scriptPubKeyHex.slice(
      36,
      36 + parseInt(addressLengthCode, 16) * 2
    );
    const toAddress = decodeAddress(toChain, addressHex);

    return {
      toChain,
      toAddress,
      bridgeFee,
      networkFee,
    };
  };
  ```

4. **Validation 3: Ensure a Valid Rune Transfer**:  
  The final validation verifies whether a valid Rune transfer occurred in the transaction. This is performed using the [Ordiscan API](https://ordiscan.com/docs/api#runes-in-tx):

  ```ts
  // validate rune conditions and fill token info in data
  let runeTransformation: TokenTransformation | undefined;
  try {
    const txRunesTransfer = await getTxRuneTransfer(tx.txid);
    for (const outRune of txRunesTransfer.data.outputs) {
      // check if rune is transferred to the lock address
      if (outRune.address !== rosenLockAddress) continue;

      // check if rune is supported by Rosen bridge
      const wrappedRune = tokenMap.search(RUNES_CHAIN, {
        tokenId: outRune.rune,
      });
      if (wrappedRune.length > 0 && Object.hasOwn(wrappedRune[0], lockData.toChain!)) {
        runeTransformation = {
          from: outRune.rune,
          to: tokenMap.getID(wrappedRune[0], lockData.toChain!),
          amount: outRune.rune_amount,
        };
        break;
      }
    }
  } catch (e) {
    console.debug(
      `Failed to get Runes data from tx [${tx.txid}]: ${e}`
    );
  }
  if (!runeTransformation) {
    console.debug(`No supported Rune is locked`);
    return;
  }
  ```

  The function designed to fetch rune transfers within a transaction appears as follows:

  ```ts
  /**
   * returns the Runes transfer of a transaction according to Ordiscan
   * @param txId 
   */
  const getTxRuneTransfer = async (txId: string): Promise<OrdiscanRunesTransfer> => {
    const apiKey = ''
    const headers: AxiosHeaders = new AxiosHeaders()
    headers.setAuthorization(`Bearer ${apiKey}`)
    const res = await Axios.get(
      `${ordiscanUrl}/v1/tx/${txId}/runes`,
      { headers: headers }
    )
    return res.data;
  }
  ```

  Since other conditions for the lock address are validated before reaching this step, the API is only called for confirmed lock transactions. This minimizes the risk of triggering rate limits.

> **Important Note**: For simplicity, the token map portion of the script is omitted.
```diff
// check if rune is supported by Rosen bridge
-const wrappedRune = tokenMap.search(RUNES_CHAIN, {
-  tokenId: outRune.rune,
-});
-if (wrappedRune.length > 0 && Object.hasOwn(wrappedRune[0], lockData.toChain!)) {
-  runeTransformation = {
-    from: outRune.rune,
-    to: tokenMap.getID(wrappedRune[0], lockData.toChain!),
-    amount: outRune.rune_amount,
-  };
+if (outRune.rune === 'ROSENPOCRUNE'){
+  runeTransformation = {
+    from: outRune.rune,
+    to: 'a4cb7e83170016f62375f4168c97eda13ad9a6c9f37e1bc19cfeee2a.7273526f73656e506f4352756e65',
+    amount: outRune.rune_amount,
+  };
  break;
}
```

By executing this script, the Rosen data is successfully extracted:

```
Valid Rosen Rune event is found:
{
  "toChain": "cardano",
  "toAddress": "addr1qydjalm0r8dfc7r2t0jjnphyl2ygw4853lwgycusw0jtnqnuhf3zfkz57qx7j3pgluqzfc5h44dtwuapnmt04jqvgr0qwd9mqk",
  "bridgeFee": "5000",
  "networkFee": "2200",
  "fromAddress": "box:32a02f0d2612225bd41e82d60f80844ae006d10a836e80cef7a83d9ebb9fa92a.0",
  "sourceChainTokenId": "ROSENPOCRUNE",
  "amount": "250000",
  "targetChainTokenId": "a4cb7e83170016f62375f4168c97eda13ad9a6c9f37e1bc19cfeee2a.7273526f73656e506f4352756e65",
  "sourceTxId": "ac16759cc66ad1f4b9fe49e068d979728302ed6fb566d94665c76a654a93eeb2"
}
```

This extracted data corresponds exactly to the encoded data described in the previous section.

> [Full script](https://gist.github.com/RaaCT0R/ca655615f26dd651342dc4b99a6accb4)
