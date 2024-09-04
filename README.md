# Issue H-1: Users will lock raffle prizes on the `WinnablesPrizeManager` contract by calling `WinnablesTicketManager::propagateRaffleWinner` with wrong CCIP inputs 

Source: https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/50 

## Found by 
0rpse, 0x0bserver, 0x73696d616f, 0xAadi, 0xbrivan, 0xrex, CatchEmAll, DrasticWatermelon, Feder, Galturok, IMAFVCKINSTARRRRRR, KungFuPanda, Oblivionis, Offensive021, Oxsadeeq, PNS, PTolev, Paradox, Penaldo, PeterSR, S3v3ru5, SadBase, SovaSlava, Trooper, Waydou, akiro, araj, dany.armstrong90, dimulski, dinkras\_, durov, dy, gajiknownnothing, iamnmt, irresponsible, jennifer37, joshuajee, matejdb, neko\_nyaa, ogKapten, philmnds, rsam\_eth, sakshamguruji, shaflow01, shikhar, tofunmi, turvec, utsav
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

# Issue H-2: Attacker will prevent any raffles by calling `WinnablesTicketManager::cancelRaffle` before admin starts raffle 

Source: https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/57 

## Found by 
0rpse, 0x0bserver, 0x73696d616f, 0xAadi, 0xShahilHussain, 0xarno, 0xbrivan, AllTooWell, BitcoinEason, Bluedragon, CatchEmAll, KaligoAudits, Oblivionis, Offensive021, PNS, PTolev, Paradox, S3v3ru5, Silvermist, TessKimy, Trident-Audits, ZC002, aman, araj, charles\_\_cheerful, denzi\_, dimi6oni, dimulski, dinkras\_, dobrevaleri, durov, eeshenggoh, frndz0ne, iamnmt, jennifer37, neko\_nyaa, ogKapten, p0wd3r, philmnds, phoenixv110, rsam\_eth, sakshamguruji, shaflow01, shikhar, shui, tjonair, utsav, vinica\_boy, y4y
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

# Issue H-3: Method refundPlayers doesn't update _lockedETH in WinnableTicketManager 

Source: https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/138 

## Found by 
0x0bserver, 0x6a70, 0x73696d616f, 0xShahilHussain, 0xbrivan, 0xjarix, 4gontuk, Afriaudit, AllTooWell, AuditorPraise, BlocSoc\_Audits, CatchEmAll, Galturok, GenevaRoc, IvanFitro, MSK, Offensive021, Paradox, Penaldo, PeterSR, S3v3ru5, Silvermist, TessKimy, Waydou, almurhasan, anonymousjoe, araj, charles\_\_cheerful, dany.armstrong90, dimi6oni, dimulski, dobrevaleri, gajiknownnothing, gerd, gkrastenov, iamnmt, ihtishamsudo, ironside, irresponsible, matejdb, neko\_nyaa, ni8mare, oxelmiguel, oxwhite, p0wd3r, pandasec, pashap9990, phoenixv110, sakshamguruji, shikhar, shui, utsav, vinica\_boy, y4y
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

# Issue M-1: The setRole() function grants role instead of removing 

Source: https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/53 

## Found by 
0x0x0xw3, 0xAadi, 0xarno, 0xbrivan, 0xnolo, Afriaudit, DrasticWatermelon, Drynooo, Galturok, KaligoAudits, KingNFT, MaanVader, MrPotatoMagic, PTolev, Paradox, Penaldo, PeterSR, PratRed, Saurabh\_Singh, Trooper, UAARRR, araj, azanux, boringslav, casper, chaduke, charles\_\_cheerful, dany.armstrong90, denzi\_, dimulski, dwaipayan01, iamnmt, ironside, joshuajee, ke1caM, korok, mahmud, matejdb, neko\_nyaa, nilay27, petro1912, roguereggiant, sharonphiliplima, shikhar, thisvishalsingh, unnamed, utsav, ydlee
### Summary

Access control in the Winnables Raffles protocol is handled with the `Roles` contract. It works similarly to OpenZeppelin's access control but uses bit flags to determine whether a user has a role. Each user has a bytes32 representing the bitfield of roles. Role `0` is an admin role, allowing its members to grant or deny(remove) roles to other users.

The `setRole(address user, uint8 role, bool status)` function, as it stands, always adds a role by performing a bitwise OR operation. However, it does not handle the removal of roles if the `status` parameter is `false`. This oversight results in incorrect role management within the contracts, potentially leading to accidental privilege grants or the inability to revoke privileges from compromised or revoked accounts.

### Root Cause

In `Roles.sol:L29` the `_setRole()` function always adds a role by performing a bitwise OR operation:

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/Roles.sol#L29-L33

This internal function is used in the `setRole()` function:

https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/Roles.sol#L35-L37

### Internal pre-conditions

The `setRole()` function can only be called by the `Admin`.

### External pre-conditions

_No response_

### Attack Path

