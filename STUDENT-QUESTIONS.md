# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

> Your answer:
One risk is the vault containing MINTER_ROLE. In SimpleStablecoin, onlyRole(MINTER_ROLE) is the same role that permits mint, and it protects burn(address, uint256). Because setUp grants that role to the vault, and because Vault.redeem calls stable.burn(user, amount) directly, the vault can destroy any holder's balance without consent, allowance, or any user-side action.\
\
My test test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance shows this. After Alice deposits 1000e6, her balance is erased to zero by a single vm.prank(address(vault)) followed by burn. There is no on-chain recourse and no event that the victim may refuse. If the vault is ever compromised by a bug, a key leak, or a faulty upgrade, every holder gets drained in a single transaction. The fix is to stop merging mint and burn under a single key. First, separate the roles within SimpleStablecoin, leave mint on MINTER_ROLE, and shift burn(address,uint256) to a new BURNER_ROLE. Second, change Vault.redeem so that it does stable.transferFrom(user, address(this), amount) and then stable.burn(address(this), amount), the user's approval becomes a consent, and the vault can only burn sUSD it was explicitly given. Thirdly, wrap grantRole in a multi signature and a timelock so that the grant path itself cannot be changed in one call. The mint side still needs the vault to hold a role, but burn no longer does.
<br><br><br>

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

> Your answer:
All three roles tied on address(this) inside setUp, which is fine for testing but will be a governance problem in production. I would split them in three independent key sets. \
\
DEFAULT_ADMIN_ROLE belongs to 4-of-7 or 5-of-9 multisignature with a 48-hour timelock because this is the only role that can grant or revoke anything else. A leaked admin key is recoverable only if there is a delay between the grant and its effect. MINTER_ROLE should never be held by a human EOA. It goes to the vault contract and passes to Peg Stability Module, so the compromised part is a contract bug rather than stolen key. PAUSER_ROLE goes to a separate emergency multisignature with a different signer set from the admin multisignature. It is because the people who can freeze the system and the people who can grant roles should not be the same people; otherwise, one compromised signer can both grant more power and freeze. If the contracts are upgradeable, UPGRADER_ROLE gets its own timelock too, that would be much safer. The burn gets a BURNER_ROLE that is only to the vault's own balance, not to an EOA. The principle to grant as least privilege as possible per role, with a delay on any grant that could be used to steal.
<br><br><br>

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

