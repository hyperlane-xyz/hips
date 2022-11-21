# HIP 2 - On-chain fee quoting

| hip | title | status | author | created |
| --- | --- | --- | --- | --- |
| 3 | On-chain fee quoting | Draft | tkporter | 2022-11-11 |

### **Brief Summary / Abstract**

To provide a better experience for Hyperlane users, an interface for gas payments is proposed that accommodates on-chain fee quoting.

### **Motivation**

For a message to be processed, a transaction must be made to the destination chain. Most Hyperlane user’s don’t want to be responsible for the infrastructure or manual effort required in submitting this transaction themselves, and prefer for a general purpose relayer to do this on their behalf. To ensure a relayer is properly compensated for transaction fees and cannot be griefed, messages must pay the relayer.

Today, an application hoping to calculate the appropriate payment for a message does so entirely off-chain using the Hyperlane SDK. When dispatching the message on the origin chain, some origin chain native tokens are paid to the relayer by calling the payable function `InterchainGasPaymaster.payGasFor`. The relayer observes the `GasPayment` event emitted by that function, and builds a database of payments for each message. The relayer will then continually re-evaluate whether messages are eligible for processing, which occurs if the USD value of the total payment for a message in origin chain native tokens exceeds the USD value of the destination transaction fees. In other words, there’s a social contract between message senders and the relayer that the relayer will submit the process transaction to the destination if it has been properly compensated.

The off-chain gas payment estimation, while nice for keeping as much data and calculation off chain to limit costs, has a number of downsides:

- It can be unclear if a message being dispatched will end up being processed by the relayer at its destination.
- Origin/destination chain native token exchange rates or destination chain gas prices can change, causing the USD value of the payment to be insufficient by the time the relayer evaluates the message for processing. A top-up payment would then be required, which is a poor user experience.
- It’s not compatible with situations that involve an extended period between off-chain gas payment estimation and message dispatching, for example messages dispatched by governance, multisig, or timelock contracts. Because top-up payments can be made by any address, these types of messages would likely require a separate payment, which is a poor user experience.
- There are no refunds for overpayments. Because all data and calculation happens off chain, when a payment is made on the origin chain there is no way to determine if a message has been overpaid for. After a message has been relayed, the relayer could potentially refund overpayments, but this incurs a transaction fee. It’s also not very clear which address should receive the refund.
- It’s hard for applications to pay for message fees on their user’s behalf. Some contracts are willing to keep a native token balance that they’ll use to pay for message fees. However if the fee amount is calculated off-chain and passed into the contract by a user, there’s nothing stopping a user from specifying a very high fee and depleting the contract’s balance. This approach of contracts paying for fees on a user’s behalf is one way of supporting callback messages.
- Developers are required to use the Hyperlane SDK to estimate correct payments.
- Generally, it’s a rough user experience. Application developers have an easier time reasoning about and dealing with on-chain data, and want to have certainty that messages will be processed. Requiring off-chain fee calculation limits the composability of Hyperlane. Sending a message should “just work.”

### **Tech Spec**

Paying for fees can be done by paying an `InterchainGasPaymaster` contract relating to an off-chain relayer that will enter a social contract to process a message.

At a high level, the `InterchainGasPaymaster` contract should accept the message ID of a Hyperlane message that has been dispatched, information about how much gas the relayer should use when processing the message, and payment in the form of native tokens.

Below is the proposed generic interface for an `InterchainGasPaymaster`. Note that implementations of an IGP may place differring semantic meaning to each of the parameters.

```solidity
interface IInterchainGasPaymaster {
    /**
     * @notice Accepts native tokens to pay for the destination gas cost of a
     *   message. Implementations may or may not require specific payment amounts.
     * @param _messageId The message ID.
     * @param _destinationDomain The destination domain of the message.
     * @param _gas An amount of gas.
     * @param _refundAddress The address to refund any overpayment to.
     */
    function payGas(
        bytes32 _messageId,
        uint32 _destinationDomain,
        uint256 _gas,
        address _refundAddress
    ) external payable;
}
```

