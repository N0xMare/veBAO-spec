# Voting Escrow Contract

## Important notes

- The Voting Escrow contract from the perspective of normal EOAs with new BAO tokens is the same as Curve's implementation. The process by which the DAO can vote to whitelist contracts (smart wallets) is the same as Curve's implementation as well if we do decide to create a market for bribes in veBAO later. 
- The key difference is that there is added functionality to commit and add a distribution contract as a variable in the voting escrow contract to be used in functions handling logic for the distribution. There is a new internal function, distr_deposit(), to allow the distribution contract to create locks on behalf of the user with locked bao if they decide to interact with the vote escrow contract using lockDistribution() from the previous chapter found in the distribution contract. The new internal function distr_deposit() is only callable through the external function create_lock_for() usable only by the commited/applied address stored in the vote escrow contract as the distr_contract.

- Added functions to note: commit_distr_contract(), apply_distr_contract(), create_lock_for(), distr_deposit()

- functions to commit and apply the distr_contract variable, only admin can call

```rust
    @external
    def commit_distr_contract(addr: address):
        """
        @notice Commit distribution contract
        """
        assert msg.sender == self.admin
        self.future_distr_contract = addr


    @external
    def apply_distr_contract():
        """
        @notice Apply distribution contract
        """
        assert msg.sender == self.admin
        _distr: address = self.future_distr_contract
        assert _distr != ZERO_ADDRESS   # distr contract not set
        self.distr_contract = self.future_distr_contract
```
- External function called by distribution contract using lockDistribution() to create locks on behalf of addresses with a BAO distribution

```rust
    @external
    @nonreentrant('lock')
    def create_lock_for(_to: address, _value: uint256, _unlock_time: uint256):
        """
        @notice Deposit `_value` tokens for `_to` address and lock until `_unlock_time`
        @param _value Amount to deposit
        @param _unlock_time Epoch time when tokens unlock, rounded down to whole weeks
        """
        self.assert_distr_contract(msg.sender)
        unlock_time: uint256 = (_unlock_time / WEEK) * WEEK  # Locktime is rounded down to weeks
        _locked: LockedBalance = self.locked[_to]

        assert _value > 0  # dev: need non-zero value
        assert _locked.amount == 0, "Withdraw old tokens first"
        assert unlock_time > block.timestamp, "Can only lock until time in the future"
        assert unlock_time <= block.timestamp + MAXTIME, "Voting lock can be 4 years max"

        self.distr_deposit(_to, _value, unlock_time, _locked, CREATE_LOCK_FOR_TYPE)
```
- Internal function used by the external function create_lock_for() to assign locks from msg.sender to another address, only callable through create_lock_for() so it requires that only distr_contract(address) can use this part of the contract. The transferFrom() call is modified with the distr.contract address instead of "_addr"/user's address found in _deposit_for() in order to allow for lock creation by another address set by the admin address as _addr in this scenario wont have any bao tokens (the distribution contract has them on behalf of the user).

```rust
    @internal
    def distr_deposit(_addr: address, _value: uint256, unlock_time: uint256, locked_balance: LockedBalance, type: int128):
        """
        @notice Deposit and lock tokens for a user
        @param _addr User's wallet address
        @param _value Amount to deposit
        @param unlock_time New time when to unlock the tokens, or 0 if unchanged
        @param locked_balance Previous locked amount / timestamp
        """
        _locked: LockedBalance = locked_balance
        supply_before: uint256 = self.supply

        self.supply = supply_before + _value
        old_locked: LockedBalance = _locked
        # Adding to existing lock, or if a lock is expired - creating a new one
        _locked.amount += convert(_value, int128)
        if unlock_time != 0:
            _locked.end = unlock_time
        self.locked[_addr] = _locked

        # Possibilities:
        # Both old_locked.end could be current or expired (>/< block.timestamp)
        # value == 0 (extend lock) or value > 0 (add to lock or extend lock)
        # _locked.end > block.timestamp (always)
        self._checkpoint(_addr, old_locked, _locked)

        if _value != 0:
            assert ERC20(self.token).transferFrom(self.distr_contract, self, _value)

        log Deposit(_addr, _value, _locked.end, type, block.timestamp)
        log Supply(supply_before, supply_before + _value)
```