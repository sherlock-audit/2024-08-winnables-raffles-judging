Agreeable Wooden Unicorn

High

# Method refundPlayers doesn't update _lockedETH in WinnableTicketManager

### Summary

The variable `_lockedETH` keeps track of the ETH(AVAX) collected by the raffles that are underway. The owner can't withdraw this amount. If a raffle is cancelled then users get to withdraw their ETH(AVAX) paid to buy tickets. But the `_lockedETH` is not updated. So in the future raffle which do gets completed the owner is supposed to get the ticket amount. But since the `_lockedETH` from previously wasn't set to 0 it having some value leads to that much amount getting stuck in the contract forever.

### Root Cause

In `https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L215-L228` the refunded amount should've been subtracted from  `_lockedETH` amount. Since it's not updated the owner will not be able to withdraw this much amount ever `https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L300-L306`

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

1. A new raffle starts. Alice buys tickets worth of 2 ETH(AVAX). The `_lockedETH` is updated from 0 to 2.
2. Bob buys tickets worth of 1 ETH(AVAX). The `_lockedETH` is updated from 2 to 3.
3. Now the Raffle gets cancelled because of some reason. Both Alice and Bob withdraws their money `refundPlayers()`
4. In Future another raffle starts. Both Alice and Bob stakes 2 ETH each. The `_lockedETH` gets updated to 4 + 3 = 7.
5. The Raffle finishes and winner is chosen. The `propagateRaffleWinner()` updated `_lockedETH` to 7-4 = 3.
6. Now when the Owner tries to withdraw the payment which was 4 ETH it reverts since the `_lockedETH` is 3. So the owner is only allowed to withdraw 1ETH. Rest of the 3ETH will be stuck in the contract.

### Impact

It leads to locking of ETH(AVAX) in the contract forever that was protocol income.

### PoC

Add the following snippet in `/test/TicketManager.js`
```javascript
it('Should be able to refund tickets purchased', async () => {
  const contractBalanceBefore = await ethers.provider.getBalance(manager.address);
  const userBalanceBefore = await ethers.provider.getBalance(buyer2.address);
  let lockedETH = await manager.getLockedEth()
  console.log("Locked ETH is: ", lockedETH)
  const tx = await manager.refundPlayers(1, [buyer2.address]);
  lockedETH = await manager.getLockedEth()
  console.log("Locked ETH after player unlock is: ", lockedETH)
  const { events } = await tx.wait();
  expect(events).to.have.lengthOf(1);
  const [ event ] = events;
  expect(event.event).to.equal('PlayerRefund');
  const contractBalanceAfter = await ethers.provider.getBalance(manager.address);
  const userBalanceAfter = await ethers.provider.getBalance(buyer2.address);
  expect(contractBalanceAfter).to.eq(contractBalanceBefore.sub(100));
  expect(userBalanceAfter).to.eq(userBalanceBefore.add(100));
  const { withdrawn } = await manager.getParticipation(1, buyer2.address);
  expect(withdrawn).to.eq(true);
});
```
### Output:
```log
Locked ETH is:  BigNumber { value: "100" }
Locked ETH after player unlock is:  BigNumber { value: "100" }
```

### Mitigation

Update the _lockedETH variable in `refundPlayers()` as below:
```solidity
function refundPlayers(uint256 raffleId, address[] calldata players) external {
    Raffle storage raffle = _raffles[raffleId];
    if (raffle.status != RaffleStatus.CANCELED) revert InvalidRaffle();
    for (uint256 i = 0; i < players.length; ) {
        address player = players[i];
        uint256 participation = uint256(raffle.participations[player]);
        if (((participation >> 160) & 1) == 1) revert PlayerAlreadyRefunded(player);
        raffle.participations[player] = bytes32(participation | (1 << 160));
        uint256 amountToSend = (participation & type(uint128).max);
        _lockedETH -= amountToSend;
        _sendETH(amountToSend, player);
        emit PlayerRefund(raffleId, player, bytes32(participation));
        unchecked { ++i; }
    }
}
```