An IGP that quotes and requires on-chain payment for a total amount of gas that is to be used by the ISM and recipient `handle` would look like:

```
contract FeeQuotingIgp {
    /**
     * @notice Emitted when a gas payment is made.
     * @param messageId The message ID.
     * @param amount The amount of native tokens paid.
     * @param gas The amount of destination gas paid for.
     */
    event GasPayment(
        bytes32 indexed messageId,
        uint256 amount,
        uint256 gas
    );

    /**
     * @notice Accepts payment for a message, reverting if not enough payment was provided.
     *   The `_ismAndHandleGas` should cover ISM metadata calldata gas, ISM.verify gas,
     *   and gas required by the recipient's `handle` function.
     *   The gas required for other operations involving the processing of the
     *   message is read from storage and is added to `_ismAndHandleGas` to calculate
     *   the total estimated amount of destination gas that's required.
     *   The stored value includes intrinsic gas, approximated message calldata
     *   gas, and any overhead gas incurred by the Mailbox contract.
     *   Using the total expected destination gas usage, the destination gas price
     *   (from an oracle), and the destination chain native token quoted in the origin
     *   chain native token (from an oracle), the required fee payment is calculated
     *   and required to be at least the msg.value. Any overpayment is sent back
     *   to the `_refundAddress`.
     *   Emits the `GasPayment` event with the total amount of gas paid for.
     * @param _messageId The message ID.
     * @param _destinationDomain The destination domain of the message.
     * @param _ismAndHandleGas The amount of gas to cover ISM metadata calldata,
     *   ISM.verify gas, and gas required by the recipient's `handle` function.
     * @param _refundAddress The address to refund any msg.value overpayment to.
     */
    function payGas(
        bytes32 _messageId,
        uint32 _destinationDomain,
        uint256 _ismAndHandleGas,
        address _refundAddress
    ) external payable;
}
```

Hyperlane users are likely to be confident in the amount of gas that their recipient's `handle` function will use on the destination, but they likely don't have the same confidence in the amount of gas they need for their ISM. Users may be using the default ISM which could change entirely, or their ISM may undergo parameter changes that could change the amount of gas required. To provide a better experience, ISM-specific IGPs can be provided that allow users to supply the amount of gas their recipient's `handle` function consumes, and the ISM-specific costs are read from storage and accurately updated by the IGP owner.

```
contract IsmFeeQuotingIgp {
    // An implementation of the `FeeQuotingIgp` provided above.
    IInterchainGasPaymaster feeQuotingIgp;

    // Destination domain => gas usage of the ISM.
    mapping(uint32 => uint256) ismDestinationGas;

    /**
     * @notice Accepts payment for a message, reverting if not enough payment was provided.
     *   The `_handleGas` should cover the gas required by the recipient's `handle` function.
     *   The gas required for ISM costs, including ISM metadata calldata and ISM.verify gas,
     *   is read from storage and is added to `_handleGas` before being passed to the feeQuotingIgp.
     * @param _messageId The message ID.
     * @param _destinationDomain The destination domain of the message.
     * @param _ismAndHandleGas The amount of gas to cover ISM metadata calldata,
     *   ISM.verify gas, and gas required by the recipient's `handle` function.
     * @param _refundAddress The address to refund any msg.value overpayment to.
     */
    function payGas(
        bytes32 _messageId,
        uint32 _destinationDomain,
        uint256 _handleGas,
        address _refundAddress
    ) external payable {
        uint256 _ismAndHandleGas = ismDestinationGas[_destinationGas] + _handleGas;

        feeQuotingIgp.payGasFor(
            _messageId,
            _destinationDomain,
            _ismAndHandleGas,
            _refundAddress
        );
    }

    function setIsmDestinationGas(uint32 _destinationDomain, uint256 _ismGas) external onlyOwner;
}
```

The off-chain relayer relating to a `FeeQuotingIgp` will index `GasPayment` events and relate them to dispatched messages based off their message ID. Because the `FeeQuotingIgp` will revert if there was insufficient payment, all `GasPayment` events relate to messages that have fully paid for an amount of gas, and the relayer will honor these payments.

