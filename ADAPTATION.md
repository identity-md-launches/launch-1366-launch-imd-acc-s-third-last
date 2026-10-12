# Adaptation for Sepolia test run 3

## Code changes

### Revision: one beneficiary handoff per day (reviewer finding 047a2339, medium)

`src/ProjectRegistry.sol`: `setBeneficiary` now refuses a change of wallet within `HANDOFF_INTERVAL`
(1 day) of the previous one, reverting `HandoffTooSoon(openAt)`; the time the next handoff opens is
public as `handoffOpenAt(project)`. Setting the same wallet again is not a handoff and starts no
interval, exactly as it set no pin before. Everything else about handoffs is unchanged: still only
the current beneficiary, only for a listed project, never while a question is open, and the pin
(`challengeForBeneficiary` required after any handoff) is still permanent.

Why: `challengeForBeneficiary` reverts before paying when the beneficiary differs from the one the
challenger named, and `challenge` reverts for good after the first handoff. A beneficiary watching
the mempool could therefore move the wallet ahead of every challenge and back before the next, for
gas only, so no challenge could ever be filed against a project in breach. The reviewer's proof
(three rounds of handoff then challenge on the mock Intake) reproduced it on the previous tree. With
one handoff per day, the challenger's retry against the wallet now in force lands, pays once and files
a question. The front-run check itself stays, since it protects the challenger's fee and bond.

Tests: `test/RegistryRevisionRegression.t.sol` adds
`test_beneficiaryCannotBlockEveryChallengeByTogglingWallets` (the reviewer's scenario: the first
front-run costs the challenger nothing, the move back is refused, the retry files a question naming
the wallet in force, a signed no delists and returns the bond) and
`test_handoffIntervalBoundaryAndRelisting` (refused one second early, allowed at `openAt`, a no-op
does not restart it, a defended challenge in between changes nothing, delisting and relisting keep
it). The two existing tests that moved a wallet and back in the same block now warp past the
interval between the two moves; they tested the pin, not an immediate move back. The accounting and
state-machine invariant handlers model `handoffOpenAt` and expect `HandoffTooSoon` on a second move
inside the interval. Runtime is now 22,294 bytes (was 22,019), under EIP-170; the constructor is
unchanged, so `launch.json` and `test/Deployment.t.sol` still match.

### First round

No other change. The branch already meets the launch rules for an `evm_contracts` launch of one
contract:

- `src/ProjectRegistry.sol`: a nonpayable constructor with seven static arguments (four addresses, three uints); no owner, so neither `msg.sender` nor `tx.origin` gets control; the constructor makes no external call, so it deploys on an empty chain. No DELEGATECALL, CALLCODE, SELFDESTRUCT, proxy, pause or mint. `OracleAttestation` functions are internal, so the bytecode needs no linking.
- The brief says `src/ProjectRegistry.sol` and `src/oracle/OracleAttestation.sol` stay as they are, except where a reproduced finding requires a change; the handoff interval above is the one such change. `launch.json` already lists the agreed arguments and this step does not write it.
- `test/Deployment.t.sol` already checks this manifest: the registry alone, wired to the live Stacker, with bond 5e18 and cooldown 3600.
- `docs/design.md`: the `setBeneficiary` row of the registry table now mentions the one-per-day handoff.
- README: the "Sepolia test run 3" row is added once the registry is live. It has no address yet, so the README is unchanged.

Checks: `forge build` passes, and `forge test` runs 160 tests: 152 pass and 8 fork tests are skipped because no RPC is set. The reviewer's proof passes on the result.

## Reviewer findings (advisory), not changed

- **[low] cdec4c33, self-challenge pre-emption at fee only.** Reproduces as described. Not changed, for the reasons the reviewer also gives: refusing `msg.sender == beneficiary` is defeated by any other wallet of the beneficiary, since a defended bond is paid to the beneficiary and a lost one returned to the asker; and a cooldown after a cleared or defended challenge would let a beneficiary buy immunity with a self-challenge, which is the audit finding `test_defendedSelfChallengeDoesNotRenewImmunity` already guards against, and would also delay honest challengers after a split panel, which the live rehearsals show is common. Every self-challenge still puts the project before a 15-seat panel judging all eight rules. A design change for the author to weigh; on mainnet a challenger can also use a private relay.
- **[low] 0a2f8d12, an answer issued up to five minutes before the ask is accepted.** Reproduces: the lower bound subtracts `ISSUED_AT_TOLERANCE`. Not changed. No unprivileged path reaches it (the answer must be delivered by the Intake under this project's open request, signed for this contract and unconsumed), and the tolerance guards liveness: the attester stamps `issuedAt` from its own clock, the callback is attempted once and never retried, and a refused honest answer costs the fee and discards the verdict. The brief's wording describes the intent, which the check still enforces to within clock drift.
- **[info] 139e729c, a split panel refunds the bond and the challenge can be refiled at once.** Reproduces; it is the documented refund on failure (an honest challenger must not lose a bond to a panel split, which the writer could force anyway by withholding). A post-clear cooldown has the immunity problem above. Documented limit, not changed.

## Audit findings (first round)

- **[low] Signer/Intake/action cannot be rotated.** The finding is accurate, but it follows from the agreed design: the brief requires no owner and leaves the contracts unchanged. If the signer is rotated, the remedy is to redeploy the registry. The Stacker never reads the registry, so it can be replaced. Not changed; this is a trust assumption.
- **[low] A beneficiary can front-run a challenge with a self-challenge.** Same as cdec4c33 above: reproduces, a griefing property of the agreed one-slot design, not changed, recorded so this live run can observe it.
- **[info] An undelivered answer is lost, and `expire` refunds the bond.** This is a liveness property of the Intake, which calls back once and never retries. No fix is needed.
- **[info] `expire` logs "expired" in both cases.** Cosmetic. Not changed.
- **[info] Coverage.** Information only; nothing to act on.
