Faithful Lemonade Penguin

Medium

# Admin can prevent users from claiming their rewards if the prize token is LINK

## Summary
An admin can unintentionally or intentionally prevent a winner from claiming their LINK token prize by allowing the contract's LINK balance to drop below the locked amount, violating the rule that winners must always be able to withdraw their rewards

## Vulnerability Detail
According to the [rules of this contest](https://audits.sherlock.xyz/contests/516):
> The principles that must always remain true are:
> - Winnables admins cannot do anything to prevent a winner from withdrawing their prize

However, if the prize token of a Raffle is the LINK token, an admin can, either intentionally or unintentionally, prevent the winner from claiming their reward. This occurs because the `_sendCCIPMessage` function reduces the fee from the amount that is locked for the Raffle, resulting in `linkToken.balanceOf(address(this)) < _tokensLocked[LINK]` and ultimately preventing the winner from claiming their reward.
```solidity
        uint256 fee = router.getFee(
            ccipDestChainSelector,
            message
        );
        uint256 currentLinkBalance = linkToken.balanceOf(address(this));

        if (fee > currentLinkBalance) {
            revert InsufficientLinkBalance(currentLinkBalance, fee);
        }
```

For better understanding, let's consider an example scenario:

- The admin transfers some LINK (let's say 2 LINK) to the `WinnablesPrizeManager.sol` to pay for fees.
- The admin locks 100 LINK to create a Raffle (the prize token for this Raffle is LINK).
- The admin also creates other Raffles, locking more NFTs, ETH, and tokens, which requires paying additional fees.
- These transactions result in 3 LINK being spent on fees—2 LINK from the amount that was deposited to pay for the fees plus 1 LINK deducted from the 100 LINK that was locked for the Raffle.
- Now, although the Raffle prize was initially set at 100 $LINK, the contract balance is only 99 LINK.
- When the winner calls the `claim` function, their transaction reverts because there is not enough LINK available to send to the winner.

An admin can do the above scenario either intentionally or unintentionally.

Despite the assumption that:

> Admins will always keep LINK in the contracts (to pay for CCIP Messages),

The implemented logic in the protocol allows an admin to prevent the winner from claiming their reward, which contradicts one of the principles:

> Winnables admins cannot do anything to prevent a winner from withdrawing their prize.

## Impact
Admin can prevent winners from getting their rewards which contradicts the principle:
 - Winnables admins cannot do anything to prevent a winner from withdrawing their prize.

## Code Snippet
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/BaseCCIPSender.sol#L42

## Tool used

Manual Review

## Recommendation
Do not deduct fee from the prize amount:
```diff
    function _sendCCIPMessage(
        address ccipDestAddress,
        uint64 ccipDestChainSelector,
        bytes memory data,
+       uint256 lockedLink
    ) internal returns (bytes32 messageId) {
        if (ccipDestAddress == address(0) || ccipDestChainSelector == uint64(0)) {
            revert MissingCCIPParams();
        }

        // Send CCIP message to the desitnation contract
        IRouterClient router = IRouterClient(CCIP_ROUTER);
        LinkTokenInterface linkToken = LinkTokenInterface(LINK_TOKEN);

        Client.EVM2AnyMessage memory message = Client.EVM2AnyMessage({
            receiver: abi.encode(ccipDestAddress),
            data: data,
            tokenAmounts: new Client.EVMTokenAmount[](0),
            extraArgs: "",
            feeToken: LINK_TOKEN
        });

        uint256 fee = router.getFee(ccipDestChainSelector, message);
        uint256 currentLinkBalance = linkToken.balanceOf(address(this));
-         if (fee > currentLinkBalance) {
+         if (fee > (currentLinkBalance - lockedLink)) {
            //special case token link
            revert InsufficientLinkBalance(currentLinkBalance, fee);
        }

        messageId = router.ccipSend(ccipDestChainSelector, message);
    }
```