# Adaptation for Sepolia test run 3

## Code changes

None. The branch already meets the launch rules for an `evm_contracts` launch of one contract:

- `src/ProjectRegistry.sol`: a nonpayable constructor with seven static arguments (four addresses, three uints); no owner, so neither `msg.sender` nor `tx.origin` gets control; the constructor makes no external call, so it deploys on an empty chain. No DELEGATECALL, CALLCODE, SELFDESTRUCT, proxy, pause or mint. `OracleAttestation` functions are internal, so the bytecode needs no linking. Runtime is 22,019 bytes.
- The brief says `src/ProjectRegistry.sol` and `src/oracle/OracleAttestation.sol` stay as they are, and `launch.json` already lists the agreed arguments. This step does not write `launch.json`, so it was left unchanged.
- `test/Deployment.t.sol` already checks this manifest: the registry alone, wired to the live Stacker, with bond 5e18 and cooldown 3600.
- README: the "Sepolia test run 3" row is added once the registry is live. It has no address yet, so the README is unchanged.

Checks: `forge build` passes, and `forge test` runs 158 tests: 150 pass and 8 fork tests are skipped because no RPC is set.

## Audit findings

- **[low] Signer/Intake/action cannot be rotated.** The finding is accurate, but it follows from the agreed design: the brief requires no owner and leaves the contracts unchanged. If the signer is rotated, the remedy is to redeploy the registry. The Stacker never reads the registry, so it can be replaced. Not changed; this is a trust assumption.
- **[low] A beneficiary can front-run a challenge with a self-challenge.** This reproduces as described: the third party's transaction reverts and they lose no funds. The panel still judges all 8 rules on each self-challenge, and a defence does not restart the cooldown. It is a griefing property of the agreed one-slot design. A fix would change the registry's behaviour, which the brief forbids. Not changed; it is recorded so this live run can observe it.
- **[info] An undelivered answer is lost, and `expire` refunds the bond.** This is a liveness property of the Intake, which calls back once and never retries. No fix is needed.
- **[info] `expire` logs "expired" in both cases.** Cosmetic. Changing it would change the contract, which the brief keeps as is. Not changed.
- **[info] Coverage.** Information only; nothing to act on.
