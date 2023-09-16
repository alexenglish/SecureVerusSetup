# Generating Addresses/Keys
For best security and safety against loss, it's recommended to generate keys in a fashion other than in the wallet file of an online daemon. Here are some options for ways to generate keys safely.

One important note - this depends heavily on calls to the verus RPC that include seed phrase word lists. You don't want these getting logged in your BASH (or other shell) history. Here is a way to temporarily turn off Bash history, if you use another shell you'll need to look up what to do there.

```
set +o history # temporarily turn off history

# commands here won't be saved

set -o history # turn it back on
```
## Converting Seed Phrases
If you have a seed phrase and need to find the address, public key, or the private key (to import it), you can use convertpassphrase on the Verus RPC.

Enclose your word list in quotes and separate the words with single spaces.

That looks like this:
```
> verus convertpassphrase "YOUR WORD LIST HERE"
{
  "walletpassphrase": "YOUR WORD LIST HERE",
  "address": "RXSAPbj9jrk26d2ooY3GTAchr932Ak7Lgn",
  "pubkey": "033008efe894f4d555e153d2b2d27f4c41f2d43534f18bf3136861ff675995f997",
  "privkey": "2826b46b68ffc68ff99b453c1d30413413422d706483bfa0f98a5e886266e76e",
  "wif": "UqMbK4NWZvAvFmbCz2ZsqZSXJqdYL1rcaDkmbkcjnWHm5KvcMoJ7"
}
```

## Generating Seed Phrases
### Verus Mobile
Create an account on Verus Mobile and save the seed phrase and receive address. If you need to get a WIF private key or public key you can use the Verus RPC command convertpassphrase.

### Verus Desktop
Same as with Verus Mobile - create a profile, add a coin, select Verus and pick Lite Mode then record the seed phrase it gives you and complete the steps to find the receive address. Like with mobile, if you need a public key or later need to import it to a native wallet you can use convertpassphrase.

### CLI BIP39 Seed Phrase Generator
In my Verus Extras repo at https://github.com/alexenglish/VerusExtras there is a script called bip39gen.sh that will generate seed phrases using the BIP39 word list. You can clone the repo and run the script without any setup.

The script will prompt for some data entry to increase entropy, which is recommended for single-board computers or other low-entropy systems.

Use this to generate a word list, then to import when needed, or to get the associated address or public key use convertpassphrase.

## Paper Wallet
You can also use the Verus paper wallet generator here: https://paperwallet.verus.io/

DO NOT USE IT DIRECTLY FROM THE WEBSITE OR ON AN INTERNET-CONNECTED SYSTEM

Follow the instructions on the page to save the page, then use it offline to generate your paper wallet.

This will produce an address and private key, but will not provide a seed phrase for storage that way, nor a public key should you need it. If you need to get a public key for an address created this way you'll need to import the private key to a wallet, then use the Verus RPC command `validateaddress` to get that information.

## Offline verusd
Lastly, you can also put verusd on an offline computer and export the wallet file it generates on startup. You'll have to make sure the system also has a copy of the zcash parameters on it, either by using the fetch-params script bundled with the CLI release before taking the system offline, or by starting up verusd, letting it download that data, then stopping verusd and discarding the wallet file it generated initially.

You can export using `verus z_exportwallet myexport` to create an export file named myexport. This will contain the addresses and private keys for 100 addresses (by default). If you need public keys for any of the addresses you can use the `validateaddress` command to get them.

# Storing Keys
These are the recommended storage methods for a private key or seed phrase:
* Stamping into metal plates and keeping them in a highly secure location
* Paper wallets stored in very safe, secure places - the bar is higher for the quality of storage for paper because of its succeptibility to water damage or burning
* Encrypted storage of the digital key data on MULTIPLE devices in multiple places, coupled with a robust and secure way to remember the password for the encrypted storage
