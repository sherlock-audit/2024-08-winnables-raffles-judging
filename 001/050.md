Mammoth Stone Grasshopper

High

# Users will lock raffle prizes on the `WinnablesPrizeManager` contract by calling `WinnablesTicketManager::propagateRaffleWinner` with wrong CCIP inputs

### Summary

The [`WinnablesTicketManager::propagateRaffleWinner`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L334) function is vulnerable to misuse, where incorrect CCIP inputs can lead to assets being permanently locked in the `WinnablesPrizeManager` contract. The function does not have input validation for the `address prizeManager` and `uint64 chainSelector` parameters. If called with incorrect values, it will fail to send the message to `WinnablesPrizeManager`, resulting in the assets not being unlocked.


### Root Cause

The root cause of the issue lies in the design of the `propagateRaffleWinner` function:
1. The function is responsible for sending a message to WinnablesPrizeManager to unlock the raffle assets.
2. The function is marked as external, so anyone can call it.
3. The function receives `address prizeManager` and `uint64 chainSelector` as inputs, which are responsible for sending the message to the `WinnablesPrizeManager` contract for it to unlock the assets previously locked for the raffle.
4. The inputs forementioned are not validated, meaning users can call the function with wrong values.
5. This cannot be undone, as the function [changes the state of the raffle](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L337) in a way that [prevents the function from being called again](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L336).


### Internal pre-conditions

A raffle must have been won by a player.


### External pre-conditions

A user must call `WinnablesTicketManager::propagateRaffleWinner` with incorrect input values.


### Attack Path

1. A user wins the raffle.
2. Some user calls `WinnablesTicketManager::propagateRaffleWinner` and provides incorrect inputs for prizeManager and chainSelector.
3. The propagateRaffleWinner function fails to send the correct message to WinnablesPrizeManager due to the parameter mismatch.
4. As a result, the assets associated with the raffle remain locked and cannot be retrieved by the raffle winner.


### Impact

This vulnerability completely disrupts the protocol, as it becomes impossible to retrieve the reward of the raffle.


### PoC

The test below, which is an edited version of [this existing test](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/test/TicketManager.js#L786), shows that the function call will be successful with a random chainSelector

```javascript
    it('Should be able to propagate when the winner is drawn', async () => {
@>    const { events } = await (await manager.propagateRaffleWinner(counterpartContractAddress, 9846, 1)).wait();
      expect(events).to.have.lengthOf(3);
      const ccipEvent = ccipRouter.interface.parseLog(events[0]);
      expect(ccipEvent.args.receiver).to.eq('0x' + counterpartContractAddress.toLowerCase().slice(-40).padStart(64, '0'));
      expect(ccipEvent.args.data).to.have.lengthOf(108);
      const drawnWinner = ethers.utils.getAddress('0x' + ccipEvent.args.data.slice(-40));
      expect(buyers.find(b => b.address === drawnWinner)).to.not.be.undefined;
      expect(ccipEvent.args.data.slice(0, 68)).to.eq('0x010000000000000000000000000000000000000000000000000000000000000001');
    });
```

### Mitigation

Implement input validation to ensure that `prizeManager` and `chainSelector` are correct before proceeding with the propagation.
