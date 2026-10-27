# Withdrawal no-prose guards

Calls beginning with `GW-` in the withdrawal no-prose diagrams expand to the
exact conditions below. `local_height` is the receiver's current block height.

## Shared checks

`GW-MILESTONE(local_height)` requires:

```text
local_height >= boomerang_params.boomerang_descriptor.milestone_block_0
```

`GW-DURESS-SELECTION(selection, columns)` requires:

```text
count(selection) == count(columns)
count(selection) == 5
selection contains exactly one displayed choice from each columns column
each selection value exists in its corresponding columns column
```

`GW-DURESS-INDEXES(indices)` requires:

```text
count(indices) == 5
indices are distinct
indices are in range [1..193]
```

`GW-DURESS-RESPONSE(response, nonce)` requires:

```text
an outstanding duress check exists for the current phase
response.nonce == nonce
GW-DURESS-INDEXES(response.content.indices)
```

`GW-SAR-PLACEHOLDER(safe, valid_duress)` requires `safe || valid_duress`.

## Approval freshness

`GW-INIT-APPROVAL-FRESH(local_height, initiator_approval, wt_approval)` requires:

```text
initiator_approval.event_block_height >=
  wt_approval.event_block_height - TOLERANCE_IN_BLOCKS_FROM_TX_APPROVAL_BY_INITIATOR_PEER_TO_TX_APPROVAL_BY_WT
initiator_approval.event_block_height <= min(
  local_height,
  wt_approval.event_block_height)
```

`GW-WT-APPROVAL-FRESH(local_height, initiator_approval, wt_approval)` requires:

```text
wt_approval.event_block_height >= max(
  initiator_approval.event_block_height,
  local_height - TOLERANCE_IN_BLOCKS_FROM_TX_APPROVAL_BY_WT_TO_RECEIVING_WT_TX_APPROVAL_BY_NON_INITIATOR_PEERS)
wt_approval.event_block_height <= min(
  initiator_approval.event_block_height + TOLERANCE_IN_BLOCKS_FROM_TX_APPROVAL_BY_INITIATOR_PEER_TO_TX_APPROVAL_BY_WT,
  local_height)
```

`GW-NONINIT-APPROVAL-AT-WT-FRESH(local_height, wt_approval, peer_approval)`
requires:

```text
peer_approval.event_block_height >= max(
  local_height - TOLERANCE_IN_BLOCKS_FROM_TX_APPROVAL_BY_NON_INITIATOR_PEER_TO_RECEIVING_NON_INITIATOR_PEERS_TX_APPROVAL_BY_WT,
  wt_approval.event_block_height)
peer_approval.event_block_height <= local_height
```

`GW-NONINIT-APPROVAL-AT-PEER-FRESH(local_height, wt_approval, peer_approval)`
requires:

```text
peer_approval.event_block_height >= max(
  wt_approval.event_block_height,
  local_height - TOLERANCE_IN_BLOCKS_FROM_TX_APPROVAL_BY_NON_INITIATOR_PEERS_TO_RECEIVING_NON_INITIATOR_TX_APPROVAL_BY_OTHER_NON_INITIATOR_PEERS)
peer_approval.event_block_height <= min(
  local_height,
  wt_approval.event_block_height + TOLERANCE_IN_BLOCKS_FROM_TX_APPROVAL_BY_WT_TO_RECEIVING_NON_INITIATOR_TX_APPROVAL_BY_OTHER_NON_INITIATOR_PEERS)
wt_approval.event_block_height >=
  local_height - TOLERANCE_IN_BLOCKS_FROM_TX_APPROVAL_BY_WT_TO_RECEIVING_NON_INITIATOR_TX_APPROVAL_BY_OTHER_NON_INITIATOR_PEERS
```

## Commitment freshness

`GW-INIT-COMMIT-AT-NONINIT-FRESH(local_height, commit)` requires:

```text
commit.event_block_height >= local_height - TOLERANCE_IN_BLOCKS_FROM_TX_COMMITMENT_BY_INITIATOR_PEER_TO_RECEIVING_INITIATOR_PEER_TX_COMMITMENT_BY_NON_INITIATOR_PEERS
commit.event_block_height <= local_height
```

`GW-NONINIT-COMMIT-AT-WT-FRESH(local_height, commit)` requires:

```text
commit.event_block_height >= local_height - TOLERANCE_IN_BLOCKS_FROM_TX_COMMITMENT_BY_NON_INITIATOR_PEER_TO_RECEIVING_NON_INITIATOR_PEERS_TX_COMMITMENT_BY_WT_HAVING_SAR_RESPONSE_BACK_TO_WT
commit.event_block_height <= local_height
```
