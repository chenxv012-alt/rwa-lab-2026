# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:

In this lab, reserveBalance(), realHoldings(), and totalClaimValue() all use USDC’s six-decimal scale. A simple reported-coverage test is therefore vault.reserveBalance() + custodian.realHoldings() >= vault.totalClaimValue(). The vault calculates the claim from totalSupply × NAV, with the required decimal conversion. This is not the same kind of invariant as counting ERC-20 collateral held by a vault. The first term is an actual on-chain USDC balance, but the second is a number entered by an address with CUSTODIAN_ROLE; recordPurchase() can raise it without receiving any USDC, exactly as Ex2 demonstrates. Moreover, raising NAV increases totalClaimValue() without creating assets. A complete solvency analysis must also distinguish shares escrowed in the redemption queue, their payouts locked at enqueue time, and cash already credited for claims. Thus, passing the simple comparison establishes consistency among reported figures at a particular time. It does not prove that real T-Bills exist, are unencumbered, are valued fairly, or can be sold in time to meet redemptions.

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:

A smart contract can identify token owners and enforce transfers, but it cannot, by itself, establish ownership of securities held in an off-chain account. The legal documents must connect a token to a defined claim against the SPV and specify who holds title, how assets are segregated, and who can act if a service provider fails. Bankruptcy remoteness matters because otherwise the issuer’s creditors might compete with token holders for the same assets. If the custodian fails, the token is not a magic key that withdraws T-Bills from the custodian’s systems. Its holder has only the rights established by the fund documents and applicable law: for example, a claim to a proportionate fund interest and to redemption proceeds when assets are recovered and processed. Recovery could be delayed, disputed, or reduced if records or segregation fail. The design therefore needs enforceable custody agreements, reconciliations, independent records, a replacement-custodian process, and disclosure of residual risk. “Bankruptcy-remote” is a structural objective, not a guarantee of instant or full recovery. 

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:

The reporter is a trust bottleneck because the fund prices subscriptions and redemption requests using the NAV it supplies. Lab 2’s Ex3 shows the effect: one NAV update changes the value of every holder’s claim while their share counts stay the same. First, the reporter’s signing key could be stolen, allowing an attacker to submit a manipulated price. Hardware-backed keys, multisignature approval, and rapid role revocation would reduce that risk. Second, an honest reporter could make a calculation or decimal-scale mistake. Independent calculations, reasonable price bounds, and a second-person review could catch errors before publication. Third, a reporter could deliberately favor an investor or yield to pressure from the issuer. Separating reporting from custody, reconciling figures with independent records, and imposing audit and legal accountability would make manipulation harder. Fourth, the reporter could become unavailable, leaving the fund to transact at an outdated NAV. A genuine update timestamp, a maximum permitted age, and a rule to suspend price-dependent transactions until a fresh report arrives would limit this risk. These controls improve reliability, but the chain still cannot independently prove the value of T-Bills held off-chain.

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:

A stale-but-honest NAV can create an information advantage without any reporter lying. If the last published price is $1.00 but the T-Bills have gained value, an informed investor may subscribe too cheaply and dilute existing holders. If their value has fallen, an informed holder may request redemption at the old, overly generous price and leave the loss with those who remain. This lab has an additional implementation problem: MockPriceFeed.latestRoundData() reports block.timestamp as updatedAt every time it is read. The value can remain unchanged for days while appearing freshly updated. Simply adding block.timestamp - updatedAt to the vault would therefore not detect a frozen feed. A corrected feed must store a lastUpdatedAt value when setPrice() actually changes or confirms the NAV, then return that stored timestamp. The vault can reject reports older than a defined limit. A production rule also needs a publication schedule, a dealing cutoff, and a fair policy for requests made during an outage.

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:

If NAV rises after a redemption request is queued, the redeemer receives the lower, previously locked payout and gives up the increase that occurred before settlement. The holders who stay benefit because the fund pays out less than the departing shares would be worth at the later NAV. For example, ten shares queued at $1.00 receive $10 even if they are worth $11 when settled; that extra dollar effectively remains with the fund. If NAV falls instead, the direction reverses: the redeemer is overpaid relative to the later value, and continuing holders bear the shortfall. Locking the price at enqueue time is not automatically wrong, but it must match the product’s disclosed dealing rule and a realistic way to hedge or fund the commitment. I would normally fix the request time and queue position, but calculate the payout using the next independently published dealing NAV, subject to a known cutoff. The holder should see the final price before irreversible settlement where practical, and exceptional delays should have a stated cancellation or repricing policy.

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:

