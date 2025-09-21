# Bitcoin Runes Integration in Rosen Bridge

Final document on the integration of Bitcoin Runes in Rosen Bridge reports the testing result..

## Contents
- [Testing](#testing)
    - [Bridge to Bitcoin Runes](#bridge-to-bitcoin-runes)
    - [Bridge from Bitcoin Runes](#bridge-from-bitcoin-runes)

## Testing
In final stage, the Bitcoin Runes id deployed on Pandora, the test environment of Rosen Bridge on Mainnet (the addresses are available in [Contract Repository](https://github.com/rosen-bridge/contract/releases) with `pandora` suffix). Three Runes with different characteristics are bridged to Cardano. Also a supported Binance token is wrapped on Bitcoin as Runes and bridged. Here are summary of the events with images:

### Bridge to Bitcoin Runes

Event Id: `34e5024a55b6f4292c88984a6fde8cb87e3fa0af1bd973c90337e85552bd0011`
- Lock `rpnFoxtrot` on Binance: https://bscscan.com/tx/0xd4fe05ca18d3044f5bde466c64dba96c1429609e5cdc232036e81f8dfeb27974
- Commitments by Binance Watchers:
    - https://explorer.ergoplatform.com/en/transactions/6cb4c1a0bb2379a96e3cb30a77a0ebe79d76657eb72b5f9cdd0492c0d52ed67c
    - https://explorer.ergoplatform.com/en/transactions/918432aa06bb3acea86bf216c6129c6c8e757973480daec5709883ba4cf46946
    - https://explorer.ergoplatform.com/en/transactions/bd1637aad819da149c787b823c31996cb16f53b64a68f0594ed2981766c4c5d5
    - https://explorer.ergoplatform.com/en/transactions/d1822904c4b73287b5872c765d029133693aa49625add8e15a5793785c8c5887
    - https://explorer.ergoplatform.com/en/transactions/5216da79c3ac1a373eb8fdc2dcac7f4a2f6c04434668b2d19baf8b75dca6618a
- Event Trigger: https://explorer.ergoplatform.com/en/transactions/0a3445ed8bb126ce845d1bff6a59f595ce3e95ae0854866f7b31a8197235d0b8
- Payment on Bitcoin: https://web3.okx.com/explorer/bitcoin/tx/96d20dfc2605fb550170dd94af96122a44f3014cc38afe94ad448774e70f4f77
- Reward Distribution: https://explorer.ergoplatform.com/en/transactions/a8a680398003cfa0ba7d5c73904e77a476460081e4ba45eb56cfa7da8311d063

<p align="center">
    <img src="Screenshots/rpnFoxtrot.Binance-BitcoinRunes.png">
    <em>Bridge <code>rpnFoxtrot</code> from Binance to Bitcoin Runes</em>
</p>

<p align="center">
    <img src="Screenshots/rpnFoxtrot.Confirmation.png">
    <em>Confirmation Popup</em>
</p>

### Bridge from Bitcoin Runes

Event Id: 64ec5282210cc129afd65263f553dac0dbd230bc482050a8ee9939a70e6f1c74
- Lock `ROSEN•POC•RUNE` on Bitcoin: https://web3.okx.com/explorer/bitcoin/tx/baf082d24c4e878a2ca823cadac341509d767335fe0c478d75328f4d3b28fddf
- Commitments by Binance Watchers:
    - https://explorer.ergoplatform.com/en/transactions/47f07d02aefb86b44f24747f3f414e637e01408c9698c77c88cb0ec9a7f9aaff
    - https://explorer.ergoplatform.com/en/transactions/8c8bb361c0bf905c6d1bd2f626c54c3968f5f4aa2190774c96fe6a6c63071a9c
    - https://explorer.ergoplatform.com/en/transactions/97a6bf50c42b85ba68f1d449aa548028d95ab1bb8bdc53bba8992141018384db
    - https://explorer.ergoplatform.com/en/transactions/e561dcda8e0cc90c0a2bd7c59ccd3319b7200051448d72a73446bd41c5716bad
    - https://explorer.ergoplatform.com/en/transactions/5d59530979239d16538b3dcca4fd6254a6971c236da3707943c1cefa5a5165b3
- Event Trigger: https://explorer.ergoplatform.com/en/transactions/27b2ca87af989fca805863a185c84fcf1f78f0044b37aa527fd144fa21aef1a3
- Payment on Cardano: https://cardanoscan.io/transaction/0195a2922a97d8d5855f8641455ac4a7a1ea7756faca0c03430d82e08ee982af
- Reward Distribution: https://explorer.ergoplatform.com/en/transactions/ef1326d65426cb3a7545eb13f9a2ae7ca9c50fffab7fa9124d5f36e2cb1a93f8

<p align="center">
    <img src="Screenshots/RPOCR.BitcoinRunes-Cardano.png">
    <em>Bridge <code>ROSEN•POC•RUNE</code> from Binance to Bitcoin Runes</em>
</p>

<p align="center">
    <img src="Screenshots/RPOCR.Confirmation.png">
    <em>Confirmation Popup</em>
</p>

<table align="center">>
  <tr>
    <td align="center">
      <img src="Screenshots/wallet-popup1.png" width="250px"><br>
      <em>Wallet Popup (Assets)</em>
    </td>
    <td align="center">
      <img src="Screenshots/wallet-popup2.png" width="250px"><br>
      <em>Wallet Popup (UTxOs)</em>
    </td>
  </tr>
</table>

Event Id: `2c43a04e9de7ffe01445e4a27bee16c5ac201ebbc063f269ef70864d12e50cff`
- Lock `RPN•FOXTROT` on Bitcoin: https://web3.okx.com/explorer/bitcoin/tx/9425a8bcf982bad75928536bf26bb81b360360a78aa42d084db32421f4caa9fd
- Commitments by Binance Watchers:
    - https://explorer.ergoplatform.com/en/transactions/8ba2045bf6d068f86da149fd4e6d4046f587d98c0de3c3194f0f9d531ccfc33f
    - https://explorer.ergoplatform.com/en/transactions/09df770202dc43ab3316e7b23615255eb6b758360ee3175a58c97af565a59f39
    - https://explorer.ergoplatform.com/en/transactions/1fcef44e6477985789b16edab9b6cc1992d6de7f69889f8570ed166ae0057a7f
    - https://explorer.ergoplatform.com/en/transactions/a10aad458236a70536511f826935dd80efd03ee360baa0686cc6a835ea178fc4
    - https://explorer.ergoplatform.com/en/transactions/d1e6721e67a4896fbf906c0401eeddb37f44ddf7003d8336076a9dde4c279063
- Event Trigger: https://explorer.ergoplatform.com/en/transactions/249cf27b1fbaf6c2199be48e4964b5a1d6a0b93e200cc895ea2b34710d78031f
- Payment on Cardano: https://cardanoscan.io/transaction/bdc5a37131eb2a3deaa852897ff42fcf2a8d97f60be9e7652c14af2fbbf713c3
- Reward Distribution: https://explorer.ergoplatform.com/en/transactions/85108211f77de4149fcd58801214ebb516903a996b2a554e8329a6872a611343

Event Id: `e44db22fb4b721c748edf4255875f826c035f24a4a88423e349bb89f08a36aa4`
- Lock `PYTHAGORAS` on Bitcoin: https://web3.okx.com/explorer/bitcoin/tx/20757ed765fe283508d2db296d8a305aea3f203e01302c97f2455aeb69c42994
- Commitments by Binance Watchers:
    - https://explorer.ergoplatform.com/en/transactions/31ace2780d2fce4ef6f6718588edc1a4eaac19dfe4463b2d3eaf2a3df0f89ebe
    - https://explorer.ergoplatform.com/en/transactions/56377ace0b16838a18cbefc352aea692aa9be02015d5c25787dd391ee701089c
    - https://explorer.ergoplatform.com/en/transactions/628de490ca8733d2dcd1c7b1dbbef14701865a0f603c69d4633bb9c96bcfa28e
    - https://explorer.ergoplatform.com/en/transactions/be2b63ede83f8e47fff017303f2bbc1662a4ff7c2f16778cf581d05c1d9c301d
    - https://explorer.ergoplatform.com/en/transactions/c28ff9a6c427beaa63f72eddf13d838d71efd774bac440024eba7b61eb9c55be
- Event Trigger: https://explorer.ergoplatform.com/en/transactions/660d780c47cde8d84f5af4c74549f720622ce8d9f6d4d87c59a23ab35b157ec0
- Payment on Cardano: https://cardanoscan.io/transaction/707ea268e89af5e0d759cf30fcf98d87f3ed73d30d7fe3b6f999002dc7d3d103
- Reward Distribution: https://explorer.ergoplatform.com/en/transactions/8111699dfc358f893b3fb72c39af705df914916b2265020b38a25114b9d8db63

<p align="center">
    <img src="Screenshots/Events-page.png">
    <em>Summay of the events in <a href="https://ui-rosen-git-dev-rosen-bridge.vercel.app/events">Rosen Pandora Events Page</a></em>
</p>

