---
description: How to move and ensure safety of your funds when operating on mainnet
---

# Using Golem on Mainnet

Okay, so we've seen Golem requestors hand out tasks to providers and saw them pay those providers for the successfully executed tasks. We've also seen how we could use layer 2 \(zkSync\) to speed up those payments and significantly cut the transactions fees.

Still, in a context of running Golem on the Ethereum mainnet, a few important questions remain largely unanswered:

* how do you **get funds to your requestor** so you can use them to pay for the tasks?
* how do you **get funds out of a Golem node** if you don't need them there anymore?
* how to **secure access to the funds** in your Golem wallet if things go haywire?

## ERC-20 vs Layer2

The most important distinction in basically every piece of payment-related activity, both when you consider transactions between the network's nodes but also when talking about getting funds in and out of Golem is whether you're using the regular ERC-20 token directly on the Ethereum blockchain or whether you're using zkSync.

While direct, on-chain transactions using ERC-20-based tokens have long become the daily bread for the Ethereum mainnet and constitute a significant part of more than a million transactions passing through the chain each day, the recent spike in both the ETH's price and the average gas prices makes it almost completely useless as a means of exchange in Golem where the value paid will usually be orders of magnitude smaller than the fee for the payment itself.

Of course, if you're willing to accept that disproportion, you may continue to use our ERC-20 payment driver but for the majority of Golem node users, zkSync will be the main platform both when paying for tasks and receiving payments for their execution.

## Your Golem wallet

{% hint style="info" %}
All instructions below assume that you have your daemon launched on another terminal. In case your forgot the command to run the daemon, it's:  
  
`yagna service run`

If you get any errors while running the other commands here, first always ensure that your daemon is still running correctly.
{% endhint %}

Golem's wallet is automatically initialized for you the first time you start your `yagna` daemon and thus, an address associated with it is also generated automatically.

Obviously, to have any kind of funds transferred to your Golem's wallet, you'll need its address. You may obtain it using the `id` command:

```text
yagna id show
```

The value described as `nodeId` in the output is the Ethereum address of your Golem node and it's also the address of its wallet. Note it down so you can use it to supply your node with funds.

## Sending funds to your account

### zkSync

So far the only supported way to enable your requestor wallet to operate on zkSync is to reach out to us to get your address funded with some GLM tokens.

