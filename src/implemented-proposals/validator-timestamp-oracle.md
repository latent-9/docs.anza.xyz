---
title: Validator Timestamp Oracle
---

Third-party users of Solana sometimes need to know the real-world time a block
was produced, generally to meet compliance requirements for external auditors or
law enforcement. This proposal describes a validator timestamp oracle that
would allow a Solana cluster to satisfy this need.

The general outline of the proposed implementation is as follows:

- At regular intervals, each validator records its observed time for a known slot
  on-chain (via a Timestamp added to a slot Vote)
- A client can request a block time using the `getBlockTime` RPC method. Each
  Bank's Clock sysvar timestamp is calculated from the validator-provided
  timestamps (see [Bank Timestamp Correction](bank-timestamp-correction.md))
  and cached in Blockstore when the block is frozen; `getBlockTime` returns
  that cached value for rooted blocks, and the current Bank's Clock sysvar
  timestamp for blocks that have not yet been rooted.

Requirements:

- Any validator replaying the ledger in the future must come up with the same
  time for every block since genesis
- Estimated block times should not drift more than an hour or so before resolving
  to real-world (oracle) data
- The block times are not controlled by a single centralized oracle, but
  ideally based on a function that uses inputs from all validators
- Each validator must maintain a timestamp oracle

For blocks that have not yet been rooted, `getBlockTime` returns the current
Bank's Clock sysvar timestamp. This estimate is unstable until the block is
rooted, as the Clock sysvar timestamp of a not-yet-rooted Bank may still
change.

## Recording Time

At regular intervals as it is voting on a particular slot, each validator
records its observed time by including a timestamp in its Vote instruction
submission. The corresponding slot for the timestamp is the newest Slot in the
Vote vector (`Vote::slots.iter().max()`). It is signed by the validator's
identity keypair as a usual Vote. In order to enable this reporting, the Vote
struct needs to be extended to include a timestamp field, `timestamp: Option<UnixTimestamp>`, which will be set to `None` in most Votes.

As of https://github.com/solana-labs/solana/pull/10630, validators submit a
timestamp every vote. This enables implementation of a block time caching
service that allows nodes to calculate the estimated timestamp immediately after
the block is rooted, and cache that value in Blockstore. This provides
persistent data and quick queries, while still meeting requirement 1) above.

### Vote Accounts

A validator's vote account will hold its most recent slot-timestamp in VoteState.

### Vote Program

The on-chain Vote program needs to be extended to process a timestamp sent with
a Vote instruction from validators. In addition to its current process_vote
functionality (including loading the correct Vote account and verifying that the
transaction signer is the expected validator), this process needs to compare the
timestamp and corresponding slot to the currently stored values to verify that
they are both monotonically increasing, and store the new slot and timestamp in
the account.

## Calculating Block Times

Each Bank's Clock sysvar timestamp is corrected on every new Bank using the
validator-provided timestamps, as described in
[Bank Timestamp Correction](bank-timestamp-correction.md): the runtime
calculates a stake-weighted median of the active validators' timestamp
estimates and bounds it so that it cannot drift too far from the theoretical
estimate.

When a Bank is frozen, its Clock sysvar timestamp is cached in Blockstore.
Under Alpenglow, the Clock sysvar timestamp is instead set by the block
producer in the block footer, within bounds that validators enforce.
