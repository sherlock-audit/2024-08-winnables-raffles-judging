Huge Fossilized Turtle

Medium

# `CCIPClient` `whenHealthy` modifier will lead to stuck `ETH` due to DoSing claim and cancel

### Summary

`CCIPClient` has a `whenHealthy` modifier in the `ccipSend()` function, which means it can DoS `_sendCCIPMessage()` calls in `WinnablesTicketManager`. This would be particularly harmful in several scenarios:
1. In case a raffle does not meet the minimum tickets threshold, it must be canceled. However, cancelling sets the status to `CANCELED` and allows users to claim refunds, but also sends a message to `WinnablesPrizeManager` to allow the admins to get their funds back. If the router is not healthy, it will revert. This procedure should be perfomed in a 2 step such that users can get their refunds right away, as they don't need to wait for the ccip router to work.
2. Users buy tickets but the router is DoSed and `WinnablesTicketManager::propagateRaffleWinner()` reverts when calling `_sendCCIPMessage()`. This means that the protocol can never claim its `ETH` although the cross chain message was not required to be successful. A two step procedure would also fix this.

Scenario 1 breaks the specification in the [readme](https://github.com/sherlock-audit/2024-08-winnables-raffles-0xsimao/tree/main?tab=readme-ov-file#q-please-discuss-any-design-choices-you-made)
> Participants in a raffle that got cancelled can always get refunded

### Root Cause

The Chainlink [Router](https://vscode.blockscan.com/ethereum/0x80226fc0Ee2b096224EeAc085Bb9a8cba1146f7D) has the `whenHealthy` modifier in `ccipSend()`, called in `_sendCCIPMessage()`, which DoSes the router as can be seen in the code linked above in lines 293-296.

`WinnablesTicketManager` does not deal with the `notHealthy` modifier.

### Internal pre-conditions

None.

### External pre-conditions

Chainlink pauses the Router.

### Attack Path

The examples are given:
**A**
1. Users participate by calling `WinnablesTicketManager::buyTickets()`.
2. Not enough tickets were bought so the raffle should be canceled, but Chainlink DoSes the router.
3. [WinnablesTicketManager::cancelRaffle()](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L282-L286) calls the Chainlink router to send a message, but it reverts due to the modifier. Users can not get their refunds back until the Chainlink router is back up.

**B**

1. Users participate by calling `WinnablesTicketManager::buyTickets()`.
2. Chainlink DoSes the router after the raffle ends, DoSing [WinnablesTicketManager::propagateRaffleWinner()](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L340).
3. The protocol can not claim the locked `ETH` due to point 2 even though the cross chain message was not required.

### Impact

In scenario **A**, users can not claim their refunds until the router is back up.
In **B**, the protocol can not claim the ETH back even though it could be safely retrieved.

### PoC

Check the mentioned Chainlink router links and the fact that the code never checks if the router is not healthy before calling `_sendCCIPMessage()`.

### Mitigation

The `WinnablesTicketManager::cancelRaffle()` and `WinnablesTicketManager::propagateRaffleWinner()` functions should be split into 2 separate steps, to always make sure users or the protocol can get their funds.