Note that calldata costs for processing the message are not explicitly calculated by the fee quoting IGP. The maximum Hyperlane message size is 2 KiB, or 2048 bytes. Currently, the Ethereum calldata gas cost is 16 gas per non-zero byte, which comes out to a maximum of `32,768` gas. To avoid an additional required mapping in the InterchainGasCalculator of remote domain to the cost per byte of calldata, a flat cost of 32,000 gas to cover potential calldata costs is charged.

### **Rationale**

#### Default vs non-default ISM distinction

#### ISM-specific fee quoting IGPs

On the origin chain, an IGP cannot know what the length of an arbitrary ISM’s metadata calldata is, nor can it know the expected gas used by an arbitrary ISM’s `verify` function. Therefore, the IGP must get this information from some source.

Hyperlane users sending messages that use the default ISM should not need to worry about the destination gas costs incurred by the default ISM. This includes the metadata calldata and the `verify` function. Because the default ISM is out of user control and the parameters of the default ISM may change, a straightforward “happy path” for default ISM users is provided via the default ISM's IGP.

Any non-default ISMs may also have their own ISM-specific IGPs created. Alternatively, users can always call the fee quoting IGP directly with their own `_ismAndHandleGas` amount.

One initial idea was to provide the ISM-specific destination gas costs on the origin chain by having a view function on the local ISM:

```
function destinationGas(uint32 _destinationGas) external returns (uint256);
```

However, this conflicts with permissionless deployment of Hyperlane onto long-tail chains. A relayer can decide to offer its relaying services to process messages to long tail chains if they so choose, and it's more natural to put the onus on the relayer to specify what those costs are. 

#### Calldata gas

The gas used by the message’s bytes in the calldata of a process transaction are not explicitly calculated by the InterchainGasPaymaster because the upper limit of the gas cost is low, and performing this calculation on-chain introduces additional storage reads and computational complexity. Additionally, providing the InterchainGasPaymaster with the exact size in bytes of the Hyperlane message isn't straightforward.

#### Flexibility

The `IInterchainGasPaymaster` interface is generic enough for an implementing contract to choose to quote gas fees on-chain, or to accept arbitrary payment from off-chain calculations.

Continuing to keep gas payments as a concept built on top of the core Hyperlane Mailbox preserves flexibility, modularity, and keeping the Mailbox as concise as possible.

#### Processing reverting messages and risk

When fee quoting occurs on-chain, the relayer commits to process the message based off token exchange rates and gas prices at the time the message is dispatched, but by the time the message is eligible for processing, token exchange rates and gas prices may be wildly different.

This message processing latency can be due to the time to finality for the origin chain of a message. The on-chain token exchange rate or gas price used by the InterchainGasPaymaster can be set to be above the market rate to compensate the relayer for the risk of the prices moving out of favor.

If the relayer’s policy is to only submit a non-reverting transaction attempting to process a message, an attack vector arises. A malicious Hyperlane user could wait until a point when token exchange rates and destination chain gas prices are extremely low, and dispatch and pay for many messages. The recipient of the messages could be configured to revert when processed on the destination. The attacker could then wait until token exchange rates or gas prices are extremely high, and then configure the recipient to no longer revert, causing the relayer to process messages at an extreme loss.

If the relayer’s policy were to always submit process transactions even if they revert, this would eliminate the attack vector and could also provide users with a reverted transaction that can be useful for debugging. However, this could break a variety of use-cases— some Hyperlane use-cases could involve an ISM with an optimistic timeout. Such an ISM would cause message processing to revert before the optimistic timeout had passed. Another use case could involve sending tokens to a recipient contract via a non-Hyperlane bridge that cannot be composed with Hyperlane messages atomically. In this case, the tokens would be required to arrive at the recipient contract prior to the Hyperlane message being able to be successfully processed.

To diminish the aforementioned attack vector while still maintaining the flexibility provided by not attempting to process reverting messages, the relayer could impose self-protective measures that still provide users with high confidence their transaction will be relayed. These measures could be that the relayer will only attempt to process messages that are at most X days old (limiting the likelihood of dramatic token exchange rate or gas price movements), or that the cost of the relayer’s process transaction must always be within a generous multiplier of what was paid.

