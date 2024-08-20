Mammoth Stone Grasshopper

High

# Attacker will prevent any raffles by calling `WinnablesTicketManager::cancelRaffle` before admin starts raffle

### Summary

The [`WinnablesTicketManager::cancelRaffle`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L278) function is vulnerable to abuse because it is an external function that allows anyone to cancel a raffle if its status is set to PRIZE_LOCKED. An attacker could exploit this by repeatedly calling `cancelRaffle` whenever a new raffle is available to be started, effectively preventing any raffles from ever being initiated.

### Root Cause

The root cause of this issue lies in the design of the function:
1. The function is external, meaning it can be called by anyone.
2. When called, it checks the underlying function `WinnablesTicketManager::_checkShouldCancel`, which allows cancellation of a raffle if the [status is PRIZE_LOCKED](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L436), which is a temporary state before the admin calls `WinnablesTicketManager::createRaffle`.
3. This opens up a window of opportunity for an attacker to cancel the raffle before it transitions to an active state.

### Internal pre-conditions

There must be a raffleId with raffleStatus == PRIZE_LOCKED


### External pre-conditions

The attacker must monitor the contract to identify when a raffle is in the PRIZE_LOCKED state, which occurs after the admin locks a prize in the `WinnablesPrizeManager` contract.
The attacker must call the `WinnablesTicketManager::cancelRaffle` before the admin calls `WinnablesTicketManager::createRaffle`.


### Attack Path

1. An attacker monitors the contract to detect when a new raffle enters the PRIZE_LOCKED state.
2. As soon as the raffle reaches this state, the attacker calls the cancelRaffle function.
3. The raffle is canceled before it can transition to an active state, preventing it from starting.
4. The attacker can repeat this process for each new raffle, effectively blocking the initiation of all raffles.


### Impact

This vulnerability allows a malicious actor to disrupt the entire raffle system. By preventing any raffles from starting, the attacker can undermine the functionality of the whole protocol.


### PoC

The test below, which can be added to the hardhat test suite, shows that a random user can cancel the raffle if it hasn't yet been started

```javascript
  describe('Buyer can cancel raffle', () => {
    before(async () => {
      snapshot = await helpers.takeSnapshot();
    });

    after(async () => {
      await snapshot.restore();
    });
    const buyers = [];

    it('Should be able to cancel a raffle', async () => {
      const now = await blockTime();
      const buyer = await getWalletWithEthers();
      await (await link.mint(manager.address, ethers.utils.parseEther('100'))).wait();
      const tx = await manager.connect(buyer).cancelRaffle(counterpartContractAddress, 1, 1);
      const { events } = await tx.wait();
      expect(events).to.have.lengthOf(3);
      const ccipMessageEvent = ccipRouter.interface.parseLog(events[0]);
      expect(ccipMessageEvent.name).to.eq('MockCCIPMessageEvent');
      expect(ccipMessageEvent.args.data).to.eq('0x000000000000000000000000000000000000000000000000000000000000000001');
      await expect(manager.getWinner(1)).to.be.revertedWithCustomError(manager, 'RaffleNotFulfilled');
    });
  });
```

### Mitigation

This vulnerability can be mitigated by updating the underlying function `WinnablesTicketManager::_checkShouldCancel` to only allow the admin to cancel a raffle that hasn't started yet.