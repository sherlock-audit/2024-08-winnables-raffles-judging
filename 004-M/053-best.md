Proper Mulberry Gecko

Medium

# The setRole() function grants role instead of removing

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