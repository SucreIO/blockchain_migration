# UMP Module

## Storage

General storage entries

```rust
/// Paras that are to be cleaned up at the end of the session.
/// The entries are sorted ascending by the para id.
OutgoingParas: Vec<ParaId>;
```

Storage related to UMP

```rust
/// The messages waiting to be handled by the relay-chain originating from a certain parachain.
///
/// Note that some upward messages might have been already processed by the inclusion logic. E.g.
/// channel management messages.
///
/// The messages are processed in FIFO order.
RelayDispatchQueues: map ParaId => Vec<UpwardMessage>;
/// Size of the dispatch queues. Caches sizes of the queues in `RelayDispatchQueue`.
///
/// First item in the tuple is the count of messages and second
/// is the total length (in bytes) of the message payloads.
///
/// Note that this is an auxilary mapping: it's possible to tell the byte size and the number of
/// messages only looking at `RelayDispatchQueues`. This mapping is separate to avoid the cost of
/// loading the whole message queue if only the total size and count are required.
///
/// Invariant:
/// - The set of keys should exactly match the set of keys of `RelayDispatchQueues`.
RelayDispatchQueueSize: map ParaId => (u32, u32); // (num_messages, total_bytes)
/// The ordered list of `ParaId`s that have a `RelayDispatchQueue` entry.
///
/// Invariant:
/// - The set of items from this vector should be exactly the set of the keys in
///   `RelayDispatchQueues` and `RelayDispatchQueueSize`.
NeedsDispatch: Vec<ParaId>;
/// This is the para that gets dispatched first during the next upward dispatchable queue
/// execution round.
///
/// Invariant:
/// - If `Some(para)`, then `para` must be present in `NeedsDispatch`.
NextDispatchRoundStartWith: Option<ParaId>;
```


## Initialization

No initialization routine runs for this module.

## Routines

Candidate Acceptance Function:

* `check_upward_messages(P: ParaId, Vec<UpwardMessage>`):
    1. Checks that there are at most `config.max_upward_message_num_per_candidate` messages.
    1. Checks that no message exceeds `config.max_upward_message_size`.
    1. Verify that `RelayDispatchQueueSize` for `P` has enough capacity for the messages

Candidate Enactment:

* `enact_upward_messages(P: ParaId, Vec<UpwardMessage>)`:
    1. Process each upward message `M` in order:
        1. Append the message to `RelayDispatchQueues` for `P`
        1. Increment the size and the count in `RelayDispatchQueueSize` for `P`.
        1. Ensure that `P` is present in `NeedsDispatch`.

The following routine is intended to be called in the same time when `Paras::schedule_para_cleanup` is called.

`schedule_para_cleanup(ParaId)`:
    1. Add the para into the `OutgoingParas` vector maintaining the sorted order.

The following routine is meant to execute pending entries in upward message queues. This function doesn't fail, even if
dispatcing any of individual upward messages returns an error.

`process_pending_upward_messages()`:
    1. Initialize a cumulative weight counter `T` to 0
    1. Iterate over items in `NeedsDispatch` cyclically, starting with `NextDispatchRoundStartWith`. If the item specified is `None` start from the beginning. For each `P` encountered:
        1. Dequeue the first upward message `D` from `RelayDispatchQueues` for `P`
        1. Decrement the size of the message from `RelayDispatchQueueSize` for `P`
        1. Delegate processing of the message to the runtime. The weight consumed is added to `T`.
        1. If `T >= config.preferred_dispatchable_upward_messages_step_weight`, set `NextDispatchRoundStartWith` to `P` and finish processing.
        1. If `RelayDispatchQueues` for `P` became empty, remove `P` from `NeedsDispatch`.
        1. If `NeedsDispatch` became empty then finish processing and set `NextDispatchRoundStartWith` to `None`.
        > NOTE that in practice we would need to approach the weight calculation more thoroughly, i.e. incorporate all operations
        > that could take place on the course of handling these upward messages.

## Session Change

1. Drain `OutgoingParas`. For each `P` happened to be in the list:.
    1. Remove `RelayDispatchQueueSize` of `P`.
    1. Remove `RelayDispatchQueues` of `P`.
    1. Remove `P` if it exists in `NeedsDispatch`.
    1. If `P` is in `NextDispatchRoundStartWith`, then reset it to `None`
    - Note that if we don't remove the open/close requests since they are going to die out naturally at the end of the session.