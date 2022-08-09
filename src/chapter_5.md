# Base Burner Contract

## Important notes

- The fee burner contract has changes from the underlying_burner.vy implemented by Curve finance in the Base Burner contract but the fee distributor is the same. 

- The burn function now uses the ballast contract (part of Bao Markets) to make swaps from DAI to baoUSD if fees are not already baoUSD. To start all of our fees will be collected and distributed in baoUSD from bao-markets revenue.

- Source: [Ballast Contract](https://github.com/baofinance/bao-markets-contracts/blob/master/contracts/InverseFinance/Stabilizer.sol)

```rust
    @payable
    @external
    def burn(_coin: address) -> bool:
        """
        @notice Receive `_coin` and if not already baoUSD, swap for baoUSD using ballast
        @param _coin Address of the coin being received
        @return bool success
        """
        assert not self.is_killed  # dev: is killed

        # transfer coins from caller
        amount: uint256 = ERC20(_coin).balanceOf(msg.sender)
        if amount != 0:
            response: Bytes[32] = raw_call(
                _coin,
                concat(
                    method_id("transferFrom(address,address,uint256)"),
                    convert(msg.sender, bytes32),
                    convert(self, bytes32),
                    convert(amount, bytes32),
                ),
                max_outsize=32,
            )
            if len(response) != 0:
                assert convert(response, bool)

        # if not baoUSD, swap it for baoUSD using the ballast
        if _coin != baoUSD:
            if not self.is_approved[_coin]:
                response: Bytes[32] = raw_call(
                    _coin,
                    concat(
                        method_id("approve(address,uint256)"),
                        convert(ballast, bytes32),
                        convert(MAX_UINT256, bytes32),
                    ),
                    max_outsize=32,
                )
                if len(response) != 0:
                    assert convert(response, bool)
                self.is_approved[_coin] = True

            # get actual balance in case of transfer fee or pre-existing balance
            amount = ERC20(_coin).balanceOf(self)
            if amount != 0:
                Stabilizer(ballast).buy(amount)

        return True
```

- the execute function now transfers baoUSD to the fee distributor instead of 3CRV

```rust
    @external
    def execute() -> bool:
        """
        @notice transfer baoUSD to the fee distributor
        @return bool success
        """
        assert not self.is_killed  # dev: is killed

        amount: uint256 = ERC20(baoUSD).balanceOf(self)

        if amount != 0:
            ERC20(baoUSD).transfer(self.receiver, amount) #transfer baoUSD to fee distributor

        return True
```