---
title: Migrate the provider API
description: Migrate to the current provider API.
sidebar_position: 12
---

# Migrate to the current provider API

In January 2021, MetaMask made a number of breaking changes to the
[provider API](../reference/provider-api.md), and removed the injected `window.web3`.
These changes are live on all platforms as of version:

- `9.0.2` of the MetaMask browser extension.
- `1.0.9` of MetaMask Mobile.

This guide describes how to migrate to the new provider API, and how to replace `window.web3`.
To understand why MetaMask made these changes, please see
[this blog post](https://medium.com/metamask/breaking-changes-to-the-metamask-provider-are-here-7b11c9388be9).

:::note
If you're a MetaMask user attempting to use a legacy Ethereum website that hasn't migrated to the
new API, please see the [MetaMask legacy Web3 extension](#use-the-metamask-legacy-web3-extension).

Except for such legacy websites, no action is required for MetaMask users.
:::

## Summary of breaking changes

### window.web3 removal

As part of the breaking changes, MetaMask stopped injecting `web3.js` version `0.20.7` as `window.web3`
into web pages.
MetaMask still injects a dummy object at `window.web3`, in order to issue warnings when websites
attempt to access `window.web3`.

### window.ethereum API changes

MetaMask made the following breaking changes to the `window.ethereum` API:

- Ensure that chain IDs returned by `eth_chainId` are **not** 0-padded.
    - For example, instead of `0x01`, we always return `0x1`, wherever the chain ID is returned or accessible.
- Stop emitting `chainIdChanged`, and instead emit `chainChanged`.
- Remove the following experimental methods:
    - `ethereum._metamask.isEnabled`
    - `ethereum._metamask.isApproved`
- Remove the `ethereum.publicConfigStore` object.
    - This object was, despite its name, never intended for public consumption.
      Its removal _may_ affect those who do not use it directly, for example, if another library you
      use relies on the object.
- Remove the `ethereum.autoRefreshOnNetworkChange` property.
    - Consumers can still set this property on the provider, it just won't do anything.
- Deprecate the `web3.currentProvider` method.
    - Use [@metamask/detect-provider](https://github.com/MetaMask/detect-provider) to detect the
      current provider.

## Replace window.web3

:::caution Pages no longer reload on chain changes
Since we removed `window.web3`, MetaMask no longer automatically reloads the page on chain/network changes.

Please see [Handling the removal of `ethereum.autoRefreshOnNetworkChange`](#handle-the-removal-of-ethereumautorefreshonnetworkchange)
for details.
:::

For historical reasons, MetaMask injected [`web3@0.20.7`](https://github.com/ethereum/web3.js/tree/0.20.7) into all web pages.
That version of `web3` is deprecated, [has known security issues](https://github.com/ethereum/web3.js/issues/3065), and is no longer maintained by the [web3.js](https://github.com/ethereum/web3.js/) team.
Therefore, we decided to remove this library.

If your website relied on our `window.web3` object, you have to migrate.
Please continue reading to understand your options.
Some are as simple as a one-line change.

:::tip
Regardless of how you choose to migrate, you may want to read the `web3@0.20.7` documentation, which
you can find [here](https://github.com/ethereum/web3.js/blob/0.20.7/DOCUMENTATION.md).
:::

### Use window.ethereum directly

For many web3 sites, the API provided by `window.ethereum` is sufficient.
Much of the `web3` API simply maps to RPC methods, all of which can be requested using
[`ethereum.request()`](../reference/provider-api.md#windowethereumrequestargs).
For example, here are a couple of actions performed using first `window.web3`, and then their
equivalents using `window.ethereum`.

```javascript
/**
 * Getting Accounts
 */

// window.web3
const accounts = web3.eth.accounts;

// window.ethereum
const accounts = await ethereum.request({ method: 'eth_accounts' });

/**
 * Sending a Transaction
 */

// window.web3
web3.eth.sendTransaction(
  {
    to: '0x...',
    'from': '0x...',
    value: '0x...',
    // And so on...
  },
  (error, result) => {
    if (error) {
      return console.error(error);
    }
    // Handle the result
    console.log(result);
  }
);

// window.ethereum
try {
  const transactionHash = await ethereum.request({
    method: 'eth_sendTransaction',
    params: [
      {
        to: '0x...',
        'from': '0x...',
        value: '0x...',
        // And so on...
      },
    ],
  });
  // Handle the result
  console.log(transactionHash);
} catch (error) {
  console.error(error);
}
```

### Use an updated convenience library

If you decide that you need a convenience library, you have to convert your usage of `window.web3`
to an updated convenience library.
We recommend [`ethers`](https://npmjs.com/package/ethers) ([documentation](https://docs.ethers.io/)).

### Use @metamask/legacy-web3

:::caution
We strongly recommend that you consider one of the other two migration paths before resorting to this one.
It is not future-proof, and we will not add new features to it.
:::

Finally, if you just want your web3 site to continue to work, we created
[`@metamask/legacy-web3`](https://npmjs.com/package/@metamask/legacy-web3).
This package is a drop-in replacement for our `window.web3` that you can add to your website even
before remove `window.web3` on all platforms.

`@metamask/legacy-web3` should work exactly like our injected `window.web3`, including by refreshing
the page on chain/network changes, but _we cannot guarantee that it works perfectly_.
We will not fix any future incompatibilities between `web3@0.20.7` and MetaMask itself, nor will we
fix any bugs in `web3@0.20.7` itself.

For installation and usage instructions, please see the
[npm listing](https://npmjs.com/package/@metamask/legacy-web3).

### Use the MetaMask legacy Web3 extension

We created the [**MetaMask Legacy Web3 Extension**](https://github.com/MetaMask/legacy-web3-extension)
for any users of websites that still expect `window.web3` to be injected.
If you install this extension alongside the regular MetaMask wallet extension, websites that rely on
our old window.web3 API should start working again.

As with the regular extension, it’s critical that you only install from the official browser
extension stores.
Please follow the relevant link below to install the Legacy Web3 extension in your browser:

- [Edge](https://microsoftedge.microsoft.com/addons/detail/metamask-legacy-web3/obkfjbjkiofoponpkmphnpaaadebfloh?hl=en-US)
- [Firefox](https://addons.mozilla.org/en-US/firefox/addon/metamask-legacy-web3/)

## Migrate to the new provider API

### Handle eth_chainId return values

The `eth_chainId` RPC method now returns correctly formatted values, e.g. `0x1` and `0x2`, instead
of _incorrectly_ formatted values, e.g. `0x01` and `0x02`.
MetaMask's implementation of `eth_chainId` used to return 0-padded values for the
[default Ethereum chains](connect/detect-network.md#chain-ids) _except_ Kovan.
If you expect 0-padded chain ID values from `eth_chainId`, make sure to update your code to expect
the correct format instead.

For more details on chain IDs and how to handle them, see the
[`chainChanged` event](../reference/provider-api.md#chainchanged).

### Handle the removal of chainIdChanged

`chainIdChanged` is a typo of `chainChanged`.
To migrate, simply listen for `chainChanged` instead:

```javascript
// Instead of this:
ethereum.on('chainIdChanged', (chainId) => {
  /* handle the chainId */
});

// Do this:
ethereum.on('chainChanged', (chainId) => {
  /* handle the chainId */
});
```

### Handle the removal of isEnabled() and isApproved()

Before the new provider API shipped, we added the `_metamask.isEnabled` and `_metamask.isApproved`
methods to enable web3 sites to check if they have
[access to the user's accounts](../reference/rpc-api.md#eth_requestaccounts).
`isEnabled` and `isApproved` functioned identically, except that `isApproved` was `async`.
These methods were arguably never that useful, and they became completely redundant with the
introduction of MetaMask's [permission system](../reference/rpc-api.md#restricted-methods).

We recommend that you check for account access in the following ways:

1. You can call the [`wallet_getPermissions`](../reference/rpc-api.md#wallet_getpermissions) RPC method
    and check for the `eth_accounts` permission.
1. You can call the `eth_accounts` RPC method and the
    [`ethereum._metamask.isUnlocked()`](../reference/provider-api.md#windowethereum_metamaskisunlocked) method.
    - MetaMask has to be unlocked before you can access the user's accounts.
      If the array returned by `eth_accounts` is empty, check if MetaMask is locked using `isUnlocked()`.
    - If MetaMask is unlocked and you still aren't receiving any accounts, it's time to request
      accounts using the [`eth_requestAccounts` RPC method](../reference/rpc-api.md#eth_requestaccounts).

### Handle the removal of ethereum.publicConfigStore

How to handle this change depends on if and how you relied on the `publicConfigStore`.
We have seen examples of listening for provider state changes the `publicConfigStore` `data` event,
and accessing the `publicConfigStore` internal state directly.

We recommend that you search your code and its dependencies for references to `publicConfigStore`.
If you find any references, you should understand what it's being used for, and migrate to
[one of the recommended provider APIs](../reference/provider-api.md) instead.
If you don't find any references, you should not be affected by this change.

Although it is possible that your dependencies use the `publicConfigStore`, we have confirmed that
the latest versions (as of January 2021) of the following common libraries were not affected by this
change:

- `ethers`
- `web3` (web3.js)

### Handle the removal of ethereum.autoRefreshOnNetworkChange

The `ethereum.autoRefreshOnNetworkChange` was a mutable boolean property used to control whether
MetaMask reloaded the page on chain/network changes.
However, it only caused the page to be reloaded if the script access a property on `window.web3`.
Therefore, this property was removed along with `window.web3`.

Despite this, we still recommend reloading the page on chain changes.
Some convenience libraries, such as [ethers](https://www.npmjs.com/package/ethers), will continue to
reload the page by default.
If you don't use such a convenience library, you'll have to reload the page manually.
Please see the [`chainChanged` event](../reference/provider-api.md#chainchanged) for details.
