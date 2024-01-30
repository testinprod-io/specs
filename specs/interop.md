<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Interoperability](#interoperability)
  - [Overview](#overview)
  - [The Dependency Set](#the-dependency-set)
    - [Chain ID](#chain-id)
  - [Interop Network Upgrade](#interop-network-upgrade)
  - [Derivation Pipeline](#derivation-pipeline)
    - [Depositing a Relaying Message](#depositing-a-relaying-message)
    - [Safety](#safety)
  - [State Transition Function](#state-transition-function)
  - [Predeploys](#predeploys)
    - [CrossL2Outbox](#crossl2outbox)
      - [Initiating Messages](#initiating-messages)
    - [CrossL2Inbox](#crossl2inbox)
      - [Invariants](#invariants)
        - [Only EOA Invariant](#only-eoa-invariant)
      - [Relaying Messages](#relaying-messages)
    - [L2ToL2CrossDomainMessenger](#l2tol2crossdomainmessenger)
      - [Transferring Ether in a Cross Chain Message](#transferring-ether-in-a-cross-chain-message)
      - [Interfaces](#interfaces)
        - [Sending Messages](#sending-messages)
        - [Relaying Messages](#relaying-messages-1)
    - [L1Block](#l1block)
  - [SystemConfig](#systemconfig)
    - [`DEPENDENCY_SET` UpdateType](#dependency_set-updatetype)
  - [L1Attributes](#l1attributes)
  - [Fault Proof](#fault-proof)
  - [Sequencer Policy](#sequencer-policy)
  - [Block Building](#block-building)
  - [Sponsorship](#sponsorship)
  - [Security Considerations](#security-considerations)
    - [Forced Inclusion of Cross Chain Messages](#forced-inclusion-of-cross-chain-messages)
    - [Cross Chain Message Latency](#cross-chain-message-latency)
    - [Dynamic Size of L1 Attributes Transaction](#dynamic-size-of-l1-attributes-transaction)
    - [Maximum Size of the Dependency Set](#maximum-size-of-the-dependency-set)
    - [Mempool](#mempool)
    - [Reliance on History](#reliance-on-history)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Interoperability

The ability for a blockchain to easily read the state of another blockchain is called interoperability.
Low latency interoperability allows for horizontally scalable blockchains, a key feature of the superchain.

Note: this document references an "interop network upgrade" as a temporary name. The actual name of the
network upgrade will be included in this document in the future.

## Overview

| Term          | Definition                                               |
|---------------|----------------------------------------------------------|
| Source Chain   | A blockchain that includes an initiating message |
| Destination Chain  | A blockchain that includes a relaying message |
| Initiating Message | A transaction submitted to a source chain that authorizes execution on a destination chain |
| Relaying Message | A transaction submitted to a destination chain that corresponds to an initiating message |
| Cross Chain Message | The cumulative execution and side effects of the initiating message and relaying message |
| Dependency Set | The set of chains that originate initiating transactions where the relaying transactions are valid |

A total of two transactions are required to complete a cross chain message. The first transaction is submitted
to the source chain which authorizes execution on a destination chain. The second transaction is submitted to a
destination chain, where the block builder SHOULD only include it if they are certain that the first transaction was
included in the source chain.

The term "block builder" is used interchangeably with the term "sequencer" for the purposes of this document but they need not be the same
entity in practice.

## The Dependency Set

The dependency set defines the set of chains that a destination chains allows as source chains. Another way of
saying it is that the dependency set defines the set of initiating messages that are valid for a relaying
message to be included. A relaying message MUST have an initiating message that is included in a chain
in the dependency set.

The dependency set is defined by chain id. Since it is impossible to enforce uniqueness of chain ids,
social consensus MUST be used to determine the chain that represents the canonical chain id. This
particularly impacts the block builder as they SHOULD use the chain id to assist in validation
of relaying messages.

### Chain ID

The concept of a chain id was introduced in [EIP-155](https://eips.ethereum.org/EIPS/eip-155) to prevent
replay attacks between chains. This EIP does not specify the max size of a chain id, although
[EIP-2294](https://eips.ethereum.org/EIPS/eip-2294) attempts to add a maximum size. Since this EIP is
stagnant, all representations of chain ids MUST be the `uint256` type.

## Interop Network Upgrade

The interop network upgrade timestamp defines the timestamp at which all functionality in this document is considered
the consensus rules for an OP Stack based network. On the interop network upgrade block, a set of deposit transaction
based upgrade transactions are deterministically generated by the derivation pipeline in the following order:

- L1 Attributes Transaction calling `setL1BlockValuesEcotone`
- User deposits from L1
- Network Upgrade Transactions
  - L1Block deployment
  - CrossL2Inbox deployment
  - CrossL2Output deployment
  - L2ToL2CrossDomainMessenger deployment
  - Update L1Block Proxy ERC-1967 Implementation Slot
  - Update CrossL2Inbox Proxy ERC-1967 Implementation Slot
  - Update CrossL2Output Proxy ERC-1967 Implementation Slot
  - Update L2ToL2CrossDomainMessenger Proxy ERC-1967 Implementation Slot
  - Mint Cross Chain Ether Liquidity in L2ToL2CrossDomainMessenger

The execution payload MUST set `noTxPool` to `false` for this block.

The exact definitions for these upgrade transactions are still to be defined.

## Derivation Pipeline

The derivation pipeline enforces invariants on safe blocks that include relaying transactions.

- The relaying message MUST have a corresponding initiating message
- The initiating message that corresponds to a relaying message MUST come from an allowed chain (dependency set)

Blocks that contain transactions that relay cross domain messages to the destination chain where the
initiating transaction does not exist MUST be considered invalid and MUST not be allowed by the
derivation pipeline to be considered safe.

There is no concept of replay protection within the protocol since there is no replay protection
mechanism that fits well for all applications. Users MAY submit an arbitrary number of relaying
messages per initiating message. This greatly simplifies the protocol and allows for
applications to build application specific replay protection.

### Depositing a Relaying Message

Deposit transactions (force inclusion transactions) give censorship resistance to layer two networks.
The derivation pipeline must gracefully handle the case in which a user uses a deposit transaction to
relay a cross chain message. To not couple preconfirmation security to consensus, deposit transactions
that relay cross chain messages MUST have an initiating message that is considered safe. This relaxes
a strict synchrony assumption on the sequencer that it MUST have all unsafe blocks of destination chains
as fast as possible to ensure that it is building correct blocks. It also prevents a class of attacks
where the user can send a relaying message before the initiating message and trick the derivation pipeline
into reorganizing the sequencer.

### Safety

The initiating messages for all relaying messages MUST be resolved as safe before an L2 block can transition from being unsafe to safe.
Users MAY optimistically accept unsafe blocks without any verification of the relaying messages. They SHOULD optimistically verify
the initiating messages exist in destination unsafe blocks to more quickly reorganize out invalid blocks.

## State Transition Function

After the full execution of a block, the set of `MessagePassed` events emitted from the `CrossL2Outbox` are collected and merklized.
This commitment MUST be included as the block header's [extra data field][block-extra-data]. The events are serialized with using
[Simple Serialize][ssz] aka SSZ.

[block-extra-data]: (https://github.com/ethereum/execution-specs/blob/1fed0c0074f9d6aab3861057e1924411948dc50b/src/ethereum/frontier/fork_types.py#L115)
[ssz]: (https://github.com/ethereum/consensus-specs/blob/dev/ssz/simple-serialize.md)

| Name          | Value |
|---------------|-------------------------------------|
| `MAX_LOG_DATA_SIZE` | `uint64(2**24)` or 16,777,216 |
| `MAX_MESSAGES_PER_BLOCK` | `uint64(2**20)` or 1,048,576 |

```python
class CrossChainMessage(Container):
    nonce: uint256
    sender: ExecutionAddress
    target: ExecutionAddress
    value: uint256
    timestamp: uint256
    chainid: uint256
    data: ByteList[MAX_LOG_DATA_SIZE]

messages = List[CrossChainMessage, MAX_MESSAGES_PER_BLOCK](
  message_0, message_1, message_2, ...)

block.extra_data = messages.hash_tree_root()
```

## Predeploys

Three new system level predeploys are introduced for managing cross chain messaging along with
an update to the `L1Block` contract with additional functionality.

### CrossL2Outbox

Address: `0x4200000000000000000000000000000000000023`

The `CrossL2Outbox` is responsible for initiating a cross chain message on the source chain.
For every cross chain message that is sent, it MUST emit an event that indicates the existence
of the cross chain message.

The event SHOULD make it as cheap as possible for the destination chain to look up the existence
of the initiating transaction when verifying the relaying transaction.

#### Initiating Messages

The following interface is used to initiate a message:

```solidity
function initiateMessage(address _target, bytes memory _message) external payable {
    CrossChainMessage memory msg = CrossChainMessage({
      nonce: nonce,
      sender: msg.sender,
      target: _target,
      timestamp: block.timestamp,
      chainid: block.chainid,
      data: _message
    });

    bytes32 messageHash = hashCrossChainMessage(msg);

    emit MessagePassed(...);

    nonce++;
}
```

An initiating message emits the following event:

```solidity
event MessagePassed(
    bytes32 indexed messageHash,
    address indexed sender,
    address indexed target,
    uint256 nonce,
    uint256 timestamp,
    uint256 chainid,
    bytes data
);
```

The chain id is of the source chain. Note that the chain id of the destination chain is not specified
as part of the cross chain message. This means that the cross chain message is valid to be played
on any destination chain. Binding to a particular destination chain can be implemented at the application layer.

The `messageHash` and chain id uniquely identify the existence of an initiating message and SHOULD
be used by a block builder to ensure the existence of the initiating message when verifying the relaying
transaction. `messageHash` SHOULD be `indexed` to allow for easy filtering of logs.

A cross chain message is defined in the following format:

```solidity
struct CrossChainMessage {
    uint256 nonce;
    address sender;
    address target;
    uint256 timestamp;
    uint256 chainid;
    bytes data;
}
```

Note that there is no `gasLimit` to avoid complexity with EVM gas introspection. The `nonce` is unique per `CrossChainMessage`
to guarantee that there is at most 1 unique `messageHash` per `CrossChainMessage`. There is no `value` field because value
is handled by the `L2ToL2CrossDomainMessenger` built on top.

The `messageHash` is constructed like so:

```solidity
function hashCrossChainMessage(CrossChainMessage memory _msg) pure returns (bytes32) {
    return keccak256(abi.encode(_msg.nonce, _msg.sender, _msg.target, _msg.timestamp, _msg.chainid, _msg.data));
}
```

### CrossL2Inbox

Address: `0x4200000000000000000000000000000000000022`

The `CrossL2Inbox` is responsible for relaying a cross chain message on the destination chain.
It is permissionless to finalize a cross chain message on behalf of any user. Certain protocol
enforced invariants must be preserved to ensure safety of the protocol.

#### Invariants

- The timestamp of relaying message MUST be less than or equal to the timestamp of the initiating message
- The chain id of the initiating message MUST be in the dependency set
- The relaying message MUST be initiated by an externally owned account such that the top level EVM call frame enters the `CrossL2Inbox`
- The chain id of the initiating message MUST not be the chain id of the relaying message

##### Only EOA Invariant

The `onlyEOA` invariant on relaying a cross chain message enables static analysis on relaying messages.
This allows for the derivation pipeline and block builders to reject relaying messages that do not
have a corresponding initiating message.

#### Relaying Messages

All of the information required to satisfy the invariants MUST be included to or committed to in the calldata
of the function that is used to execute relaying messages. Both the block builder and the smart contract use
this information to ensure that all system invariants are held.

```solidity
function relayMessage(bytes32 _hash, CrossChainMessage memory _msg) public {
    require(msg.sender == tx.origin);
    require(_hash == hashCrossChainMessage(_msg));
    require(_msg.timestamp <= block.timestamp);
    require(_msg.chainid != block.chainid);
    require(L1Block.isInInteropSet(_msg.chainid));

    bool success = SafeCall.call({
      _target: _msg.target,
      _calldata: _msg.data
    });
}
```

By including the hash in the calldata, it makes static analysis much easier for block builders.
Without the check onchain, the block builder would need to hash the `CrossChainMessage` to build the
`eth_getLogs` query. With the hash onchain, the block builder can directly trust the hash.

### L2ToL2CrossDomainMessenger

Address: `0x4200000000000000000000000000000000000024`

The `L2ToL2CrossDomainMessenger` is a higher level abstraction on top of the `CrossL2Outbox` that
provides features necessary for secure transfers of `ether` and ERC20 tokens between L2 chains.
Messages sent through the `L2ToL2CrossDomainMessenger` on the source chain receive both replay protection
as well as domain binding, ie the relaying transaction can only be valid on a single chain.

#### Transferring Ether in a Cross Chain Message

The `L2ToL2CrossDomainMessenger` MUST be initially set in state with an ether balance of `type(uint248).max`.
This initial balance exists to provide liquidity for cross chain transfers. It is large enough to always have
ether present to dispurse while still being able to accept inbound transfers of ether without overflowing.
The `L2ToL2CrossDomainMessenger` MUST only transfer out ether if the caller is the `L2ToL2CrossDomainMessenger`
from a source chain.

The `L2CrossDomainMessenger` is not updated to include the `L2ToL2CrossDomainMessenger` functionality because
there is no need to introduce complexity with L1 to L2 messages and the L2 to L2 liquidity.

#### Interfaces

The `L2ToL2CrossDomainMessenger` uses a similar interface to the `L2CrossDomainMessenger` but
the `_minGasLimit` is removed to prevent complexity around EVM gas introspection and the `_destination`
chain is included instead.

##### Sending Messages

```solidity
function sendMessage(uint256 _destination, address _target, bytes calldata _message) external payable {
    bytes memory data = abi.encodeCall(L2ToL2CrossDomainMessenger.relayMessage, (_destination, nonce, msg.sender, _target, msg.value, _message));

    CROSS_L2_OUTBOX.initiateMessage({
        _target: address(this),
        _message: data
    });

    nonce++;
}
```

##### Relaying Messages

```solidity
function relayMessage(uint256 _source, uint256 _nonce, address _sender, address _target, uint256 _value, bytes memory _message) external {
    require(msg.sender == address(CROSS_L2_INBOX));
    require(_source == block.chainid);

    bytes32 messageHash = keccak256(abi.encode(_source, _nonce, _sender, _target, _value, _message));
    require(sentMessages[messageHash] == false);

    xDomainMsgSender = _sender;

    bool success = SafeCall.call({
       _target: _target,
       _value: _value,
       _calldata: _message
    });

    require(success);

    xDomainMsgSender = Constants.DEFAULT_L2_SENDER;
    sentMessages[messageHash] = true;
}
```

It is important to ensure that the source chain is in the dependency set of the destination chain, otherwise
it is possible to send a message that is not playable.

### L1Block

The `L1Block` contract is updated to include the set of allowed chains. The L1 Atrributes transaction
sets the set of allowed chains. The `L1Block` contract MUST provide a public getter to check is a particular
chain is in the dependency set.

The `setL1BlockValuesInterop()` function MUST be called on every block after the interop upgrade block.
The interop upgrade block itself MUST include a call to `setL1BlockValuesEcotone`.

## SystemConfig

The `SystemConfig` is updated to manage the dependency set. The chain operator can add or remove chains from the dependency set
through the `SystemConfig`. A new `ConfigUpdate` event `UpdateType` enum is added that corresponds to a change in the dependency set.

The `SystemConfig` MUST enforce that the maximum size of the dependency set is `type(uint8).max` or 255.

### `DEPENDENCY_SET` UpdateType

When a `ConfigUpdate` event is emitted where the `UpdateType` is `DEPENDENCY_SET`, the L2 network will update its dependency set.
The chain operator SHOULD be able to add or remove chains from the dependency set.

## L1Attributes

The L1 Atrributes transaction is updated to include the dependency set. Since the dependency set is dynamically sized,
a `uint8` "interopSetSize" parameter prefixes tightly packed `uint256` values that represent each chain id.

| Input arg         | Type                    | Calldata bytes          | Segment |
| ----------------- | ----------------------- | ----------------------- | --------|
| {0x760ee04d}      | bytes4                  | 0-3                     | n/a     |
| baseFeeScalar     | uint32                  | 4-7                     | 1       |
| blobBaseFeeScalar | uint32                  | 8-11                    |         |
| sequenceNumber    | uint64                  | 12-19                   |         |
| l1BlockTimestamp  | uint64                  | 20-27                   |         |
| l1BlockNumber     | uint64                  | 28-35                   |         |
| basefee           | uint256                 | 36-67                   | 2       |
| blobBaseFee       | uint256                 | 68-99                   | 3       |
| l1BlockHash       | bytes32                 | 100-131                 | 4       |
| batcherHash       | bytes32                 | 132-163                 | 5       |
| interopSetSize    | uint8                   | 164-165                 | 6       |
| chainIds          | [interopSetSize]uint256 | 165-(32*interopSetSize) | 6+      |

## Fault Proof

TODO: need more detail

No changes to the dispute game bisection are required. The only changes required are to the fault proof program itself.
The insight is that the fault proof program can be a superset of the state transition function.

## Sequencer Policy

The sequencer can include relaying transactions that have corresponding initiating transactions
that only have preconfirmation levels of security if they trust the destination sequencer. Using an
allowlist and identity turns sequencing into an interated game which increases the ability for
sequencers to trust each other. Better preconfirmation technology will help to scale the sequencer
set to untrusted actors.

## Block Building

The block builder SHOULD use static analysis on relaying messages to verify that the initiating
message exists. When a transaction has a top level [to][tx-to] field that is equal to the `CrossL2Inbox`
and the 4byte selector in the calldata matches the `relayMessage(bytes32,CrossChainMessage)` interface
the block builder should use the chain id that is encoded in the `CrossChainMessage` to know which chain includes the initiating
transaction and then use the `_hash` to populate an `eth_getLogs` query to check for the existence of the initiating message.

[tx-to]: (https://github.com/ethereum/execution-specs/blob/1fed0c0074f9d6aab3861057e1924411948dc50b/src/ethereum/frontier/fork_types.py#L52)

## Sponsorship

If a user does not have ether to pay for the gas of a relaying message, application layer sponsorship solutions can be created.
It is possible to create an MEV incentive by paying `tx.origin` in the relaying message. This can be done by wrapping the
`L2ToL2CrossDomainMessenger` with a pair of relaying contracts.

## Security Considerations

### Forced Inclusion of Cross Chain Messages

The design is particular to not introduce any sort of "forced inclusion" between L2s. This design space introduces
risky synchrony assumptions and forces the introduction of a message queue to prevent denial of service attacks where
all chains in the network decide to send cross chain messages to the same chain at the same time.

"Forced inclusion" transactions are good for censorship resistance. In the worst case of censoring sequencers, it will
take at most 2 sequencing windows for the cross chain message to be processed. The initiating transaction can be sent
via a deposit which MUST be included in the source chain or the sequencer will be reorganized at the end of the sequencing
window that includes the deposit transaction. If the relaying transaction is censored, it will take another sequencing window
of time to force the inclusion of the relaying message per the [spec][depositing-a-relaying-message].

TODO: verify exact timing of when reorg happens with deposits that are skipped

[depositing-a-relaying-message]: (interop.md#depositing-a-relaying-message)

### Cross Chain Message Latency

The latency at which a cross chain message is relayed from the moment at which it was initiated is bottlenecked by
the security of the preconfirmations. An initiating transaction and a relaying transaction MAY have the same timestamp,
meaning that a secure preconfirmation scheme enables atomic cross chain composability. Any sort of equivocation on behalf
of the sequencer will result in the production of invalid blocks.

### Dynamic Size of L1 Attributes Transaction

The L1 Attributes transaction includes the dependency set which is dynamically sized. This means that
the worst case (largest size) transaction must be accounted for when ensuring that it is not possible
to create a block that has force inclusion transactions that go over the L2 block gas limit.
It MUST be impossible to produce an L2 block that consumes more than the L2 block gas limit.
Limiting the dependency set size is an easy way to ensure this.

### Maximum Size of the Dependency Set

The maximum size of the dependency set is constrained by the L2 block gas limit. The larger the dependency set,
the more costly it is to fully verify the network. It also makes the block building role more centralized
as it requires more hardware to verify relaying transactions before inclusion.

### Mempool

Since the validation of the relaying message relies on a remote RPC request, this introduces a denial of
service attack vector. The cost of network access is magnitudes larger than in memory validity checks.
The mempool SHOULD perform cheaper checks before any sort of network access is performed. The results
of the check SHOULD be cached such that another request does not need to be performed when building the
block although consistency is not guaranteed.

### Reliance on History

When fully executing historical blocks, a dependency on historical receipts from remote chains is present.
[EIP-4444][eip-4444] will eventually provide a solution for making historical receipts available without
needing to require increasingly large execution client databases.

[eip-4444]: (https://eips.ethereum.org/EIPS/eip-4444)
