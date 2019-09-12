# Burstcoin-Exchange-Integration

1 Introduction
-
This document explains how to integrate Burstcoin into a currency exchange platform

By CurbShifter [based on this NXT document](https://nxtwiki.org/wiki/Exchange_Integration) available under GNU Free Documentation License 1.3 or later


2 Table of Contents
-

1. Introduction

2. Table of Contents

3. Prerequisites

4. Accepting Deposits

4.1. Message-Based Deposits

4.2. Account-Based Deposits
 
5. Withdrawing / Sending Money

5.1 Adding a Message To a (Payment) Transaction

5.2 Hot and Cold Wallets

5.3 Additional information

5.3.1 Account Format

5.3.2 Burst and NQT Amounts

5.3.3 Asset and currency QNT amounts

5.3.4 Minimum Fee


3 Prerequisites
-
Burstcoin node running 24/7 with fully synchronized blockchain.
[Burstcoin Reference Software](https://github.com/burst-apps-team/burstcoin/releases)

It is also recommend to [setup a testnet node](https://burstcoin.community/testnet/) for QA and testing purposes.

All examples in this document use the 6876 testnet port. The mainnet port is 8123.

4 Accepting Deposits
-
Unlike Bitcoin's throw away addresses, Burstcoin addresses are somewhat expensive to maintain.

Each address and public key combination are maintained forever in the blockchain, however maintaining a completely empty account is less costly than maintaining an account with non zero balance.

Due to this we do not recommend using a throw away address per deposit. Instead, we recommend creating at most one deposit address per user or even direct all deposits to the same address and maintain user identity using a message attached to the deposit transaction.

Deposits can be accepted using one of the following approaches:

1. Message Based Deposits - sent to a single address, the user is identified by an attached message.
2. Address per User - a new deposit address for each user.

When using both methods, always use a strong passphrase for the deposit account and protect the deposit account with a public key before accepting deposits.

4.1 Message-Based Deposits
-
Each payment transaction can have an attached message, either in plain text or encrypted. This allows you to identify the customer through a customer number or order number mentioned in the message.

Which identifier you will use in the message to connect the payment to a specific user account is up to you and won’t be discussed in this document.

To monitor your account for new incoming payments, the `getAccountTransactions` API call is used. You can specify the amount of confirmations needed. (10 recommended)

The API call takes the following parameters:

1. account - deposit account id or in reed solomon format (with BURST prefix).
2. timestamp - if specified, transactions should be newer than this block timestamp
3. type - the type of transaction. For payments this should be 0
4. subtype - the transaction subtype. For payments this should be 0 as well
5. firstIndex - for pagination purposes
6. lastIndex - for pagination purposes
7. numberOfConfirmations - ignore transations with less
 
To monitor a specific account for payment transactions use the following URL:

> [http://localhost:6876/burst?requestType=getAccountTransactions&account=BURST-5BE2-6SGA-K455-BCCY3&type=0&subtype=0&numberOfConfirmations=10]( http://localhost:6876/burst?requestType=getAccountTransactions&account=BURST-5BE2-6SGA-K455-BCCY3&type=0&subtype=0&numberOfConfirmations=10)

> note; examples use local node and testnet port

This is the JSON response (irrelevant fields omitted):

    
    {
    "transactions": [
    {
    "type": 2,
    "subtype": 1,
    "timestamp": 132622393,
    "deadline": 1440,
    "senderPublicKey": "64ecf79e02001408d86a3148192f10abb94a15f1add7fb7e0d581d0efe406306",
    "recipient": "10446462338210047360",
    "recipientRS": "BURST-5BE2-6SGA-K455-BCCY3",
    "amountNQT": "0",
    "feeNQT": "100000000",
    "signature": "1685ba8dc12b71726406693045b4421e42b1fe57cd42428c6b97701271f1980624f8be54d6ca210fd29bb63abe652ae6beff98095e7931a1df2d82dc6c67e7ef",
    "signatureHash": "155df32a81847c00df529c9d09bc5e1f26618036f68554240b94ab99a50932cb",
    "fullHash": "20aa5e79d5a49b5c07d877fcb4bf85cdc2bb058650f9228f66521799be0bc037",
    "transaction": "6673108509650758176",
    "attachment": {
    "version.AssetTransfer": 1,
    "asset": "3509939581101213262",
    "version.Message": 1,
    "message": "Free RAFFLE share",
    "messageIsText": true,
    "quantityQNT": "10"
    },
    "sender": "13476626730621357107",
    "senderRS": "BURST-JM3M-MHWM-UVQ6-DSN3Q",
    "height": 548306,
    "version": 1,
    "ecBlockId": "17903993250352434776",
    "ecBlockHeight": 548297,
    "block": "13565806093770253653",
    "confirmations": 115271,
    "blockTimestamp": 132622449
    },
       ... more transactions ...
      ]
    }
    

Loop over this array of transactions and process the transactions one by one. Note that this response includes both incoming and outgoing payment transactions. You should filter out your own (outgoing) payments by looking at the sender or senderRS account address.

The important information of a transaction response is:

1. senderRS - the sender’s account id
2. confirmations - the number of confirmations
3. amountNQT - the amount sent in NQT form (plancks, like SATs in BTC)
4. attachment.message - optional, an attached plain text message
5. attachment.encryptedMessage - optional, an attached encrypted message
6. timestamp - the time the transaction was made, in seconds since the genesis block
7. blockTimestamp - the time of the block since the genesis block
8. confirmations - number of confirmations received for the block in which the transaction is included

### confirmations ###

For most transactions, waiting for 10 confirmations should be enough. However, for transactions with large amount, special attention should be given to the transaction timestamp and deadline parameters, since blocks can become orphaned and transactions cancelled as a result in case their deadline has passed.

> When genesis time + timestamp + deadline * 60 is bigger than transaction.blockTime + 23 hours, a transaction can be accepted when the confirmations count reaches 10.

If (genesis time + transaction.timestamp + transaction.deadline * 60) is smaller than (transaction.blockTimestamp + 23 hours), you should wait until the transaction has 720 confirmations before crediting the user’s account. 720 blocks is the maximum depth a blockchain reorganization can go. By waiting that long, you ensure the transaction is always included. Transactions that only required 10 confirmations will be put back in the blockchain automatically due to their longer deadline time.

> The default deadline in the client is 24 hours, which means that in 99% of the cases only 10 confirmations will be necessary before crediting the user’s account.

> Genesis time for the Burst blockchain is GMT: Monday, 11 August 2014 02:00:00
Unix epoch 1407722400

To identify the user, you must look at the transaction attachment. If a plain text message is included, attachment.message is set and attachment.messageIsText is set to “true” (as string).

If an encrypted message was attached instead, attachment.encryptedMessage should exist instead. This is not just a string, but an object and contains two keys; data and nonce.

### decrypt ###

To decrypt the message use the decryptFrom API.

This API call takes the following parameters:

1. account - account id that sent you the encrypted message
2. data - the encrypted message data extracted from `transaction.attachment.encryptedMessage.data`
3. nonce - the encrypted message nonce extracted from `transaction.attachment.encryptedMessage.nonce`
4. decryptedMessageIsText - set to “true" if the message you’re trying to decrypt is text
5. secretPhrase - passphrase of the account that received the encrypted message i.e. the deposit account

Example: to decrypt the message sent by testnet transaction 12700027308938063138 send the following request parameters:

> http://localhost:6876/burst?requestType=decryptFrom&secretPhrase=[passphrase_from_account_BURST-EVHD-5FLM-3NMQ-G46NR]&account=BURST-XK4R-7VJU-6EQG-7R335&data=9fd7a70625996990a4cf83bf9b1568830f557136044fb3209dd7343eec2ed96ec312457c4840dabaa8cbd8c1e9b8554b&nonce=650ef2a8641c19b9fd90a9ef22a2d50af90aa3b0de3d7a28b5ff2ad193369e7a&decryptedMessageIsText=true

The response is:

    {
    "decryptedMessage": "test message",
    "requestProcessingTime": 2
    }
    
After you have decrypted the message, you can now credit the customer account with the amount specified in transaction.amountNQT.

Note: If you wish you can show pending deposits to the user for transaction which did not yet reach the required number of confirmations.

You could also check for new transactions by specifying the last block’s timestamp+1 as the timestamp parameter: 
> http://localhost:6876/burst?requestType=getAccountTransactions&account=BURST-XK4R-7VJU-6EQG-7R335&type=0&subtype=0&timestamp=83099831

To get the last block timestamp, you would look at the last processed transaction blockTimestamp, or use the getBlockchainStatus API


4.2 Account-Based Deposits
-
As discussed above, Burst accounts are an expensive resource and should not be treated like disposable Bitcoin addresses.

For account-based deposits, you basically generate a new random passphrase for each user. The passphrase should be very strong and at least 35 characters long.

Once you have this passphrase, you can get the account id and public key via the `getAccountId` API call.

1. secretPhrase - account passphrase
2. publicKey - account public

Note that you only need to specify one of the above parameters, not both. So in our case, you just specify the secretPhrase parameter.

> http://localhost:6876/burst?requestType=getAccountId&secretPhrase=1234

The response is:

	{
	"accountRS": "BURST-5WUN-YL5V-K29F-F43EJ",
	"publicKey": "fddcda69eeca58e5d783ad1032d080d2758a4e427881b6a4a6fe43d9e7f4ac34",
	"requestProcessingTime": 2,
	"account": "15577989544718496596"
	}

On your site’s deposit page, you will need to show the account address extracted from the accountRS field i.e. BURST-5WUN-YL5V-K29F-F43EJ.

If the account hasn’t yet had any incoming transactions, you will also need to display the publicKey to the user.

The public key doesn’t have to be displayed any more after it has had it’s first incoming transaction.

When a user sends funds to a new account, it needs to add the public key, so that an announcement of this key can be made. Once done, this is no longer needed. Accounts without a public key are only protected by the 64 bit account address not by the 256 public key.

Tracking New Account-Based Deposits
-

To track new deposits, it’s easiest to simply inspect all transactions in a block to see if any of them are to account addresses you generated. An alternative method would be to use the getBlockchainTransactions API detailed in message-based deposits. (This is to be done in a loop)

Use the getBlockchainStatus API to check if there is a new block. This API call has no parameters.

> http://localhost:6876/burst?requestType=getBlockchainStatus

The response includes lastBlock (block id) and numberOfBlocks (the height).

If numberOfBlocks is different from the previous execution of this API request, one or more new blocks have been generated. The transactions from the block which now has 10 confirmations have to be fetched. You should save in your database the height of the last block you processed.

Use the getBlock API to get the block at the height of 10 blocks ago (10 confirmations).

Pass it the `numberOfBlocks` parameter from the getBlockchainStatus API response after subtracting 11 from this value.

1. height - height of the block, zero-based.
2. includeTransactions - set to true to return the array of transactions included in the block

For each transaction, see if the recipientRS field corresponds to one of the deposit accounts you generated for your users. If so, this is an incoming payment. Credit the user’s internal balance and send the money to your hot wallet.

Similarly to message based deposits, special attention should be given to the transaction timestamp and deadline parameters.

After all transactions of this block have been checked, see if you’ve processed the previous block before or not. If not, traverse through the previous blocks chain until you reach the last processed block.

5 Withdrawing / Sending Money
-
When a user wants to withdraw to a specific account, you ask him for the account id he wants to withdraw to. When you receive this account id, you must first check if that account has a public key attached to it or not (i.e. if it’s new or not).

To do this, you must use the getAccountPublicKey API. It takes 1 parameter, account.

account - account you want the public key of

> http://localhost:6876/burst?requestType=getAccountPublicKey&account=BURST-C6L6-UQ5W-RBJK-AWDSJ

If the account does not have a public key, you will get this error: `{ "errorCode": 5, "errorDescription": "Unknown account" }`
When you get this error, you should also ask the user for his public key or at least display a warning explaining the risk of using an account without a public key.

When you have both the account id and public key, you can verify that they are correct by comparing the given account id with the account id generated by the public key using the getAccountID API call.

1. secretPhrase - account passphrase
2. publicKey - account public key

We want to calculate only by publicKey so our request looks like this:

> http://localhost:6876/burst?requestType=getAccountId&publicKey=28f56a81e0f8555b07eacffd0e697b21cbbbdf3cf620db14522732b763564f13

You’ll get back a response like this: `{"accountRS":"BURST-C6L6-UQ5W-RBJK-AWDSJ","publicKey":"28f56a81e0f8555b07eacffd0e697b21cbbbdf3cf620db14522732b763564f13","requestProcessingTime":0,"account":"9827273118446850628"}`

Now compare the response accountRS to the account id the user provided. If they are equal, you can go ahead and perform the withdrawal.

## Sending Burstcoin ##

Sending Burstcoin is done via the sendMoney API call. The relevant parameters are:

1. recipient - recipient’s account address
2. amountNQT - amount of BURST (in NQT)
3. feeNQT - transaction fee for the transaction in NQT. 
4. secretPhrase - sender’s account passphrase
5. deadline - deadline for the transaction in minutes. Should be set to the maximum value of 1440
6. recipientPublicKey - recipient public key as provided by the user, only needed if the user account has no public key yet (on first transaction)

At the moment (v1.9.2) the recipientPublicKey is optional, however not specifying it, puts the user's funds at risk. The recipientPublicKey is mandatory in case you like to attach an encrypted message to the withdrawal transaction of a new account.

This request has to be sent using HTTP POST.

The response should look like this if everything went OK: `{ "fullHash": “10788f7ad3f145b5209da6145327d7fed869…”, ... a lot more information ... }`

A correctly executed response should always contain the "transaction" field which represents the newly created transaction id.

If there’s an error, you may get a response such as this (other errors may apply):`{ "errorCode": 5, "errorDescription": "Unknown account" }`


5.1 Adding a Message To a (Payment) Transaction
-
You can add messages to any kind of transaction.

To do so, specify the below parameters in your request:

1. message - plain text message.
2. messageIsText - should be set to the string “true” if text.
3. messageToEncrypt - plain text message that should be encrypted.
4. messageToEncryptIsText - should be set to the string "true" if text.

In case you want to attach a plain text message, specify message and set messageIsText to “true”.

### encrypt ###
If you want to attach an encrypted message that can only be read by the recipient, specify `messageToEncrypt` and set `messageToEncryptIsText` to “true”. To create the messge you may use the `encryptTo` API call.

Note that these messages are not mutually exclusive, you can add both a plain text and encrypted message in the same transaction.

Allowing the user to add a message on your withdrawal page is recommended, so that you can coordinate with other services who use a message-based deposit system.

5.2 Hot and Cold Wallets
-
You should not keep all of your user’s deposits in a single hot wallet. A hot wallet is a wallet for which the passphrase is stored somewhere on your server, so that you can send money from it.

Instead, you should have both a hot and cold wallet. The cold wallet should hold most of the coins and not be accessible from any of your servers. Ideally, you’d manually send from your cold wallet to your hot wallet when more coins are needed for day-to-day operations.

So the best thing to do is to have money sent to your cold wallet address, and then send out to your hot wallet manually when needed.


5.3 Additional information
-
5.3.1 Burst Account Format
---
The account ID is stored internally as a 64 bit signed long variable. When used in APIs it is usually returned as both unsigned number represented as string and using alphanumeric Reed-Solomon representation starting with "BURST-" prefix.

For example: BURST-ER8M-SYV3-R7EK-EUF3L

In API request parameters and response JSON, you will find both representations, the numeric representation is typically displayed as account, sender, recipient. The alphanumeric representation is typically displayed as accountRS, senderRS, recipientRS (simply always add “RS”). RS stands for Reed-Solomon. This form of address improves reliability by introducing redundancy that can detect and correct errors when entering and using account ID’s.

5.3.2 Burst and NQT Amounts
-
All amounts should be converted to NQT format to be used in API calls. NQT is the name given to 0.00000001 Burst (or 10^(-8) in mathematical shorthand). The NQT to Burst ratio is equivalent to the Satoshi to Bitcoin ratio. Simply put, 1 Burst is 100000000 NQT, therefore to convert Burst to NQT, simply multiply by 100000000.

5.3.3 Asset and currency QNT amounts
-
Each Asset and Currency (generally referred to as "Holding") has specific number of decimal positions to which this holding is divisible. API requests and responses always expect the holding quantity to be specified as a whole number without decimal positions. We refer to this value as QNT. For example, when transferring 12.34 units of holding XYZ which has 4 decimal positions, specify the QNT value as 123400.

5.3.4 Minimum Fee
-
All outgoing transactions require a fee of at least 735000 NQT at the moment. The minimum value is 735000 NQT. use `suggestFee` API call. for cheap/standard/priority suggestions.


> Content is available under GNU Free Documentation License 1.3 or later unless otherwise noted.