In 2008, the Reserve Primary Fund valued its shares below $1 after losses linked to Lehman Brothers. Redemption payments were delayed, and regulators permitted a temporary suspension. In 2023, USDC traded below its peg amid concern about reserves at Silicon Valley Bank. Circle maintained its stated commitment to 1:1 redemption, but banking disruptions created delays and a backlog that it worked through as banking services resumed. These were different products with different legal promises, not interchangeable precedents. SEC; Circle.

For a tokenized T-Bill fund, “closing the redemption channel” could mean temporarily rejecting new requests or delaying settlement while securities are sold and cash arrives. It should not mean deleting or silently reordering existing tickets. The queue should preserve each request’s position and locked payout, unless the governing terms expressly provide a fair, disclosed procedure for cancellation or repricing. In this lab, pause() is too blunt for a liquidity squeeze: it blocks tBILL minting, transfers into the queue, and the burns needed for settlement. However, a request settled before the pause can still claim() its credited USDC, because that step does not change a tBILL balance. I would use separate, accountable controls for subscriptions, new redemption requests, settlement, and existing cash claims, with public reasons and review dates.

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:

A regulated fund needs a defensible admission rule, but “not yet verified” should not automatically be treated as “sanctioned.” Before someone receives or trades shares, the fund may need verified identity, investor eligibility, jurisdiction, and sanctions screening. Those criteria should be published in the fund’s terms and applied consistently. A pending KYC applicant should receive a clear route to approval or appeal rather than an unexplained permanent rejection. A genuinely sanctioned address may require immediate blocking under applicable rules, but the reason and authorized process still matter. I would separate statuses such as pending, approved, restricted, and blocked instead of using one Boolean for every situation. Compliance officers should decide individual eligibility using documented evidence; governance should approve the policy; technical administrators should implement changes without being able to invent policy alone. Sensitive changes should be logged, reviewed, and subject to dual control for unusual overrides. The contract enforces a decision, but it cannot decide whether a real person is eligible or whether a screening match is a false positive.

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:

The case for these powers is that a regulated issuer may have to stop sanctioned transfers, correct proven fraud, comply with a court order, or recover assets sent to an ineligible account. Without an emergency control, the fund may be unable to meet legal duties or protect other investors. The case against them is equally serious: the same controls can censor lawful holders, confiscate value, or be abused after a key compromise. The Ex4.3 collision also shows that “freeze” and “seize” need separate semantics. A freeze should stop ordinary transfers; an exceptional burn should require a distinct, narrow route that does not accidentally reopen ordinary transfers. I would give compliance a limited freeze role, place compulsory burns behind a separate multisignature controlled by independent legal and governance representatives, and require a documented legal basis and second approval. Every action should emit an event, identify a case reference, and permit review or appeal where lawful. Roles should have revocation and recovery procedures, and ordinary minting keys should not double as unilateral seizure keys. No code design removes the need for accountable human process.

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | test_Ex2_SharesExistWhileTheCustodianHoldsNothing|
| a | `realHoldings()` moves with no cash moving | test_Ex2_RealHoldingsIsJustANumber|
| b | The claim value floats while the share count stays |test_Ex3_TheClaimFloats_TheShareCountDoesNot |
| b | One `attest` re-prices every holder at once |test_Ex3_OneCallMovesTheWholeBook |
| d | A non-whitelisted address cannot receive shares, even by paying |test_Ex4_NoKyc_NoShares_EvenIfYouPay |
| d | A whitelisted holder cannot send shares to a non-whitelisted address |test_Ex4_TransferChecksBothEndpoints |
| d | Removing an address freezes that holder — and blocks a burn |test_Ex4_FreezeBeatsConfiscation |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
The most likely additional exploit in this lab targets plank b: attestation. Although TBillVault.attest() requires REPORTER_ROLE, MockPriceFeed.setPrice() is an unrestricted external function. A whitelisted attacker without reporter privileges can call it directly to lower the NAV, pay USDC to subscribe at that artificial price, and receive far more tBILL shares than the same payment would buy at the fair NAV. The attacker can then restore the previous price, making the newly minted shares appear to represent a much larger claim. Existing holders are diluted, and the attacker may later try to redeem the inflated position when sufficient cash is available. A focused test would give the attacker whitelist status but no REPORTER_ROLE, call setPrice() directly, and check whether the NAV and sharesForAssets() change. This exposes the bypass in the current mock. After fixing access control, the test should expect an authorization error and confirm that the NAV and share supply remain unchanged.