> Your answer:
OpenZeppelin v5's Pausable is a single boolean that check inside _update. _update is the one internal function every balance change flows through, including mint, burn, and transfer. So pause() cannot distinguish ordinary transfer from a burn, which is why it freezes redemption along with everything else. To allow redemption while blocking transfers, it needs to make the check conditional in _update. Override _update in SimpleStablecoin and revert only when from != address(0) && to != address(0). Minting from zero and burning to zero skip the check, which is exactly what redemption needs, because Vault.redeem calls stable.burn(user, amount). Another way is two separate booleans checked independently, so an emergency operator can pause transfers without touching the redemption channel. A third option is to remove the vault address from the pause check, but this hard-codes the vault into the token and is fragile if the vault is ever replaced. The conditional _update is the cleanest. The token stays generic, the pause covers what an emergency actually needs to cover, and the redemption channel still stays open.
<br><br><br>

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:
The two cases are opposite responses to the same shock, and they show that the redemption channel is the peg. In 2008, the Reserve Primary Fund "broke the buck" after Lehman, and redemptions were frozen for multiple days. Every holders who wanted out was forced to queued. The secondary market treated that queue as a loss, so a fund whose assets were in the end recoverable was traded as though it were bankrupt. The freeze was the depeg. In 2023, USDC's reserve bank SVB failed over a weekend, and USDC traded down to $0.87 on exchanges, but Circle left the redemption channel open. The moment when FDIC insurance arrived, arbitrageurs who had bought at $0.87 redeemed at $1.00 and the peg restored within days. Leaving redemption open does two things. It gives holders the option to exit at par, so there is no panic queue, and it also lets arbitrageurs arbitrage the price back to peg. Closing it destroys both, and the only remaining price is the market's guess at the size of the hole. The invariant totalCollateral() >= totalSupply() can still hold, and the coin can still depeg if the redemption channel is shut. 
<br><br><br>

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer:
There are two distinct classes of depeg. The first one is solvency. The invariant totalCollateral() >= totalSupply() is false. This happens if the mint path has weakness, the collateral asset loses value, or the collateral is stolen from the vault. The inequality is directly violated, and the extra sUSD has nothing behind it.\
\
The second is liquidity. The invariant still holds, but holders cannot exchange sUSD for collateral at par. The reason is pause() freezing redemption, an off-chain issuer blocking withdrawals behind a KYC gate, a bank holiday, or secondary-market panic that no one arbitrages. The redemption channel is shut.\
\
These are different problems with different fixes. Solvency needs money, audit, or a working liquidation path. Liquidity needs the redemption channel reopened and a PSM or market maker that will push the price back.
<br><br><br>

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer:
Bribing a MINTER_ROLE holder is the shortest path from zero to a depeg. First, the attacker gains the role by bribing one signer on the multisig, phishing a key, or exploiting a misconfigured upgrade. Next,  stable.mint(attacker, 1_000_000e6) succeeds because the only check on mint is onlyRole(MINTER_ROLE), and the attacker now holds it. Supply rises 1M sUSD with no new collateral input to the vault. And then, attacker does not have to redeem. If redeemed, it would revert once the vault's USDC runs out because of Vault.redeem calls transferFrom(vault, user, amount), and the vault only holds what was deposited. So, attacker sells the sUSD on the open market at a small discount. Buyers see the supply spike so they panic, and the price collapses to whatever the market thinks the remaining collateral is worth. Honest holders eat the loss. Their sUSD still reads "1" on-chain but buys less than a dollar. \
\
The step that could have stopped them is the grant path itself. grantRole behind a 4/7 multisignature plus a 48-hour timelock means the attacker would need multiple signatures and would be visible on-chain for two days before the mint.\
\
The second line of defense is a per-epoch mint upmost enforced in the vault and an off-chain monitor that pauses minting when totalSupply > totalCollateral.\
\
The third line is remove the human MINTER_ROLE entirely, only the vault can mint, and only against collateral, so the role no longer exists to be bought.
<br><br><br>

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer:
No, the invariant cannot be a direct balanceOf. MockUSDC is a bearer asset on the same chain as the vault and collateral.balanceOf(address(this)) is a fact about the chain. Treasuries are not on-chain, title is recorded in a custodian's or broker's books, and the vault's on-chain contract has no legal claim by itself. totalCollateral() becomes a reading from something off-chain. A custody attestation signed by the custodian, an oracle like Chainlink's proof-of-reserve, or a tokenized wrapper that itself is an ERC-20 backed by the custodian. \
\
4 problems appear. First is the timeliness. A custody attestation is stale by construction, so contract needs a maximum-age check that reverts if the reading is too old. Second, settlement latency. Treasuries settle T+1, so instant redemption is impossible unless the vault holds a cash buffer. Third is legal risk. If the custodian fails, the on-chain tokens may not have a claim on the underlying. Fourth is NAV drift. Treasuries are about $1, so the invariant needs a price or NAV input, not just a count. The invariant becomes attestedValue(treasuries) >= totalSupply(), where attestedValue is a trusted oracle, and the trust model of the whole system changes with it.
<br><br><br>

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer:
A building is the hardest version because it is illiquid, indivisible, and slow to value. We cannot put a deed on-chain, we can only wrap it. The minimal structure is an SPV or LLC holds legal title to the property. The SPV issues shares as an ERC-20, the vault holds the shares, totalCollateral() reads shares multiply by NAV_per_share, where NAV comes from an appraiser. Legal counsel forms the SPV and writes the operating agreement. A property manager collects rent and pays expenses. A custodian holds the ownership. An independent appraiser values the property. An auditor reviews the SPV's books. An SPV director signs on its behalf. Redemption is an item. You cannot give a land, so redemption returns SPV shares, and the holder sells them on a secondary market. If no market exists, redemption is effectively closed, which is the liquidity depeg. Yield complicates the design. Rent accrues to the SPV, and you must decide whether to distribute to all share holders or reinvest it, which is a separate ERC-4626-style contract. The invariant becomes NAV multiplies totalShares >= totalSupply, with NAV from a quarterly appraisal and a max-age check. 
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts | test_Ex4_Mint_RevertsForNonMinter |
| Transfers revert while paused | test_Ex4_Pause_BlocksTransfers |
| **Redemption** reverts while paused | test_Ex4_Pause_BlocksRedeem |
| An attacker cannot burn someone else's balance | test_Ex4_AttackerCannotBurnOthersBalance |
| ...but the vault holding `MINTER_ROLE` can | test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance |

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

> Your answer:
Most likely to be attacked is the vault's burn power. It is the only one that destroys value silently, on every holder at once, with no attacker-side balance to trace. A leaked MINTER_ROLE that is used for minting will leave an obvious supply spike that any monitor will catch. A burn leaves totalSupply dropping and users' balances falling to zero, with the attacker needing nothing more than the vault's key. There is also no user-consent guard on the path. burn(address,uint256) is onlyRole(MINTER_ROLE), the vault holds the role, and the user cannot refuse. In the deployed code, the exploit is one line. Compromise the vault, call stable.burn(victim, balance) for every victim, and the peg is gone with no on-chain recourse. This is why test_Ex4_AttackerCannotBurnOthersBalance and test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance are meant to be read together. The guard is written correctly, but the key was handed to the vault. The fix is to split the burn role and make Vault.redeem consume an allowance rather than privilege. 

## Architecture
The jpg photo is "architecture.jpg", path is "/workspaces/stablecoin-lab-2026/architecture.jpg".
![Architecture Diagram](./architecture.jpg)