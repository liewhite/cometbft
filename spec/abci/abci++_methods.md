---
order: 2
title: Methods
---

# Methods

## Methods existing in ABCI

### Echo

* **Request**:
    * `Message (string)`: A string to echo back
* **Response**:
    * `Message (string)`: The input string
* **Usage**:
    * Echo a string to test an abci client/server implementation

### Flush

* **Usage**:
    * Signals that messages queued on the client should be flushed to
    the server. It is called periodically by the client
    implementation to ensure asynchronous requests are actually
    sent, and is called immediately to make a synchronous request,
    which returns when the Flush response comes back.

### Info

* **Request**:

    | Name          | Type   | Description                              | Field Number |
    |---------------|--------|------------------------------------------|--------------|
    | version       | string | The CometBFT software semantic version | 1            |
    | block_version | uint64 | The CometBFT Block Protocol version    | 2            |
    | p2p_version   | uint64 | The CometBFT P2P Protocol version      | 3            |
    | abci_version  | string | The CometBFT ABCI semantic version     | 4            |

* **Response**:

    | Name                | Type   | Description                                         | Field Number |
    |---------------------|--------|-----------------------------------------------------|--------------|
    | data                | string | Some arbitrary information                          | 1            |
    | version             | string | The application software semantic version           | 2            |
    | app_version         | uint64 | The application protocol version                    | 3            |
    | last_block_height   | int64  | Latest height for which the app persisted its state | 4            |
    | last_block_app_hash | bytes  | Latest AppHash returned by `Commit`                 | 5            |

* **Usage**:
    * Return information about the application state.
    * Used to sync CometBFT with the application during a handshake
      that happens on startup or on recovery.
    * The returned `app_version` will be included in the Header of every block.
    * CometBFT expects `last_block_app_hash` and `last_block_height` to
      be updated and persisted during `Commit`.