1. `Admin` deploys the `WinnablesTicketTest` contract.
2. `Admin` grants role `1` to `Alice` by calling the `setRole()` function of the `WinnablesTicketTest` contract. The role is granted to `Alice`.
3. `Alice` mints 10 tickets to `Bob` using the role.
4. `Admin` revokes role `1` from `Alice` by calling the `setRole()` function.
5. The role is not removed from `Alice`. She can still mint tickets to `Bob`.

### Impact

The improper implementation results in incorrect role management within the contracts, potentially leading to accidental privilege grants or the inability to revoke privileges from compromised or revoked accounts.

### PoC

```javascript
describe('Ticket behaviour', () => {
...
    it('Should not be able to mint tickets afer role deny', async () => {
      await (await ticket.setRole(signers[2].address, 1, true)).wait();

      const { events } = await (await ticket.connect(signers[2]).mint(signers[3].address, 1, 1)).wait();
      expect(events).to.have.lengthOf(2);
      expect(events[0].event).to.eq('NewTicket');
      expect(events[1].event).to.eq('TransferSingle');
      
      await (await ticket.setRole(signers[2].address, 1, false)).wait();
      await expect(ticket.connect(signers[2]).mint(signers[2].address, 1, 1)).to.be.revertedWithCustomError(
        ticket,
        'MissingRole'
      );
    });
...
```

### Mitigation

Modify the `_setRole()` function to handle both adding and removing roles based on the status parameter:

```solidity
function _setRole(address user, uint8 role, bool status) internal virtual {
    uint256 roles = uint256(_addressRoles[user]);
    if (status) {
        _addressRoles[user] = bytes32(roles | (1 << role));
    } else {
        _addressRoles[user] = bytes32(roles & ~(1 << role));
    }
    emit RoleUpdated(user, role, status);
}
```

# Issue M-2: Incorrect casting in `BaseCCIPContract::_packCCIPContract()` 

Source: https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/72 

## Found by 
0x73696d616f, 0xbrivan, 4b, CatchEmAll, MinhTriet, PTolev, Pataroff, Penaldo, PeterSR, TessKimy, Trooper, anonymousjoe, azanux, denzi\_, dwaipayan01, nilay27, petro1912, rbserver, roguereggiant
### Summary

The style of casting the `chainSelector` is incorrect hence it will evaluate to zero and not form part of the bytes returned.

### Root Cause

In [`BaseCCIPContract::_packCCIPContract()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/BaseCCIPContract.sol#L40), the [`chainSelector()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/BaseCCIPContract.sol#L43) was casted to uint256 after the bitwise operation which is not supposed to be so

### Internal pre-conditions

whenever the function params are set it will occur

### External pre-conditions

There are no preconditions, anytime it is called by another function it will occur.

### Attack Path

This is an internal casting error so whenever another function calls `BaseCCIPContract::_packCCIPContract()` it will occur.

### Impact

The `_packCCIPContract()` returns an incorrect value since the `chainSelector` will not be part of the encoded bytes. 

### PoC

Code with the casting error
```solidity
    function _packCCIPContract(address contractAddress, uint64 chainSelector) internal pure returns(bytes32) { return bytes32(
            uint256(uint160(contractAddress)) | uint256(chainSelector << 160));
    }
```

these are the test results after supplying it with random values
```solidity
[PASS] test() (gas: 5678)
Traces:
  [5678] returnTest::test_packCCIPContract()
    ├─ [427] BitwiseOperations::_packCCIPContract(returnTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 43114 [4.311e4]) [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000007fa9385be102ac3eac297483dd6233d62b3e1496
    └─ ← [Stop] 

Suite result: ok. 1 passed;
```
we can observe that the test passed but only the address has been encoded

code with the right casting style
```solidity
    function CCIP(address contractAddress, uint64 chainSelector) public pure returns(bytes32) {
        return bytes32(
            uint256(uint160(contractAddress)) |
            (uint256(chainSelector) << 160)
        );
    }
```

these are the test results
```solidity
[PASS] testCCIP() (gas: 5705)
Traces:
  [5705] returnTest::test_packCCIPContract()
    ├─ [454] BitwiseOperations::_packCCIPContract(returnTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 43114 [4.311e4]) [staticcall]
    │   └─ ← [Return] 0x00000000000000000000a86a7fa9385be102ac3eac297483dd6233d62b3e1496
    └─ ← [Stop] 

Suite result: ok. 1 passed;
```
from this test result we can observe that the chainSelector has been encoded to the bytes32 returned and not only the address

### Mitigation

Instead of casting after the bitwise operation, cast before the operation
```diff
    function _packCCIPContract(address contractAddress, uint64 chainSelector) internal pure returns(bytes32) {
        return bytes32(
-            uint256(uint160(contractAddress)) | uint256(chainSelector << 160));
+            uint256(uint160(contractAddress)) | uint256(chainSelector) << 160);
    }

```



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Winnables/public-contracts/pull/7


# Issue M-3: `CCIPClient` `whenHealthy` modifier will lead to stuck `ETH` due to DoSing claim and cancel 

Source: https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/236 

## Found by 
0x73696d616f, Waydou
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

