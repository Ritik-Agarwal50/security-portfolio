# Execution Fee Loss Due to Premature Data Deletion

### Protocol: Gamma Protocol  
### Severity: High  
### Status: ✅ Valid

---

## 🔍 Description

During the execution of the `PerpetualVault::_handleReturn()` function, the contract burns the user’s shares and deletes their `depositInfo` mapping before processing execution fee refunds.

Specifically, `_burn(depositId)` deletes the `depositInfo[depositId]` entry, which contains the user’s `executionFee` data. Immediately after, the refund logic attempts to compare `depositInfo[depositId].executionFee` to the used fee (`usedFee`), but since the mapping entry is already deleted, this check never passes.

As a result, any excess execution fee that should be refunded to the user remains locked, causing a loss of funds.

---

## 🧠 Impact

- Users are unable to reclaim the excess execution fee when withdrawing.
- Execution fees meant to be refunded remain stuck inside the contract.
- This leads to direct user fund loss and damages trust in the protocol.

---

## 🛠️ Code Reference

```solidity
function _handleReturn(uint256 withdrawn, bool positionClosed, bool refundFee) internal {
    ...
    emit Burned(depositId, depositInfo[depositId].recipient, depositInfo[depositId].shares, amount);
    _burn(depositId);

    if (refundFee) {
        uint256 usedFee = callbackGasLimit * tx.gasprice;
        if (depositInfo[depositId].executionFee > usedFee) { // This condition fails since depositInfo is deleted
            try IGmxProxy(gmxProxy).refundExecutionFee(
                depositInfo[counter].owner, 
                depositInfo[counter].executionFee - usedFee
            ) {} catch {}
        }
    }
    ...
}

function _burn(uint256 depositId) internal {
    EnumerableSet.remove(userDeposits[depositInfo[depositId].owner], depositId);
    totalShares = totalShares - depositInfo[depositId].shares;
    delete depositInfo[depositId]; // Premature deletion causing the bug
}
