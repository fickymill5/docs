# Architecture of an Exchange

What follows is the suggested architecture for a centralized Exchange. Even if your Exchange does not adopt this exact design, it defines the concepts that are used throughout the other pages in this section.

## General Structure

The suggested architecture uses a **Central Exchange wallet** with multiple **User reception wallets** controlled by the Exchange.

<figure markdown>
  ![General structure of an Exchange](exchanges-general.png){ loading=lazy .allow-zoom }
  <figcaption>General structure of an Exchange.</figcaption>
</figure>

### Exchange's Central Wallet (Hot)

This account contains the Exchange's funds required to perform user operations: Users' deposits are ultimately routed here, and users' withdrawals are taken from here.

The private keys to this account need to be on an online machine (the [Exchange server](#exchange-server)) so this is considered a **hot wallet**. For security reasons, it is recommended that the hot wallet only contains enough funds to perform daily operations, whereas the bulk of the funds are kept in the [cold wallet](#exchanges-central-wallet-cold).

### Exchange's Central Wallet (Cold)

The private keys to this account are kept in an offline machine so it is less vulnerable to attacks. Moreover, it is recommended that this is a multi-signature account so the approval of more than one administrator is required to move funds from it.

Periodically (e.g., once a day) funds are transferred from or to the [hot wallet](#exchanges-central-wallet-hot) so it can continue operating while the bulk of the funds are protected in the cold wallet.

### Users' Reception Wallets

When users sign up with the Exchange, a reception wallet is created for them in order to perform deposits. The reception wallets are usually empty: as soon as they receive funds these are transferred to the [Exchange's hot wallet](#exchanges-central-wallet-hot).

The private keys to the reception wallets always remain under the Exchange's control; these wallets are offered to users as a convenience only. Users cannot perform any operation on these wallets other than deposits.

### User's Wallet

This is the origin of deposits made to the Exchange and the receiver of withdrawals made from the Exchange. It can be a wallet in control of the user (the user holds its private key), a custodial wallet or another Exchange, for example.

### Exchange Server

This is an online server, part of the Exchange's infrastructure, that receives withdrawal requests from users and monitors the reception wallets to detect incoming deposits. It holds the private keys to the [hot wallet](#exchanges-central-wallet-hot) and to all the [reception wallets](#users-reception-wallets) so it can move funds from them in response to user's requests.

!!! caution
    This server must be available 24/7 so it is a clear target for malicious actors.

### Balances DB

This database keeps track of every user's funds, since the actual tokens from all users are pooled together in the hot and cold wallets.

The [Exchange server](#exchange-server) updates this DB in response to user's deposits and withdrawals.

### Flare Observer Node

An observer node is a regular Flare node that does not partake in [consensus](glossary.md#consensus) but is still aware of the current state of the blockchain and allows submitting transactions. **It is highly recommended** that Exchanges deploy their own observer nodes to access the network, instead of relying on third-party services. Read the [Deploying an Observer Node](../infra/observation/deploying.md) guide to learn how to do this.

## Detecting Deposits

The [Exchange server](#exchange-server) must be continuously monitoring transfers into **ALL** [reception wallets](#users-reception-wallets) to detect incoming deposits. Here's a summary of the process:

<figure markdown>
  ![Depositing to an Exchange](exchanges-deposits.png){ loading=lazy .allow-zoom }
  <figcaption>Depositing to an Exchange.</figcaption>
</figure>

1. The user **deposits** (transfers) funds to their assigned [reception wallet](#users-reception-wallets).

2. The transaction is **detected** by the [Exchange server](#exchange-server) monitoring the wallets.

    The server can discover a new transaction as soon as it is submitted by **subscribing** to the `pendingTransactions` event. This allows showing the transaction as "pending" in the UI, but there is still a chance that it is reverted.

    To avoid problems, the Exchange should only act on transactions appearing on blocks **old enough** for the chance of them being reverted to be negligible. This can be done by subscribing to the `newBlockHeaders` event and examining the transactions in a previous block (for example, 5 blocks behind).

    The code below exemplifies this process ([See the web3.js documentation](https://web3js.readthedocs.io) for the API details):

    ```  typescript
    // https://web3js.readthedocs.io
    const Web3 = require('web3');

    // Use your own node URL
    // https://docs.flare.network/dev/reference/coston-testnet/
    const web3 = new Web3("wss://coston-api.flare.network/ext/bc/C/ws");

    // Use your receiving wallet address
    const receivingAddress = "0x947c76694491d3fD67a73688003c4d36C8780A97";

    web3.eth.subscribe("pendingTransactions")
    .on("data", async (transactionHash) => {
        // New transaction hash received.
        // Retrieve the actual transaction.
        let tx = await web3.eth.getTransaction(transactionHash);
        // If it is directed to our address...
        if (tx.to === receivingAddress) {
            // Mark it as pending.
            console.log("Transaction", tx.hash, "is pending");
        }
    }).on("error", console.error);

    web3.eth.subscribe("newBlockHeaders")
    .on("data", async (blockHeader) => {
        // New block has been produced.
        // Retrieve a block old enough to be considered confirmed.
        let block = await web3.eth.getBlock(blockHeader.number - 5);

        // Get all its transactions.
        block.transactions.forEach(async (transactionHash) => {
            // Retrieve the actual transaction.
            let tx = await web3.eth.getTransaction(transactionHash);
            // If it is directed to our address...
            if (tx.to === receivingAddress) {
                // Mark it as confirmed.
                console.log("Transaction", tx.hash,
                    "is confirmed in block", block.number);
            }
        });
    }).on("error", console.error);
    ```

    !!! caution
        Note that all transactions from a block are retrieved **simultaneously** and this can easily trigger a rate limit on the node. A proper implementation should avoid this by **serializing requests** or **managing the request rate manually**.

3. The server then **checks the wallet address** to find which user account it belongs to, and **adds the received amount to the user's balance**.

4. The server **announces a transaction** to the network (through the Exchange's own [observer node](#flare-observer-node)) to move the received funds to the [hot wallet](#exchanges-central-wallet-hot).

    [See a JavaScript example in the Ethereum documentation](https://ethereum.org/en/developers/tutorials/sending-transactions-using-web3-and-alchemy/). Since you will be using your own node, you can skip the Alchemy part and directly use the `web3` package as in the example above.

5. The received funds are **transferred to the hot wallet** when the transaction is approved by the network. The reception wallets always remain empty.

## Performing Withdrawals

Users must request withdrawals directly to the [Exchange server](#exchange-server) **through its user interface**. After checking that the user has the required balance, the funds are transferred from the Exchange's hot wallet directly to the user's wallet. Here's a summary of the process:

<figure markdown>
  ![Withdrawing from an Exchange](exchanges-withdrawals.png){ loading=lazy .allow-zoom }
  <figcaption>Withdrawing from an Exchange.</figcaption>
</figure>

1. The user **requests a withdrawal** to the [Exchange server](#exchange-server). The request includes some kind of user ID, the requested amount and the destination wallet's address.

2. The server **checks** that the user has the required balance to perform the withdrawal.

3. The server **announces a transaction** to the network (through the Exchange's own [observer node](#flare-observer-node)) to move the requested funds from the [hot wallet](#exchanges-central-wallet-hot) to the requested destination address.

    [See a JavaScript example in the Ethereum documentation](https://ethereum.org/en/developers/tutorials/sending-transactions-using-web3-and-alchemy/). Since you will be using your own node, you can skip the Alchemy part and directly use the `web3` package as in the example above.

    !!! caution
        Please make sure you **sign the transaction** before submitting it, as shown in the example.
        Unsigned transactions are ignored by the network.

4. The requested funds are **transferred to the user's wallet** when the transaction is approved by the network.