#### Operations

Two concerning features of on-chain fee quoting are:

- The dollar costs of the transaction fees required to keep token exchange rates and gas prices accurate
- Operational cost and developer effort

On-chain fee quoting requires every remote chain’s token exchange rate and gas price to be available on each local chain. Ethereum tends to be the only chain in which gas costs are expensive enough for consistently writing these via an oracle to be concerning. Because costs on all other chains tend to be low and inaccurately pricing cheap chains is not very critical, price updates made to the Ethereum chain for remote chain prices can be done relatively infrequently. For non-Ethereum chains, transaction costs tend to be lower so price updates can be made more frequently.

In an effort to understand how expensive this will be and how often token exchange rates and gas prices should be updated, some Dune analytics dashboards were created to understand LayerZero’s oracle on Ethereum for on-chain fee quoting. This oracle writes the `remote native token / origin native token`  exchange rate and the remote chain’s gas price to a single storage slot — each value is a `uint128`, so with storage packing only a single storage slot is used.

See [this dashboard](https://dune.com/trevor_hyperlane/layerzero-relayer-oracle) to see all costs as well as detailed graphs for Avalanche, Binance Smart Chain, Polygon, and Arbitrum. See [this dashboard](https://dune.com/trevor_hyperlane/layerzero-ethereum-relayer-oracle-op-ftm-moonbeam) for detailed graphs for  Optimism, Fantom, and Ethereum.

Some notable learnings are:

- From March 15 (the day in which the LZ oracle started making updates) to November 8, about 241 days, 68.31 ETH has been spent on transaction fees. Using the median ETH/USD price from each day, that’s about $131,000 in transaction fees.
- On Ethereum, the oracle updates exchange rates and gas prices for the Ethereum remote chain. These seemingly unnecessary updates comprise of nearly 1/3 of all updates.
- The on-chain `remote native token / origin native token` exchange rate tends to be a configured a percentage higher than the real market price. This has been configured at various periods of time as 30%, 10%, and 5% higher.
- Gas prices are significantly more volatile than token exchange rates. Chains that tend to not have extremely volatile gas prices tend to not require frequent updates.
- The % change between two consecutive posted gas prices on chain tends to be ≥ 10%. They seem to be configured differently for some chains.

How could dollar costs be lowered?

- Many price feeds already exist that could help for getting the current `remote native token / origin native token` price. However, because there typically aren’t gas price oracles and the gas price tends to be more volatile than token exchange rates, gas prices would still need need to consistently be written to storage. If the token exchange rate and gas price fit into a single storage slot, both both the token exchange rate and gas price can be updated for about the same gas costs. This has the additional benefit of not requiring external calls to read token exchange rates, which have a gas overhead.
- Where possible, it could be considered to rely upon the existing token exchange rates and gas prices already present on chain, for example in the LayerZero contracts. This would save costs incurred by an oracle, but has some downsides:
    - There would be a reliance upon a service without any SLA.
    - LayerZero has swapped out the relayer contract that stores the exchange rates and gas prices once in the past. Changes to this contract would need to be observed.
- The oracle agent can be configured to report exchange rates and prices well above the “real” values, and only have generous deviation thresholds. Message processing costs do not seem to be a dealbreaker for Hyperlane users.

### **Backwards Compatibility**

This is not backward compatible with the existing InterchainGasPaymaster used in V1 of the Hyperlane protocol. Hyperlane users moving to V2 hoping to make use of the general purpose relayer to send transactions would be required to move to using the new InterchainGasPaymaster interface.

### **Security Considerations**

The relayer entering a commitment to process a message that may only be processable in the future requires the relayer to take on risk of prices moving out of its favor. The relayer should consider reasonable self-protective policies that balance user experience and cost.

Hyperlane users should be aware of that calling into an InterchainGasPaymaster contract may present a vector for re-entrancy if the InterchainGasPaymaster is not trusted.