Using the address you obtained in the previous sesction, reach out to us on our Discord at [https://chat.golem.network](https://chat.golem.network) where, after a short verification process, you'll be issued some GLM tokens directly to your zkSync wallet and your Golem node will be good to go!

The same instruction, already containing your mainnet adress can be obtained by running:

```text
yagna payment fund --network=mainnet --driver=zksync
```

### ERC-20

On the other hand, if you'd like to use the regular ERC-20 transactions to pay the providers, you'll need to supply your address with some actual GLM tokens, plus some ETH to pay for all the gas fees. Just use you regular wallet to send some GLM and ETH tokens to the node's address.

Again, you'll get the same instruction plus your mainnet address if you just run:

```text
yagna payment fund --network=mainnet --driver=erc20
```

## Enable the mainnet account

In the current version though, the daemon is set-up to use Ethereum's Rinkeby testnet by default.

In order to enable the daemon to use the mainnet, you'll need to tell it so using:

```text
yagna payment init --sender --network==mainnet --driver=zksync
```

or

```text
yagna payment init --sender --network=mainnet --driver=erc20
```

to enable the zkSync or the ERC-20 mainnet drivers respectively. 

{% hint style="warning" %}
Again, unless you have good reasons not to, we recommend using zkSync for much lower transaction fees.
{% endhint %}

{% hint style="danger" %}
The initialization must be performed after every restart of the `yagna` daemon.
{% endhint %}

## Getting your funds out of the Golem node

### zkSync

If you'd like to convert your GLM tokens on zkSync back to regular ERC-20 GLM tokens on the Ethereum blockchain, you need to exit the zkSync Layer2.

There are two ways you may perform this operation. In the first case, you may exit from zkSync to the same address. As a result, your GLM tokens will once more become regular ERC-20 tokens and will stay within the same wallet they were in. To do that, just issue the following command:

```text
yagna payment exit --network=mainnet
```

The alternative option is to provide an address to which the exit should be performed. That way, instead of sending the tokens to the address the request was signed by, zkSync will send them to the address you have provided. This is especially useful if you no longer need to use the GLM tokens within the Golem network and would rather have them back on your regular cryptocurrency wallet that may be completely independent of Golem. The command you need is:

```text
yagna payment exit --to-address=YOUR-ETHEREUM-ADDRESS --network=mainnet
```

{% hint style="danger" %}
Be careful though, as Golem does not perform any validation of the supplied address.
{% endhint %}

### ERC-20

\[ content needed \]

## Backing up your Golem wallet

As you're probably aware, in the Ethereum universe, your funds are only as secure as the private key for the account that holds them. Because of that, to ensure that you don't lose access to your GLM/ETH tokens stored on your Golem account, it's crucial for you to be able to back-up your Golem wallet and store the key in a safe storage, separate from the node itself.

To create a backup of your Golem wallet, export it with:

```text
yagna id export --file-path=./key.json
```

The resultant `key.json` file in the directory you ran the command from will contain the private key for your Golem wallet.

Once the file is created, examine its contents and ensure that the `address` property in it is identical to the address of your node's Ethereum address. Normally there's no reason for this field to differ - but treat this step as a redundant check to ensure that you have backed up what you intended to back-up.

{% hint style="danger" %}
Ensure you store that key file in a safe place. In case your Golem wallet gets corrupted or lost, if you don't have the backup, your funds will be lost forever.

Likewise, it's probably a good idea to encrypt the keystore file so that someone who'd take hold of the file wouldn't be able to take control of your funds.
{% endhint %}

## Restoring your backed-up Golem wallet

If, for whatever reason, your Golem wallet is destroyed or corrupted e.g. you moved on to a new machine and forgot to take move Golem's installation with it, you'll be faced with the necessity to recover your wallet from your previously backed-up keystore file.

To restore your wallet, first start with a fresh yagna install:

```text
curl -sSf https://join.golem.network/as-requestor | bash -
```

Once yagna is installed, run it with:

```text
yagna service run
```

Now, as usual, leave the daemon running in the background and proceed with the rest of the process in another terminal window.

### Retrieve your keystore

Here you'll need the key.json file you had previously backed up. Do whatever you need to restore it - e.g. decrypt it if you previously encrypted it. For the process to work, it must be the same plain-text json file that yagna originally exported.

Be sure that your `key.json` file is in your current working directory and run:

```text
yagna id create --from-keystore ./key.json
```

This should create a new identity in yagna based on your backed-up wallet.

### Set the new identity as yagna's default

**1.** Using the Ethereum address of your backed-up wallet, run:

```text
yagna id update --set-default 0x-the-address
```

**2. STOP YOUR YAGNA DAEMON**

\(just press Ctrl-C in the console that's running `yagna service run` and wait for the daemon to exit\)

**3.** Remove `yagna`'s accounts configuration file

{% tabs %}
{% tab title="Ubuntu" %}
```text
rm $HOME/.local/share/yagna/accounts.json
```
{% endtab %}

{% tab title="mac OS X" %}
```text
rm $HOME/Library/Application\ Support/GolemFactory.yagna/accounts.json
```
{% endtab %}

{% tab title="Windows" %}
```
del %APPDATA%\GolemFactory\yagna\accounts.json
```
{% endtab %}
{% endtabs %}

4. **Start your yagna daemon again** \(as usual, do it in a separate command line terminal and allow it to run in the background\)

```text
yagna service run
```

**5.** Ensure yagna is using your newly-restored wallet

```text
yagna id show
```

The `nodeId` property should display the Ethereum address of your backed-up wallet.
