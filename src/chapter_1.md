# Overview

## Source: [bao-dao-contracts](https://github.com/baofinance/bao-dao-contracts)

## Bao Distribution Contract

- Bao Finance has voted to move toward a voting escrow model, from a simple governance ERC20 snapshot on mainnet, and at the same time migrate to a new token contract that will facilitate these changes without restriction from the master farmer governance model set previously. This process involves migrating all BAO tokens from the original token contract to the new token contract in order to give everyone with previous BAO balances and new users the ability to use the new system for governance/revenue sharing. This includes both circulating and locked BAO tokens. The proposal detailing the Bao token Migration and Distribution can be found here [Bao Distribution Proposal](https://gov.bao.finance/t/bip-14-token-migration-distribution/1140) and technical specifications of how the Bao Distribution contract interacts with the Voting escrow contract can be found in chapter 2 (Bao Distribution Contract) of this book.

- The Bao Distribution/Migration will be the process by which governance tokens are pre-mined in parellel to liquidity gauge emissions starting.

## Bao Token Contract

- The bao token semantically is exactly the same as the CRV token but with a newer version of vyper, chapter 3 (Bao Token Contract)

## Voting Escrow Contract

- The Voting Escrow contract has added functionality in order for the [distribution contract](https://github.com/baofinance/bao-token/blob/main/src/BaoDistribution.sol) to interact and create locks for existing users that have previously locked BAO in line with the distribution proposal to allow locked Bao holders to lock into veBAO when the distribution and migration process has started. chapter 4 (Voting Escrow Contract)

## Base Burner Contract

- Fee burning happens on a weekly basis to be sent to veBAO holders denominated in baoUSD (BAO Synthetic Dollar) and derives from the APR generated by the utilization of baoUSD in [Bao Markets](https://github.com/baofinance/bao-markets-contracts).

- In the future there is potential for other fee sources to be voted in by the DAO and would therefore create new burn/swap scenarios but for now the only revenue source is baoUSD APR received in baoUSD

- Fee Burning is routed through the [Ballast Contract](https://docs.bao.finance/franchises/bao-markets-hard-synths#ballast) ([Ballast Source here](https://github.com/baofinance/bao-markets-contracts/blob/master/contracts/InverseFinance/Stabilizer.sol)), in the [Base Burner](contracts/burners/BaseBurner.vy) contract and that is where swaps from DAI to baoUSD will occur if the situation arises. After the burning process occurs baoUSD is sent to the [FeeDistributor](contracts/FeeDistributor.vy) in order to be sent to veBAO holders. More technical details on this implementation can be found in chapter 5 (Base Burner Contract).
