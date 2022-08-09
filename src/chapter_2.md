# Bao Distirbution Contract

## Important notes

- The Bao Distribution contract is part of the migration process to the new token contract. Those who have old locked BAO tokens will use this contract to receive a proportional balance of the new BAO token scaled down by 1000 streamed over 3 years. The same thing will happen with old circulating bao as we have a contract to make swaps 1000:1 for new BAO tokens which will be able to be locked in veBAO or traded just like any other ERC20.

- Source: [Bao Distribituion](https://github.com/baofinance/bao-token/blob/main/src/BaoDistribution.sol)

## Key Functions that allow for interaction with voting escrow contract

### Start Distrbution

- This function must be called (with a valid merkle proof) in order for those with locked BAO to start streaming the locked tokens owed to their
wallet before being able to interact with other functions in the contract, namely the "lockDistribution()" function which interacts with the voting escrow contract.

```rust
    /**
     * Starts the distribution of BAO for msg.sender.
     *
     * @param _proof Merkle proof to verify msg.sender's inclusion and claimed amount.
     * @param _amount Amount of tokens msg.sender is owed. Used to generate the merkle tree leaf.
     */
    function startDistribution(bytes32[] memory _proof, uint256 _amount) external {
        if (distributions[msg.sender].dateStarted != 0) {
            revert DistributionAlreadyStarted();
        } else if (!verifyProof(_proof, keccak256(abi.encodePacked(msg.sender, _amount)))) {
            revert InvalidProof(msg.sender, _amount, _proof);
        }

        uint64 _now = uint64(block.timestamp);
        distributions[msg.sender] = DistInfo(
            _now,
            0,
            _now,
            _amount
        );
        emit DistributionStarted(msg.sender);
    }
```
### Lock Distribution

- Given the user has called startDistribution() successfully they can then decide to call lockDistribution(uint256 _time) if they do not want to claim their tokens directly to their address (using claim() or endDistribution()). This call will instead allow the distribution contract on behalf of the user/address to create a lock at a specified time (uint256 _time) in the voting escrow contract that must be a minimum of 3 years in line with the length of the distribution streaming curve and a maximum of 4 years in line with the voting escrow contract's max lock time. This happens by calling create_lock_for() a new function added to the voting escrow contract specifically for use by the distibution contract to create locks on behalf of those with old locked BAO that want to transfer their governance power to the new governance system. The distribution contract address must be set within the voting escrow contract in order for those function calls to work. 

```rust
    /**
     * Lock all tokens that have NOT been claimed since msg.sender's last claim
     *
     * The Lock into veBAO will be set at _time with this function call assuming _time is 
     * in between 3 or 4 years to correspond with the length of distribution curve (3 years) from startDistribution() call timestamp
     * and the voting escrow max lock of 4 years.
     */
    function lockDistribution(uint256 _time) external nonReentrant {
        uint256 _claimable = claimable(msg.sender, 0);
        if (_claimable == 0) {
            revert ZeroClaimable();
        }
        if (_time < 94608000) {
            revert outsideLockRange();
        }

        DistInfo storage distInfo = distributions[msg.sender];
        uint64 timestamp = uint64(block.timestamp);

        uint256 daysSinceStart = FixedPointMathLibrary.mulDivDown(uint256(timestamp - distInfo.dateStarted), 1e18, 86400);

        // Calculate total tokens left in distribution after the above claim
        uint256 tokensLeft = distInfo.amountOwedTotal - distCurve(distInfo.amountOwedTotal, daysSinceStart);

        baoToken.approve(address(votingEscrow), tokensLeft);

        //lock tokensLeft for msg.sender for _time years (minimum of 3 years)
        votingEscrow.create_lock_for(msg.sender, tokensLeft, _time);

        emit DistributionLocked(msg.sender, tokensLeft);
    }
```