> Note: Semantic version is a reference to [semantic versioning](https://semver.org/). Semantic versions in info will be displayed as X.X.x.

### InitChain

* **Request**:

    | Name             | Type                                            | Description                                         | Field Number |
    |------------------|-------------------------------------------------|-----------------------------------------------------|--------------|
    | time             | [google.protobuf.Timestamp][protobuf-timestamp] | Genesis time                                        | 1            |
    | chain_id         | string                                          | ID of the blockchain.                               | 2            |
    | consensus_params | [ConsensusParams](#consensusparams)             | Initial consensus-critical parameters.              | 3            |
    | validators       | repeated [ValidatorUpdate](#validatorupdate)    | Initial genesis validators, sorted by voting power. | 4            |
    | app_state_bytes  | bytes                                           | Serialized initial application state. JSON bytes.   | 5            |
    | initial_height   | int64                                           | Height of the initial block (typically `1`).        | 6            |

* **Response**:

    | Name             | Type                                         | Description                                      | Field Number |
    |------------------|----------------------------------------------|--------------------------------------------------|--------------|
    | consensus_params | [ConsensusParams](#consensusparams)          | Initial consensus-critical parameters (optional) | 1            |
    | validators       | repeated [ValidatorUpdate](#validatorupdate) | Initial validator set (optional).                | 2            |
    | app_hash         | bytes                                        | Initial application hash.                        | 3            |

* **Usage**:
    * Called once upon genesis.
    * If `ResponseInitChain.Validators` is empty, the initial validator set will be the `RequestInitChain.Validators`
    * If `ResponseInitChain.Validators` is not empty, it will be the initial
      validator set (regardless of what is in `RequestInitChain.Validators`).
    * This allows the app to decide if it wants to accept the initial validator
      set proposed by CometBFT (ie. in the genesis file), or if it wants to use
      a different one (perhaps computed based on some application specific
      information in the genesis file).
    * Both `RequestInitChain.Validators` and `ResponseInitChain.Validators` are [ValidatorUpdate](#validatorupdate) structs.
      So, technically, they both are _updating_ the set of validators from the empty set.

### Query

* **Request**:

    | Name   | Type   | Description                                                                                                                                                                                                                                                                            | Field Number |
    |--------|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
    | data   | bytes  | Raw query bytes. Can be used with or in lieu of Path.                                                                                                                                                                                                                                  | 1            |
    | path   | string | Path field of the request URI. Can be used with or in lieu of `data`. Apps MUST interpret `/store` as a query by key on the underlying store. The key SHOULD be specified in the `data` field. Apps SHOULD allow queries over specific types like `/accounts/...` or `/votes/...`      | 2            |
    | height | int64  | The block height for which you want the query (default=0 returns data for the latest committed block). Note that this is the height of the block containing the application's Merkle root hash, which represents the state as it was after committing the block at Height-1            | 3            |
    | prove  | bool   | Return Merkle proof with response if possible                                                                                                                                                                                                                                          | 4            |

* **Response**:

    | Name      | Type                  | Description                                                                                                                                                                                                        | Field Number |
    |-----------|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
    | code      | uint32                | Response code.                                                                                                                                                                                                     | 1            |
    | log       | string                | The output of the application's logger. **May be non-deterministic.**                                                                                                                                              | 3            |
    | info      | string                | Additional information. **May be non-deterministic.**                                                                                                                                                              | 4            |
    | index     | int64                 | The index of the key in the tree.                                                                                                                                                                                  | 5            |
    | key       | bytes                 | The key of the matching data.                                                                                                                                                                                      | 6            |
    | value     | bytes                 | The value of the matching data.                                                                                                                                                                                    | 7            |
    | proof_ops | [ProofOps](#proofops) | Serialized proof for the value data, if requested, to be verified against the `app_hash` for the given Height.                                                                                                     | 8            |
    | height    | int64                 | The block height from which data was derived. Note that this is the height of the block containing the application's Merkle root hash, which represents the state as it was after committing the block at Height-1 | 9            |
    | codespace | string                | Namespace for the `code`.                                                                                                                                                                                          | 10           |

* **Usage**:
    * Query for data from the application at current or past height.
    * Optionally return Merkle proof.
    * Merkle proof includes self-describing `type` field to support many types
    of Merkle trees and encoding formats.

### CheckTx

* **Request**:

    | Name | Type        | Description                                                                                                                                                                                                                                         | Field Number |
    |------|-------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
    | tx   | bytes       | The request transaction bytes                                                                                                                                                                                                                       | 1            |
    | type | CheckTxType | One of `CheckTx_New` or `CheckTx_Recheck`. `CheckTx_New` is the default and means that a full check of the tranasaction is required. `CheckTx_Recheck` types are used when the mempool is initiating a normal recheck of a transaction.             | 2            |

* **Response**:

    | Name       | Type                                                        | Description                                                           | Field Number |
    |------------|-------------------------------------------------------------|-----------------------------------------------------------------------|--------------|
    | code       | uint32                                                      | Response code.                                                        | 1            |
    | data       | bytes                                                       | Result bytes, if any.                                                 | 2            |
    | gas_wanted | int64                                                       | Amount of gas requested for transaction.                              | 5            |
    | codespace  | string                                                      | Namespace for the `code`.                                             | 8            |
    | sender     | string                                                      | The transaction's sender (e.g. the signer)                            | 9            |
    | priority   | int64                                                       | The transaction's priority (for mempool ordering)                     | 10           |

* **Usage**:

    * Technically optional - not involved in processing blocks.
    * Guardian of the mempool: every node runs `CheckTx` before letting a
      transaction into its local mempool.
    * The transaction may come from an external user or another node
    * `CheckTx` validates the transaction against the current state of the application,
      for example, checking signatures and account balances, but does not apply any
      of the state changes described in the transaction.
    * Transactions where `ResponseCheckTx.Code != 0` will be rejected - they will not be broadcast
      to other nodes or included in a proposal block.
      CometBFT attributes no other value to the response code.

### BeginBlock

* **Request**:

    | Name                 | Type                                        | Description                                                                                                       | Field Number |
    |----------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------|--------------|
    | hash                 | bytes                                       | The block's hash. This can be derived from the block header.                                                      | 1            |
    | header               | [Header](../core/data_structures.md#header) | The block header.                                                                                                 | 2            |
    | last_commit_info     | [CommitInfo](#commitinfo)           | Info about the last commit, including the round, and the list of validators and which ones signed the last block. | 3            |
    | byzantine_validators | repeated [Evidence](abci++_basic_concepts.md#evidence)              | List of evidence of validators that acted maliciously.                                                            | 4            |

* **Response**:

    | Name   | Type                      | Description                         | Field Number |
    |--------|---------------------------|-------------------------------------|--------------|
    | events | repeated [Event](abci++_basic_concepts.md#events) | type & Key-Value events for indexing | 1           |

* **Usage**:
    * Signals the beginning of a new block.
    * Called prior to any `DeliverTx` method calls.
    * The header contains the height, timestamp, and more - it exactly matches the
    CometBFT block header. We may seek to generalize this in the future.
    * The `CommitInfo` and `ByzantineValidators` can be used to determine
    rewards and punishments for the validators.

### DeliverTx

* **Request**:

    | Name | Type  | Description                    | Field Number |
    |------|-------|--------------------------------|--------------|
    | tx   | bytes | The request transaction bytes. | 1            |

* **Response**:

    | Name       | Type                      | Description                                                           | Field Number |
    |------------|---------------------------|-----------------------------------------------------------------------|--------------|
    | code       | uint32                    | Response code.                                                        | 1            |
    | data       | bytes                     | Result bytes, if any.                                                 | 2            |
    | log        | string                    | The output of the application's logger. **May be non-deterministic.** | 3            |
    | info       | string                    | Additional information. **May be non-deterministic.**                 | 4            |
    | gas_wanted | int64                     | Amount of gas requested for transaction.                              | 5            |
    | gas_used   | int64                     | Amount of gas consumed by transaction.                                | 6            |
    | events     | repeated [Event](abci++_basic_concepts.md#events) | Type & Key-Value events for indexing transactions (eg. by account).   | 7            |
    | codespace  | string                    | Namespace for the `code`.                                             | 8            |

* **Usage**:
    * [**Required**] The core method of the application.
    * `DeliverTx` is called once for each transaction in the block.
    * When `DeliverTx` is called, the application must execute the transaction deterministically
    in full before returning control to CometBFT.
    * Alternatively, the application can apply a candidate state corresponding
     to the same block previously executed via `PrepareProposal` or `ProcessProposal` any time between the calls to `BeginBlock`, the various
     calls to `DeliverTx` and `EndBlock`.
    * `ResponseDeliverTx.Code == 0` only if the transaction is fully valid.


### EndBlock

* **Request**:

    | Name   | Type  | Description                        | Field Number |
    |--------|-------|------------------------------------|--------------|
    | height | int64 | Height of the block just executed. | 1            |

* **Response**:

    | Name                    | Type                                         | Description                                                     | Field Number |
    |-------------------------|----------------------------------------------|-----------------------------------------------------------------|--------------|
    | validator_updates       | repeated [ValidatorUpdate](#validatorupdate) | Changes to validator set (set voting power to 0 to remove).     | 1            |
    | consensus_param_updates | [ConsensusParams](#consensusparams)          | Changes to consensus-critical time, size, and other parameters. | 2            |
    | events                  | repeated [Event](abci++_basic_concepts.md#events)                    | Type & Key-Value events for indexing                            | 3            |

* **Usage**:
    * Signals the end of a block.
    * Called after all the transactions for the current block have been delivered, prior to the block's `Commit` message.
    * Optional `validator_updates` triggered by block `H`. These updates affect validation
      for blocks `H+1`, `H+2`, and `H+3`.
    * Heights following a validator update are affected in the following way:
        * `H+1`: `NextValidatorsHash` includes the new `validator_updates` value.
        * `H+2`: The validator set change takes effect and `ValidatorsHash` is updated.
        * `H+3`: `last_commit_info (BeginBlock)` is changed to include the altered validator set and `*_last_commit` fields in `PrepareProposal`, `ProcessProposal` now include the altered validator set.
    * `consensus_param_updates` returned for block `H` apply to the consensus
      params for block `H+1`. For more information on the consensus parameters,
      see the [application spec entry on consensus parameters](./abci++_app_requirements.md#consensus-parameters).
    * `validator_updates` and `consensus_param_updates` may be empty. In this case, CometBFT will keep the current values.



### Commit

* **Request**:

    | Name   | Type  | Description                        | Field Number |
    |--------|-------|------------------------------------|--------------|

    Commit signals the application to persist application state. It takes no parameters.
* **Response**:

    | Name          | Type  | Description                                                            | Field Number |
    |---------------|-------|------------------------------------------------------------------------|--------------|
    | data          | bytes | The Merkle root hash of the application state.                         | 2            |
    | retain_height | int64 | Blocks below this height may be removed. Defaults to `0` (retain all). | 3            |

* **Usage**:
    * Signal the application to persist the application state.
    * Return an (optional) Merkle root hash of the application state
    * `ResponseCommit.Data` is included as the `Header.AppHash` in the next block
        * It may be empty or hard-coded, but MUST be **deterministic** - it must not be a function of anything that did not come from the parameters of the execution calls (`BeginBlock/DeliverTx/EndBlock methods`) and the previous committed state.
    * Later calls to `Query` can return proofs about the application state anchored
    in this Merkle root hash
    * Use `RetainHeight` with caution! If all nodes in the network remove historical
    blocks then this data is permanently lost, and no new nodes will be able to
    join the network and bootstrap. Historical blocks may also be required for
    other purposes, e.g. auditing, replay of non-persisted heights, light client
    verification, and so on.

### ListSnapshots

* **Request**:

    | Name   | Type  | Description                        | Field Number |
    |--------|-------|------------------------------------|--------------|

    Empty request asking the application for a list of snapshots.

* **Response**:

    | Name      | Type                           | Description                    | Field Number |
    |-----------|--------------------------------|--------------------------------|--------------|
    | snapshots | repeated [Snapshot](#snapshot) | List of local state snapshots. | 1            |

* **Usage**:
    * Used during state sync to discover available snapshots on peers.
    * See `Snapshot` data type for details.

### LoadSnapshotChunk

* **Request**:

    | Name   | Type   | Description                                                           | Field Number |
    |--------|--------|-----------------------------------------------------------------------|--------------|
    | height | uint64 | The height of the snapshot the chunk belongs to.                      | 1            |
    | format | uint32 | The application-specific format of the snapshot the chunk belongs to. | 2            |
    | chunk  | uint32 | The chunk index, starting from `0` for the initial chunk.             | 3            |

* **Response**:

    | Name  | Type  | Description                                                                                                                                           | Field Number |
    |-------|-------|-------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
    | chunk | bytes | The binary chunk contents, in an arbitray format. Chunk messages cannot be larger than 16 MB _including metadata_, so 10 MB is a good starting point. | 1            |

* **Usage**:
    * Used during state sync to retrieve snapshot chunks from peers.

### OfferSnapshot

* **Request**:

    | Name     | Type                  | Description                                                              | Field Number |
    |----------|-----------------------|--------------------------------------------------------------------------|--------------|
    | snapshot | [Snapshot](#snapshot) | The snapshot offered for restoration.                                    | 1            |
    | app_hash | bytes                 | The light client-verified app hash for this height, from the blockchain. | 2            |

* **Response**:

    | Name   | Type              | Description                       | Field Number |
    |--------|-------------------|-----------------------------------|--------------|
    | result | [Result](#result) | The result of the snapshot offer. | 1            |

#### Result

```protobuf
  enum Result {
    UNKNOWN       = 0;  // Unknown result, abort all snapshot restoration
    ACCEPT        = 1;  // Snapshot is accepted, start applying chunks.
    ABORT         = 2;  // Abort snapshot restoration, and don't try any other snapshots.
    REJECT        = 3;  // Reject this specific snapshot, try others.
    REJECT_FORMAT = 4;  // Reject all snapshots with this `format`, try others.
    REJECT_SENDER = 5;  // Reject all snapshots from all senders of this snapshot, try others.
  }
```

* **Usage**:
    * `OfferSnapshot` is called when bootstrapping a node using state sync. The application may
    accept or reject snapshots as appropriate. Upon accepting, CometBFT will retrieve and
    apply snapshot chunks via `ApplySnapshotChunk`. The application may also choose to reject a
    snapshot in the chunk response, in which case it should be prepared to accept further
    `OfferSnapshot` calls.
    * Only `AppHash` can be trusted, as it has been verified by the light client. Any other data
    can be spoofed by adversaries, so applications should employ additional verification schemes
    to avoid denial-of-service attacks. The verified `AppHash` is automatically checked against
    the restored application at the end of snapshot restoration.
    * For more information, see the `Snapshot` data type or the [state sync section](../p2p/legacy-docs/messages/state-sync.md).

### ApplySnapshotChunk

* **Request**:

    | Name   | Type   | Description                                                                 | Field Number |
    |--------|--------|-----------------------------------------------------------------------------|--------------|
    | index  | uint32 | The chunk index, starting from `0`. CometBFT applies chunks sequentially. | 1            |
    | chunk  | bytes  | The binary chunk contents, as returned by `LoadSnapshotChunk`.              | 2            |
    | sender | string | The P2P ID of the node who sent this chunk.                                 | 3            |

* **Response**:

    | Name           | Type                | Description                                                                                                                                                                                                                             | Field Number |
    |----------------|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
    | result         | Result  (see below) | The result of applying this chunk.                                                                                                                                                                                                      | 1            |
    | refetch_chunks | repeated uint32     | Refetch and reapply the given chunks, regardless of `result`. Only the listed chunks will be refetched, and reapplied in sequential order.                                                                                              | 2            |
    | reject_senders | repeated string     | Reject the given P2P senders, regardless of `Result`. Any chunks already applied will not be refetched unless explicitly requested, but queued chunks from these senders will be discarded, and new chunks or other snapshots rejected. | 3            |

```proto
  enum Result {
    UNKNOWN         = 0;  // Unknown result, abort all snapshot restoration
    ACCEPT          = 1;  // The chunk was accepted.
    ABORT           = 2;  // Abort snapshot restoration, and don't try any other snapshots.
    RETRY           = 3;  // Reapply this chunk, combine with `RefetchChunks` and `RejectSenders` as appropriate.
    RETRY_SNAPSHOT  = 4;  // Restart this snapshot from `OfferSnapshot`, reusing chunks unless instructed otherwise.
    REJECT_SNAPSHOT = 5;  // Reject this snapshot, try a different one.
  }
```

* **Usage**:
    * The application can choose to refetch chunks and/or ban P2P peers as appropriate. CometBFT
    will not do this unless instructed by the application.
    * The application may want to verify each chunk, e.g. by attaching chunk hashes in
    `Snapshot.Metadata` and/or incrementally verifying contents against `AppHash`.
    * When all chunks have been accepted, CometBFT will make an ABCI `Info` call to verify that
    `LastBlockAppHash` and `LastBlockHeight` matches the expected values, and record the
    `AppVersion` in the node state. It then switches to block sync or consensus and joins the
    network.
    * If CometBFT is unable to retrieve the next chunk after some time (e.g. because no suitable
    peers are available), it will reject the snapshot and try a different one via `OfferSnapshot`.
    The application should be prepared to reset and accept it or abort as appropriate.

## New methods introduced in ABCI 2.0

### PrepareProposal

#### Parameters and Types

* **Request**:

    | Name                 | Type                                            | Description                                                                                   | Field Number |
    |----------------------|-------------------------------------------------|-----------------------------------------------------------------------------------------------|--------------|
    | max_tx_bytes         | int64                                           | Currently configured maximum size in bytes taken by the modified transactions.                | 1            |
    | txs                  | repeated bytes                                  | Preliminary list of transactions that have been picked as part of the block to propose.       | 2            |
    | local_last_commit    | [ExtendedCommitInfo](#extendedcommitinfo)       | Info about the last commit, obtained locally from CometBFT's data structures.               | 3            |
    | misbehavior          | repeated [Misbehavior](#misbehavior)            | List of information about validators that misbehaved.                                         | 4            |
    | height               | int64                                           | The height of the block that will be proposed.                                                | 5            |
    | time                 | [google.protobuf.Timestamp][protobuf-timestamp] | Timestamp of the block that that will be proposed.                                            | 6            |
    | next_validators_hash | bytes                                           | Merkle root of the next validator set.                                                        | 7            |
    | proposer_address     | bytes                                           | [Address](../core/data_structures.md#address) of the validator that is creating the proposal. | 8            |

* **Response**:

    | Name                    | Type                                             | Description                                                                                 | Field Number |
    |-------------------------|--------------------------------------------------|---------------------------------------------------------------------------------------------|--------------|
    | txs              | repeated bytes                   | Possibly modified list of transactions that have been picked as part of the proposed block. | 2            |

* **Usage**:
    * `RequestPrepareProposal`'s parameters `txs`, `misbehavior`, `height`, `time`,
      `next_validators_hash`, and `proposer_address` are the same as in `RequestProcessProposal`.
    * `RequestPrepareProposal.local_last_commit` is a set of the precommit votes that allowed the
      decision of the previous block.
    * Fields `height`, `time`, `proposer_address`, and `next_validators_hash` match the values from
      the header of the proposed block.
    * `RequestPrepareProposal` contains a preliminary set of transactions `txs` that CometBFT
      retrieved from the mempool, called _raw proposal_. The Application can modify this
      set and return a modified set of transactions via `ResponsePrepareProposal.txs` .
        * The Application _can_ modify the raw proposal: it can reorder, remove or add transactions.
          Let `tx` be a transaction in `txs` (set of transactions within `RequestPrepareProposal`):
            * If the Application considers that `tx` should not be proposed in this block, e.g.,
              there are other transactions with higher priority, then it should not include it in
              `ResponsePrepareProposal.txs`. However, this will not remove `tx` from the mempool.
            * If the Application wants to add a new transaction to the proposed block, then the
              Application includes it in `ResponsePrepareProposal.txs`. CometBFT will not add
              the transaction to the mempool.
        * The Application should be aware that removing and adding transactions may compromise
          _traceability_.
          > Consider the following example: the Application transforms a client-submitted
            transaction `t1` into a second transaction `t2`, i.e., the Application asks CometBFT
            to remove `t1` from the block and add `t2` to the block. If a client wants to eventually check what
            happened to `t1`, it will discover that `t1` is not in a
            committed block (assuming a _re-CheckTx_ evited it from the mempool), getting the wrong idea that `t1` did not make it into a block. Note
            that `t2` _will be_ in a committed block, but unless the Application tracks this
            information, no component will be aware of it. Thus, if the Application wants
            traceability, it is its responsability to support it. For instance, the Application
            could attach to a transformed transaction a list with the hashes of the transactions it
            derives from.
    * CometBFT MAY include a list of transactions in `RequestPrepareProposal.txs` whose total
      size in bytes exceeds `RequestPrepareProposal.max_tx_bytes`.
      Therefore, if the size of `RequestPrepareProposal.txs` is greater than
      `RequestPrepareProposal.max_tx_bytes`, the Application MUST remove transactions to ensure
      that the `RequestPrepareProposal.max_tx_bytes` limit is respected by those transactions
      returned in `ResponsePrepareProposal.txs` .
    * As a result of executing the prepared proposal, the Application may produce block events or transaction events.
      The Application must keep those events until a block is decided. It will then forward the events to the `BeginBlock-DeliverTx-EndBlock` functions depending on where each event should be placed, thereby returning the events to CometBFT.
    * CometBFT does NOT provide any additional validity checks (such as checking for duplicate
      transactions).
      <!--
      As a sanity check, CometBFT will check the returned parameters for validity if the Application modified them.
      In particular, `ResponsePrepareProposal.txs` will be deemed invalid if there are duplicate transactions in the list.
       -->
    * If CometBFT fails to validate the `ResponsePrepareProposal`, CometBFT will assume the
      Application is faulty and crash.
    * The implementation of `PrepareProposal` can be non-deterministic.


#### When does CometBFT call `PrepareProposal` ?


When a validator _p_ enters consensus round _r_, height _h_, in which _p_ is the proposer,
and _p_'s _validValue_ is `nil`:

1. CometBFT collects outstanding transactions from _p_'s mempool
    * the transactions will be collected in order of priority
    * _p_'s CometBFT creates a block header.
2. _p_'s CometBFT calls `RequestPrepareProposal` with the newly generated block, the local
   commit of the previous height (with vote extensions), and any outstanding evidence of
   misbehavior. The call is synchronous: CometBFT's execution will block until the Application
   returns from the call.
3. The Application uses the information received (transactions, commit info, misbehavior, time) to
    (potentially) modify the proposal.
    * the Application MAY fully execute the block and produce a candidate state (immediate execution)
    * the Application can manipulate transactions:
        * leave transactions untouched
        * add new transactions (not present initially) to the proposal
        * remove transactions from the proposal (but not from the mempool thus effectively _delaying_ them) - the
          Application does not include the transaction in `ResponsePrepareProposal.txs`.
        * modify transactions (e.g. aggregate them). As explained above, this compromises client traceability, unless
          it is implemented at the Application level.
        * reorder transactions - the Application reorders transactions in the list
4. The Application includes the transaction list (whether modified or not) in the return parameters
   (see the rules in section _Usage_), and returns from the call.
5. _p_ uses the (possibly) modified block as _p_'s proposal in round _r_, height _h_.

Note that, if _p_ has a non-`nil` _validValue_ in round _r_, height _h_,
the consensus algorithm will use it as proposal and will not call `RequestPrepareProposal`.

### ProcessProposal

#### Parameters and Types

* **Request**:

    | Name                 | Type                                            | Description                                                                               | Field Number |
    |----------------------|-------------------------------------------------|-------------------------------------------------------------------------------------------|--------------|
    | txs                  | repeated bytes                                  | List of transactions of the proposed block.                                               | 1            |
    | proposed_last_commit | [CommitInfo](#commitinfo)                       | Info about the last commit, obtained from the information in the proposed block.          | 2            |
    | misbehavior          | repeated [Misbehavior](#misbehavior)            | List of information about validators that misbehaved.                                     | 3            |
    | hash                 | bytes                                           | The hash of the proposed block.                                                           | 4            |
    | height               | int64                                           | The height of the proposed block.                                                         | 5            |
    | time                 | [google.protobuf.Timestamp][protobuf-timestamp] | Timestamp of the proposed block.                                                          | 6            |
    | next_validators_hash | bytes                                           | Merkle root of the next validator set.                                                    | 7            |
    | proposer_address     | bytes                                           | [Address](../core/data_structures.md#address) of the validator that created the proposal. | 8            |

* **Response**:

    | Name                    | Type                                             | Description                                                                       | Field Number |
    |-------------------------|--------------------------------------------------|-----------------------------------------------------------------------------------|--------------|
    | status                  | [ProposalStatus](#proposalstatus)                | `enum` that signals if the application finds the proposal valid.                  | 1            |

* **Usage**:
    * Contains all information on the proposed block needed to fully execute it.
        * The Application may fully execute the block as though it was handling the calls to `BeginBlock-DeliverTx-EndBlock`.
        * However, any resulting state changes must be kept as _candidate state_,
          and the Application should be ready to discard it in case another block is decided.
    * `RequestProcessProposal` is also called at the proposer of a round.
      Normally the call to `RequestProcessProposal` occurs right after the call to `RequestPrepareProposal` and
      `RequestProcessProposal` matches the block produced based on `ResponsePrepareProposal` (i.e.,
      `RequestPrepareProposal.txs` equals `RequestProcessProposal.txs`).
      However, no such guarantee is made since, in the presence of failures, `RequestProcessProposal` may match
      `ResponsePrepareProposal` from an earlier invocation or `ProcessProposal` may not be invoked at all.
    * The height and time values match the values from the header of the proposed block.
    * If `ResponseProcessProposal.status` is `REJECT`, consensus assumes the proposal received
      is not valid.
    * The Application MAY fully execute the block &mdash; immediate execution
    * The implementation of `ProcessProposal` MUST be deterministic. Moreover, the value of
      `ResponseProcessProposal.status` MUST **exclusively** depend on the parameters passed in
      the call to `RequestProcessProposal`, and the last committed Application state
      (see [Requirements](./abci++_app_requirements.md) section).
    * Moreover, application implementors SHOULD always set `ResponseProcessProposal.status` to `ACCEPT`,
      unless they _really_ know what the potential liveness implications of returning `REJECT` are.

#### When does CometBFT call `ProcessProposal` ?

When a node _p_ enters consensus round _r_, height _h_, in which _q_ is the proposer (possibly _p_ = _q_):

1. _p_ sets up timer `ProposeTimeout`.
2. If _p_ is the proposer, _p_ executes steps 1-6 in [PrepareProposal](#prepareproposal).
3. Upon reception of Proposal message (which contains the header) for round _r_, height _h_ from
   _q_, _p_ verifies the block header.
4. Upon reception of Proposal message, along with all the block parts, for round _r_, height _h_
   from _q_, _p_ follows the validators' algorithm to check whether it should prevote for the
   proposed block, or `nil`.
5. If the validators' consensus algorithm indicates _p_ should prevote non-nil:
    1. CometBFT calls `RequestProcessProposal` with the block. The call is synchronous.
    2. The Application checks/processes the proposed block, which is read-only, and returns
       `ACCEPT` or `REJECT` in the `ResponseProcessProposal.status` field.
       * The Application, depending on its needs, may call `ResponseProcessProposal`
         * either after it has completely processed the block (immediate execution),
         * or after doing some basic checks, and process the block asynchronously. In this case the
           Application will not be able to reject the block, or force prevote/precommit `nil`
           afterwards.
         * or immediately, returning `ACCEPT`, if _p_ is not a validator
           and the Application does not want non-validating nodes to handle `ProcessProposal`
    3. If _p_ is a validator and the returned value is
         * `ACCEPT`: _p_ prevotes on this proposal for round _r_, height _h_.
         * `REJECT`: _p_ prevotes `nil`.

<!--

### ExtendVote

#### Parameters and Types

* **Request**:

    | Name   | Type  | Description                                                                   | Field Number |
    |--------|-------|-------------------------------------------------------------------------------|--------------|
    | hash   | bytes | The header hash of the proposed block that the vote extension is to refer to. | 1            |
    | height | int64 | Height of the proposed block (for sanity check).                              | 2            |

* **Response**:

    | Name              | Type  | Description                                             | Field Number |
    |-------------------|-------|---------------------------------------------------------|--------------|
    | vote_extension    | bytes | Information signed by by CometBFT. Can have 0 length. | 1            |

* **Usage**:
    * `ResponseExtendVote.vote_extension` is application-generated information that will be signed
      by CometBFT and attached to the Precommit message.
    * The Application may choose to use an empty vote extension (0 length).
    * `RequestExtendVote.hash` corresponds to the hash of a proposed block that was made available
      to the Application in a previous call to `ProcessProposal` for the current height.
    * `ResponseExtendVote.vote_extension` will only be attached to a non-`nil` Precommit message. If the consensus algorithm is to
      precommit `nil`, it will not call `RequestExtendVote`.
    * The Application logic that creates the extension can be non-deterministic.

#### When does CometBFT call `ExtendVote`?

When a validator _p_ is in consensus state _prevote_ of round _r_, height _h_, in which _q_ is the proposer; and _p_ has received

* the Proposal message _v_ for round _r_, height _h_, along with all the block parts, from _q_,
* `Prevote` messages from _2f + 1_ validators' voting power for round _r_, height _h_, prevoting for the same block _id(v)_,

then _p_ locks _v_  and sends a Precommit message in the following way

1. _p_ sets _lockedValue_ and _validValue_ to _v_, and sets _lockedRound_ and _validRound_ to _r_
2. _p_'s CometBFT calls `RequestExtendVote` with _id(v)_ (`RequestExtendVote.hash`). The call is synchronous.
3. The Application returns an array of bytes, `ResponseExtendVote.extension`, which is not interpreted by the consensus algorithm.
4. _p_ includes `ResponseExtendVote.extension` in a field of type [CanonicalVoteExtension](#canonicalvoteextension),
   it then populates the other fields in [CanonicalVoteExtension](#canonicalvoteextension), and signs the populated
   data structure.
5. _p_ constructs and signs the [CanonicalVote](../core/data_structures.md#canonicalvote) structure.
6. _p_ constructs the Precommit message (i.e. [Vote](../core/data_structures.md#vote) structure)
   using [CanonicalVoteExtension](#canonicalvoteextension) and [CanonicalVote](../core/data_structures.md#canonicalvote).
7. _p_ broadcasts the Precommit message.

In the cases when _p_ is to broadcast `precommit nil` messages (either _2f+1_ `prevote nil` messages received,
or _timeoutPrevote_ triggered), _p_'s CometBFT does **not** call `RequestExtendVote` and will not include
a [CanonicalVoteExtension](#canonicalvoteextension) field in the `precommit nil` message.

### VerifyVoteExtension

#### Parameters and Types

* **Request**:

    | Name              | Type  | Description                                                                               | Field Number |
    |-------------------|-------|-------------------------------------------------------------------------------------------|--------------|
    | hash              | bytes | The hash of the proposed block that the vote extension refers to.                         | 1            |
    | validator_address | bytes | [Address](../core/data_structures.md#address) of the validator that signed the extension. | 2            |
    | height            | int64 | Height of the block (for sanity check).                                                   | 3            |
    | vote_extension    | bytes | Application-specific information signed by CometBFT. Can have 0 length.                 | 4            |

* **Response**:

    | Name   | Type                          | Description                                                    | Field Number |
    |--------|-------------------------------|----------------------------------------------------------------|--------------|
    | status | [VerifyStatus](#verifystatus) | `enum` signaling if the application accepts the vote extension | 1            |

* **Usage**:
    * `RequestVerifyVoteExtension.vote_extension` can be an empty byte array. The Application's
      interpretation of it should be
      that the Application running at the process that sent the vote chose not to extend it.
      CometBFT will always call `RequestVerifyVoteExtension`, even for 0 length vote extensions.
    * `RequestVerifyVoteExtension` is not called for precommit votes sent by the local process.
    * `RequestVerifyVoteExtension.hash` refers to a proposed block. There is not guarantee that
      this proposed block has previously been exposed to the Application via `ProcessProposal`.
    * If `ResponseVerifyVoteExtension.status` is `REJECT`, the consensus algorithm will reject the whole received vote.
      See the [Requirements](./abci++_app_requirements.md) section to understand the potential
      liveness implications of this.
    * The implementation of `VerifyVoteExtension` MUST be deterministic. Moreover, the value of
      `ResponseVerifyVoteExtension.status` MUST **exclusively** depend on the parameters passed in
      the call to `RequestVerifyVoteExtension`, and the last committed Application state
      (see [Requirements](./abci++_app_requirements.md) section).
    * Moreover, application implementers SHOULD always set `ResponseVerifyVoteExtension.status` to `ACCEPT`,
      unless they _really_ know what the potential liveness implications of returning `REJECT` are.

#### When does CometBFT call `VerifyVoteExtension`?

When a node _p_ is in consensus round _r_, height _h_, and _p_ receives a Precommit
message for round _r_, height _h_ from validator _q_ (_q_ &ne; _p_):

1. If the Precommit message does not contain a vote extension with a valid signature, _p_
   discards the Precommit message as invalid.
   * a 0-length vote extension is valid as long as its accompanying signature is also valid.
2. Else, _p_'s CometBFT calls `RequestVerifyVoteExtension`.
3. The Application returns `ACCEPT` or `REJECT` via `ResponseVerifyVoteExtension.status`.
4. If the Application returns
   * `ACCEPT`, _p_ will keep the received vote, together with its corresponding
     vote extension in its internal data structures. It will be used to populate the [ExtendedCommitInfo](#extendedcommitinfo)
     structure in calls to `RequestPrepareProposal`, in rounds of height _h + 1_ where _p_ is the proposer.
   * `REJECT`, _p_ will deem the Precommit message invalid and discard it.

### FinalizeBlock

#### Parameters and Types

* **Request**:

    | Name                 | Type                                            | Description                                                                               | Field Number |
    |----------------------|-------------------------------------------------|-------------------------------------------------------------------------------------------|--------------|
    | txs                  | repeated bytes                                  | List of transactions committed as part of the block.                                      | 1            |
    | decided_last_commit  | [CommitInfo](#commitinfo)                       | Info about the last commit, obtained from the block that was just decided.                | 2            |
    | misbehavior          | repeated [Misbehavior](#misbehavior)            | List of information about validators that misbehaved.                                     | 3            |
    | hash                 | bytes                                           | The block's hash.                                                                         | 4            |
    | height               | int64                                           | The height of the finalized block.                                                        | 5            |
    | time                 | [google.protobuf.Timestamp][protobuf-timestamp] | Timestamp of the finalized block.                                                         | 6            |
    | next_validators_hash | bytes                                           | Merkle root of the next validator set.                                                    | 7            |
    | proposer_address     | bytes                                           | [Address](../core/data_structures.md#address) of the validator that created the proposal. | 8            |

* **Response**:

    | Name                    | Type                                                        | Description                                                                      | Field Number |
    |-------------------------|-------------------------------------------------------------|----------------------------------------------------------------------------------|--------------|
    | events                  | repeated [Event](abci++_basic_concepts.md#events)           | Type & Key-Value events for indexing                                             | 1            |
    | tx_results              | repeated [ExecTxResult](#exectxresult)                      | List of structures containing the data resulting from executing the transactions | 2            |
    | validator_updates       | repeated [ValidatorUpdate](#validatorupdate)                | Changes to validator set (set voting power to 0 to remove).                      | 3            |
    | consensus_param_updates | [ConsensusParams](#consensusparams)                         | Changes to gas, size, and other consensus-related parameters.                    | 4            |
    | app_hash                | bytes                                                       | The Merkle root hash of the application state.                                   | 5            |

* **Usage**:
    * Contains the fields of the newly decided block.
    * This method is equivalent to the call sequence `BeginBlock`, [`DeliverTx`],
      and `EndBlock` in the previous version of ABCI.
    * The height and time values match the values from the header of the proposed block.
    * The Application can use `RequestFinalizeBlock.decided_last_commit` and `RequestFinalizeBlock.misbehavior`
      to determine rewards and punishments for the validators.
    * The Application executes the transactions in `RequestFinalizeBlock.txs` deterministically,
      according to the rules set up by the Application, before returning control to CometBFT.
      Alternatively, it can apply the candidate state corresponding to the same block previously
      executed via `PrepareProposal` or `ProcessProposal`.
    * `ResponseFinalizeBlock.tx_results[i].Code == 0` only if the _i_-th transaction is fully valid.
    * The Application must provide values for `ResponseFinalizeBlock.app_hash`,
      `ResponseFinalizeBlock.tx_results`, `ResponseFinalizeBlock.validator_updates`, and
      `ResponseFinalizeBlock.consensus_param_updates` as a result of executing the block.
        * The values for `ResponseFinalizeBlock.validator_updates`, or
          `ResponseFinalizeBlock.consensus_param_updates` may be empty. In this case, CometBFT will keep
          the current values.
        * `ResponseFinalizeBlock.validator_updates`, triggered by block `H`, affect validation
          for blocks `H+1`, `H+2`, and `H+3`. Heights following a validator update are affected in the following way:
            * Height `H+1`: `NextValidatorsHash` includes the new `validator_updates` value.
            * Height `H+2`: The validator set change takes effect and `ValidatorsHash` is updated.
            * Height `H+3`: `*_last_commit` fields in `PrepareProposal`, `ProcessProposal`, and
              `FinalizeBlock` now include the altered validator set.
        * `ResponseFinalizeBlock.consensus_param_updates` returned for block `H` apply to the consensus
          params for block `H+1`. For more information on the consensus parameters,
          see the [consensus parameters](./abci%2B%2B_app_requirements.md#consensus-parameters)
          section.
    * `ResponseFinalizeBlock.app_hash` contains an (optional) Merkle root hash of the application state.
    * `ResponseFinalizeBlock.app_hash` is included as the `Header.AppHash` in the next block.
        * `ResponseFinalizeBlock.app_hash` may also be empty or hard-coded, but MUST be
          **deterministic** - it must not be a function of anything that did not come from the parameters
          of `RequestFinalizeBlock` and the previous committed state.
    * Later calls to `Query` can return proofs about the application state anchored
      in this Merkle root hash.
    * The implementation of `FinalizeBlock` MUST be deterministic, since it is
      making the Application's state evolve in the context of state machine replication.
    * Currently, CometBFT will fill up all fields in `RequestFinalizeBlock`, even if they were
      already passed on to the Application via `RequestPrepareProposal` or `RequestProcessProposal`.

#### When does CometBFT call `FinalizeBlock`?

When a node _p_ is in consensus height _h_, and _p_ receives

* the Proposal message with block _v_ for a round _r_, along with all its block parts, from _q_,
  which is the proposer of round _r_, height _h_,
* `Precommit` messages from _2f + 1_ validators' voting power for round _r_, height _h_,
  precommitting the same block _id(v)_,

then _p_ decides block _v_ and finalizes consensus for height _h_ in the following way

1. _p_ persists _v_ as the decision for height _h_.
2. _p_'s CometBFT calls `RequestFinalizeBlock` with _v_'s data. The call is synchronous.
3. _p_'s Application executes block _v_.
4. _p_'s Application calculates and returns the _AppHash_, along with a list containing
   the outputs of each of the transactions executed.
5. _p_'s CometBFT hashes all the transaction outputs and stores it in _ResultHash_.
6. _p_'s CometBFT persists the transaction outputs, _AppHash_, and _ResultsHash_.
7. _p_'s CometBFT locks the mempool &mdash; no calls to `CheckTx` on new transactions.
8. _p_'s CometBFT calls `RequestCommit` to instruct the Application to persist its state.
9. _p_'s CometBFT, optionally, re-checks all outstanding transactions in the mempool
   against the newly persisted Application state.
10. _p_'s CometBFT unlocks the mempool &mdash; newly received transactions can now be checked.
11. _p_ starts consensus for height _h+1_, round 0

-->

## Data Types existing in ABCI

Most of the data structures used in ABCI are shared [common data structures](../core/data_structures.md). In certain cases, ABCI uses different data structures which are documented here:

### Validator

* **Fields**:

    | Name    | Type  | Description                                                         | Field Number |
    |---------|-------|---------------------------------------------------------------------|--------------|
    | address | bytes | [Address](../core/data_structures.md#address) of validator          | 1            |
    | power   | int64 | Voting power of the validator                                       | 3            |

* **Usage**:
    * Validator identified by address
    * Used in RequestBeginBlock as part of VoteInfo
    * Does not include PubKey to avoid sending potentially large quantum pubkeys
    over the ABCI

### ValidatorUpdate

* **Fields**:

    | Name    | Type                                             | Description                   | Field Number |
    |---------|--------------------------------------------------|-------------------------------|--------------|
    | pub_key | [Public Key](../core/data_structures.md#pub_key) | Public key of the validator   | 1            |
    | power   | int64                                            | Voting power of the validator | 2            |

* **Usage**:
    * Validator identified by PubKey
    * Used to tell CometBFT to update the validator set

### Misbehavior

* **Fields**:

    | Name               | Type                                            | Description                                                                  | Field Number |
    |--------------------|-------------------------------------------------|------------------------------------------------------------------------------|--------------|
    | type               | [MisbehaviorType](#misbehaviortype)             | Type of the misbehavior. An enum of possible misbehaviors.                   | 1            |
    | validator          | [Validator](#validator)                         | The offending validator                                                      | 2            |
    | height             | int64                                           | Height when the offense occurred                                             | 3            |
    | time               | [google.protobuf.Timestamp][protobuf-timestamp] | Timestamp of the block that was committed at height `height`                 | 4            |
    | total_voting_power | int64                                           | Total voting power of the validator set at height `height`                   | 5            |

#### MisbehaviorType

* **Fields**

    MisbehaviorType is an enum with the listed fields:

    | Name                | Field Number |
    |---------------------|--------------|
    | UNKNOWN             | 0            |
    | DUPLICATE_VOTE      | 1            |
    | LIGHT_CLIENT_ATTACK | 2            |

### ConsensusParams

* **Fields**:

    | Name      | Type                                                          | Description                                                                  | Field Number |
    |-----------|---------------------------------------------------------------|------------------------------------------------------------------------------|--------------|
    | block     | [BlockParams](../core/data_structures.md#blockparams)         | Parameters limiting the size of a block and time between consecutive blocks. | 1            |
    | evidence  | [EvidenceParams](../core/data_structures.md#evidenceparams)   | Parameters limiting the validity of evidence of byzantine behaviour.         | 2            |
    | validator | [ValidatorParams](../core/data_structures.md#validatorparams) | Parameters limiting the types of public keys validators can use.             | 3            |
    | version   | [VersionsParams](../core/data_structures.md#versionparams)    | The ABCI application version.                                                | 4            |

### ProofOps

* **Fields**:

    | Name | Type                         | Description                                                                                                                                                                                                                  | Field Number |
    |------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
    | ops  | repeated [ProofOp](#proofop) | List of chained Merkle proofs, of possibly different types. The Merkle root of one op is the value being proven in the next op. The Merkle root of the final op should equal the ultimate root hash being verified against.. | 1            |

### ProofOp

* **Fields**:

    | Name | Type   | Description                                    | Field Number |
    |------|--------|------------------------------------------------|--------------|
    | type | string | Type of Merkle proof and how it's encoded.     | 1            |
    | key  | bytes  | Key in the Merkle tree that this proof is for. | 2            |
    | data | bytes  | Encoded Merkle proof for the key.              | 3            |

### Snapshot

* **Fields**:

    | Name     | Type   | Description                                                                                                                                                                       | Field Number |
    |----------|--------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
    | height   | uint64 | The height at which the snapshot was taken (after commit).                                                                                                                        | 1            |
    | format   | uint32 | An application-specific snapshot format, allowing applications to version their snapshot data format and make backwards-incompatible changes. CometBFT does not interpret this. | 2            |
    | chunks   | uint32 | The number of chunks in the snapshot. Must be at least 1 (even if empty).                                                                                                         | 3            |
    | hash     | bytes  | An arbitrary snapshot hash. Must be equal only for identical snapshots across nodes. CometBFT does not interpret the hash, it only compares them.                               | 4            |
    | metadata | bytes  | Arbitrary application metadata, for example chunk hashes or other verification data.                                                                                              | 5            |

* **Usage**:
    * Used for state sync snapshots, see the [state sync section](../p2p/legacy-docs/messages/state-sync.md) for details.
    * A snapshot is considered identical across nodes only if _all_ fields are equal (including
    `Metadata`). Chunks may be retrieved from all nodes that have the same snapshot.
    * When sent across the network, a snapshot message can be at most 4 MB.

## Data types introduced or modified in ABCI++

### VoteInfo

* **Fields**:

    | Name                        | Type                    | Description                                                    | Field Number |
    |-----------------------------|-------------------------|----------------------------------------------------------------|--------------|
    | validator                   | [Validator](#validator) | The validator that sent the vote.                              | 1            |
    | signed_last_block           | bool                    | Indicates whether or not the validator signed the last block.  | 2            |

* **Usage**:
    * Indicates whether a validator signed the last block, allowing for rewards based on validator availability.
    * This information is typically extracted from a proposed or decided block.



### ExtendedVoteInfo

* **Fields**:

    | Name              | Type                    | Description                                                                  | Field Number |
    |-------------------|-------------------------|------------------------------------------------------------------------------|--------------|
    | validator         | [Validator](#validator) | The validator that sent the vote.                                            | 1            |
    | signed_last_block | bool                    | Indicates whether or not the validator signed the last block.                | 2            |
    | vote_extension    | bytes                   | Reserved for future use. | 3            |

* **Usage**:
    * Indicates whether a validator signed the last block, allowing for rewards based on validator availability.
    * This information is extracted from CometBFT's data structures in the local process.
    * `vote_extension` is reserved for future use when vote extensions are added. Currently, this field is always set to `nil`.


### CommitInfo

* **Fields**:

    | Name  | Type                           | Description                                                                                  | Field Number |
    |-------|--------------------------------|----------------------------------------------------------------------------------------------|--------------|
    | round | int32                          | Commit round. Reflects the round at which the block proposer decided in the previous height. | 1            |
    | votes | repeated [VoteInfo](#voteinfo) | List of validators' addresses in the last validator set with their voting information.       | 2            |


### ExtendedCommitInfo

* **Fields**:

    | Name  | Type                                           | Description                                                                                                       | Field Number |
    |-------|------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|--------------|
    | round | int32                                          | Commit round. Reflects the round at which the block proposer decided in the previous height.                      | 1            |
    | votes | repeated [ExtendedVoteInfo](#extendedvoteinfo) | List of validators' addresses in the last validator set with their voting information, including vote extensions. | 2            |

<!--
### ExecTxResult

* **Fields**:

    | Name       | Type                                                        | Description                                                           | Field Number |
    |------------|-------------------------------------------------------------|-----------------------------------------------------------------------|--------------|
    | code       | uint32                                                      | Response code.                                                        | 1            |
    | data       | bytes                                                       | Result bytes, if any.                                                 | 2            |
    | log        | string                                                      | The output of the application's logger. **May be non-deterministic.** | 3            |
    | info       | string                                                      | Additional information. **May be non-deterministic.**                 | 4            |
    | gas_wanted | int64                                                       | Amount of gas requested for transaction.                              | 5            |
    | gas_used   | int64                                                       | Amount of gas consumed by transaction.                                | 6            |
    | events     | repeated [Event](abci++_basic_concepts.md#events)           | Type & Key-Value events for indexing transactions (e.g. by account).  | 7            |
    | codespace  | string                                                      | Namespace for the `code`.                                             | 8            |

-->

### ProposalStatus

```proto
enum ProposalStatus {
  UNKNOWN = 0; // Unknown status. Returning this from the application is always an error.
  ACCEPT  = 1; // Status that signals that the application finds the proposal valid.
  REJECT  = 2; // Status that signals that the application finds the proposal invalid.
}
```

* **Usage**:
    * Used within the [ProcessProposal](#processproposal) response.
        * If `Status` is `UNKNOWN`, a problem happened in the Application. CometBFT will assume the application is faulty and crash.
        * If `Status` is `ACCEPT`, the consensus algorithm accepts the proposal and will issue a Prevote message for it.
        * If `Status` is `REJECT`, the consensus algorithm rejects the proposal and will issue a Prevote for `nil` instead.

<!--
### VerifyStatus

```proto
enum VerifyStatus {
  UNKNOWN = 0; // Unknown status. Returning this from the application is always an error.
  ACCEPT  = 1; // Status that signals that the application finds the vote extension valid.
  REJECT  = 2; // Status that signals that the application finds the vote extension invalid.
}
```

* **Usage**:
    * Used within the [VerifyVoteExtension](#verifyvoteextension) response.
        * If `Status` is `UNKNOWN`, a problem happened in the Application. CometBFT will assume the application is faulty and crash.
        * If `Status` is `ACCEPT`, the consensus algorithm will accept the vote as valid.
        * If `Status` is `REJECT`, the consensus algorithm will reject the vote as invalid.


### CanonicalVoteExtension

>**TODO**: This protobuf message definition is not part of the ABCI++ interface, but rather belongs to the
> Precommit message which is broadcast via P2P. So it is to be moved to the relevant section of the spec.

* **Fields**:

    | Name      | Type   | Description                                                                                | Field Number |
    |-----------|--------|--------------------------------------------------------------------------------------------|--------------|
    | extension | bytes  | Vote extension provided by the Application.                                                | 1            |
    | height    | int64  | Height in which the extension was provided.                                                | 2            |
    | round     | int32  | Round in which the extension was provided.                                                 | 3            |
    | chain_id  | string | ID of the blockchain running consensus.                                                    | 4            |
    | address   | bytes  | [Address](../core/data_structures.md#address) of the validator that provided the extension | 5            |

* **Usage**:
    * CometBFT is to sign the whole data structure and attach it to a Precommit message
    * Upon reception, CometBFT validates the sender's signature and sanity-checks the values of `height`, `round`, and `chain_id`.
      Then it sends `extension` to the Application via `RequestVerifyVoteExtension` for verification.


-->

[protobuf-timestamp]: https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Timestamp
