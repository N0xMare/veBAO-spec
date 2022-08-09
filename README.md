# veBAO Technical specifications

- This document highlights the main design differences in the vote escrow system of the BAO token from the protocol I forked, Curve Finance.

- The functionality involving the gauge controller, voting escrow, fee distributor and other accompanying contracts is operationaly the same as Curve Finance's system (or at least it should be lol)

- The significant changes that were made are found in the voting escrow contract and burning process to allow for added functionality invovling creating locks on behalf of other EOAs for the distribution/token migration from the old system of snapshot governance to the new vote escrow system. The burning process is now directed from our synthetic market revenues to veBAO holders denominated in baoUSD (synth dollar) instead of 3CRV from curve AMM fees.
