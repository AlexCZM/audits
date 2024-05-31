# H-01 Users can abuse incorrect userGaugeProfitIndex accounting to drain up the gauge rewards

> # Lines of code
> https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L416-L418
> 
> # Vulnerability details
> ## Impact
> The `ProfitManager.claimGaugeRewards` doesn't update `userGaugeProfitIndex` when `deltaIndex == 0`. Because of this users who stake/unstake at least once and then stake again are rewarded for the period they weren't staked.
> 
> Moreover if they can flashloan a big amount of Guild token, they can drain ProfitManager Credit balance since the rewards are distributed at a pro-rata share.
> 
> ## Proof of Concept
> Let's take the following example:
> 
> 1. user Alice stake Guild to a gauge (`incrementGauge`). `claimGaugeRewards` is
>    [called](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/GuildToken.sol#L258).
>    This call [returns 0](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L416)
>    since `_userGaugeWeight == 0`; userGaugeWeight is incremented later by
>    [`super._incrementGaugeWeight`](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/GuildToken.sol#L260C9-L260C36)
>    [here](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/ERC20Gauges.sol#L240C15-L240C15).
> 
> `userGaugeProfitIndex` is not updated;
> 
> 2. protocol accumulate interest and `gaugeProfitIndex` is [updated](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L396-L399) (these are the rewards alocated for guildSplit).
> 3. Alice unstake  her entire weight from the gauge: `decrementGauge` -> [calls](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/GuildToken.sol#L220) `claimGaugeRewards`:
>    
>    * this time `_userGaugeWeight != 0`;
>    * `deltaIndex != 0` => Alice gets her share of the accumulated interest.
>    * `userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;` -> OK
>    * `super._decrementGaugeWeight` is executed next and `getUserGaugeWeight` is cleared
> 4. time passes, lendingTerm (gauge) accumulate more interests, `gaugeProfitIndex` is updated accordingly;
> 5. Alice stakes again by calling `incrementGauge`. The things from step (1) happens again:`claimGaugeRewards` returns 0 because `_userGaugeWeight == 0` => `userGaugeProfitIndex` is not set to `_gaugeProfitIndex` -> NOK
> 6. Alice can unstake immediately and because `userGaugeProfitIndex` wasn't updated when when she staked 2nd time, she is rewarded for the period she had 0 weight allocated on the gauge -> NOK
> 
> Protocol can get drained following next steps:
> 
> * Alice allocates a minimum weight (1 wei) to all available gauges;
> * Alice does/waits steps 1-4 above;
> * Alice stakes a big amount (eg. from a flash loan) in step (5).
>   Credit earned per gauge is [directly proportional](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L429) to the amount staked in step 5:
>   `creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;`
> 
> Coded PoC: Add following test in "../unit/governance/ProfitManager.t.sol" file and execute it with `forge test --mt testStealGaugeRewards -vvv`
> 
> ```solidity
>     function testStealGaugeRewards() public {// H4
>         // grant roles to test contract
>         vm.startPrank(governor);
>         core.grantRole(CoreRoles.GOVERNOR, address(this));
>         core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
>         core.grantRole(CoreRoles.GUILD_MINTER, address(this));
>         core.grantRole(CoreRoles.GAUGE_ADD, address(this));
>         core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
>         core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
>         vm.stopPrank();
> 
>         uint256 guildAmount = 100e18;// `stake` amount (incrementGauge, decrementGauge weight amount)
> 
>         vm.prank(governor);
>         profitManager.setProfitSharingConfig(
>             0.5e18, // surplusBufferSplit
>             0, // creditSplit
>             0.5e18, // guildSplit
>             0, // otherSplit
>             address(0) // otherRecipient
>         );
>         guild.setMaxGauges(1);
>         guild.addGauge(1, gauge1);
>         guild.mint(alice, guildAmount);
>         guild.mint(bob, guildAmount);
> 
>         vm.prank(alice);
>         guild.incrementGauge(gauge1, guildAmount);
>         vm.prank(bob);
>         guild.incrementGauge(gauge1, guildAmount);
> 
>         // simulate 40 profit on gauge1:
>         // - 20 should go to surplusBuffer;
>         // - 20 should be split equally between alice and bob
>         credit.mint(address(profitManager), 40e18);
>         profitManager.notifyPnL(gauge1, 40e18);
> 
>         assertEq(profitManager.claimRewards(alice), 10e18, "alice claimRewards1 == 10e18");
>         assertEq(profitManager.claimRewards(bob), 10e18, "bob claimRewards1 == 10e18");
>         
>         assertEq(profitManager.userGaugeProfitIndex(alice,gauge1), 1.10e18, "alice userGaugeProfitIndex != 1.10 e18");
>         assertEq(profitManager.userGaugeProfitIndex(bob,gauge1), 1.10e18, "bob userGaugeProfitIndex != 1.10 e18");
> 
>         // bob unstake
>         vm.prank(bob);
>         guild.decrementGauge(gauge1, guildAmount);
>         assertEq(guild.getUserGaugeWeight(bob, gauge1), 0, "bob is still staking in gauge1");
> 
>         // simulate a 16 credit profit split:
>         // - 8 goes to surplus buffer;
>         // - 8 goes to the guildSplit => 100% to alice since she's the only staker
>         credit.mint(address(profitManager), 16e18);
>         profitManager.notifyPnL(gauge1, 16e18);
>         assertEq(profitManager.claimRewards(alice), 8e18,  "alice second round of rewards");
> 
>         // bob stakes again 
>         vm.prank(bob);
>         guild.incrementGauge(gauge1, guildAmount);
> 
>         assertEq(profitManager.userGaugeProfitIndex(bob, gauge1),
>             profitManager.gaugeProfitIndex(gauge1), 
>             "bob userGaugeProfitIndex was not updated");
>         assertEq(profitManager.claimRewards(bob), 0, "bob should get no rewards");
>         //bob should have 10 Credit from first claimRewards 
>         assertEq(credit.balanceOf(bob), 10e18, "bobs credit balance != 10");
>     }
> ```
> 
> ## Tools Used
> Manual review
> 
> ## Recommended Mitigation Steps
> Removing [this](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L416-L418) check from `claimGaugeRewards` seems to solve the problem. `userGaugeProfitIndex` is updated when needed (delta != 0) and function will still return 0 `creditEarned` (as it should when userGaugeWeight == 0).
> 
> ```js
>         if (_userGaugeWeight == 0) {
>             return 0;
>         }
> ```
> 
> However further analysis for potential side effects is required.
> 
> ## Assessed type
> Other


***********************************************************************************

# H-02 CreditMultiplier is not applied to `creditAsked` when a bid for an active auction is placed. This can reduce the creditMultiplier and thus the value of creditToken holders

> # Lines of code
> https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L228-L230 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L751-L755 https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L758-L767
> 
> # Vulnerability details
> ## Impact
> * auction can't be won as long as auction is in first phase;
> * interest is not paid;
> * protocol (and credit holders) are forcing to incur a loss when auction enters second phase (100% collateralToBidder, less and less Credit payed back).
> * when auction enters second phase, bidder pays less Credit than it should.
> 
> ## Proof of Concept
> In case of a late partial repayment or offboarding lendingTerm loans can be `call`ed.
> 
> `LendingTerm.getLoanDebt` is [used](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L665) to calculate the outstanding borrowed amount of a loan. It includes the principal, interest, openFee. A `creditMultiplier` correction is [applied](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L228-L230) to reflect the updated loan value in case creditToken <-> underlyingToken ratio decreased after the loan was opened.
> 
> ```js
> //getLoanDebt
>         uint256 creditMultiplier = ProfitManager(refs.profitManager).
>             creditMultiplier();
>         loanDebt = (loanDebt * loan.borrowCreditMultiplier) / 
>             creditMultiplier;
> ```
> 
> `loans` mapping is [updated](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L665-L667) with up-to-date `loadDebt` amount.
> 
> ```js
> //_call
>         // update loan in state
>         uint256 loanDebt = getLoanDebt(loanId);
>         loans[loanId].callTime = block.timestamp;
>         loans[loanId].callDebt = loanDebt;
> ```
> 
> The auction is started. `auctions[loanId]` [store](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/AuctionHouse.sol#L101) same up-to-data `loanDebt` (named callDebt now).
> 
> ```js
>         // save auction in state
>         auctions[loanId] = Auction({
>             startTime: block.timestamp,
>             endTime: 0,
>             lendingTerm: msg.sender,
>             collateralAmount: loan.collateralAmount,
>             callDebt: callDebt
>         });
> ```
> 
> Now auction is active and anyone can `bid` on it. `AuctionHouse.getBidDetail` is used to get the `collateralReceived` (by bidder) for `creditAsked`. As we can [see](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/AuctionHouse.sol#L134-L153) updated `callDebt` (updated at the moment the auction started) is returned for `creditAsked`. (we ignore the fact less Credit is asked in second phase of the auction).
> 
> ```js
>         if (block.timestamp < _startTime + midPoint) {
>             // ask for the full debt
> >>          creditAsked = auctions[loanId].callDebt;
> 
>             // compute amount of collateral received
>             uint256 elapsed = block.timestamp - _startTime; // [0, midPoint[
>             uint256 _collateralAmount = auctions[loanId].collateralAmount; // SLOAD
>             collateralReceived = (_collateralAmount * elapsed) / midPoint;
>         }
>         // second phase of the auction, where less and less CREDIT is asked
>         else if (block.timestamp < _startTime + auctionDuration) {
>             // receive the full collateral
>             collateralReceived = auctions[loanId].collateralAmount;
> 
>             // compute amount of CREDIT to ask
>             uint256 PHASE_2_DURATION = auctionDuration - midPoint;
>             uint256 elapsed = block.timestamp - _startTime - midPoint; // [0, PHASE_2_DURATION[
>             uint256 _callDebt = auctions[loanId].callDebt; // SLOAD
> >>          creditAsked = _callDebt - (_callDebt * elapsed) / PHASE_2_DURATION;
>         }
> ```
> 
> `creditAsked` is [passed down](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/AuctionHouse.sol#L185) to `onBid` callback.
> 
> * The `principal` is calculated by applying the `borrowCreditMultiplier / creditMultiplier` [correction](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L751-L755):
> 
> ```js
>         uint256 creditMultiplier = ProfitManager(refs.profitManager)
>             .creditMultiplier();
>         uint256 borrowAmount = loans[loanId].borrowAmount;
>         uint256 principal = (borrowAmount *
>             loans[loanId].borrowCreditMultiplier) / creditMultiplier;
> ```
> 
> * `creditFromBidder` is the `creditAsked` == `callDebt`  == `loanDebt` updated at the moment auction started. No new correction is applied.
> 
> Let me explain what I mean by that and why it matters.
> 
> The auction can last many blocks, up to [30 minutes](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/test/proposals/gips/GIP_0.sol#L175-L179) based on in the scope deployment script.
> 
> ```js
>     //GIP_0.sol
>             AuctionHouse auctionHouse = new AuctionHouse(
>                 AddressLib.get("CORE"),
>                 650, // midPoint = 10m50s
>                 1800 // auctionDuration = 30m
>             );
> ```
> 
> `creditMultiplier` can have a correction down (due to bad debt accumulation) between (1) the loan was called (auction started) and (2) the auction was bid on.
> 
> On the time axis we have 3 creditMultiplier values of interest :
> 
> `borrowCreditMultiplier` >= `auctionStartedCreditMultiplier` >= `bidOnAuctionCreditMultiplier` == `creditMultiplier`
> 
> Going back on `onBid` callback :
> 
> -> `principal` = `borrowAmount * borrowCreditMultiplier/ bidOnAuctionCreditMultiplier`
> 
> -> `creditFromBidder` = `borrowAmountPlusInterests * borrowCreditMultiplier / auctionStartedCreditMultiplier`
> 
> In the first phase of auction more and more collateral is offered for the same amount of CREDIT to pay. Let's suppose a bidder bids in this phase, offering `creditFromBidder` for a x% ( x < 100%) of collateral. But, in case `creditMultiplier` decreased between `startAuction` and `bid` moment, `principal` can be bigger than `creditFromBidder` amount, forcing the code to enter [else](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L761-L767) branch:
> 
> ```js
>         } else {
>             pnl = int256(creditFromBidder) - int256(principal);
>             principal = creditFromBidder;
>             require(
>                 collateralToBorrower == 0,
>                 "LendingTerm: invalid collateral movement"
>             );
> ```
> 
> Because `collateralToBorrower` must be 0 even if auction is in first half (and collateral is split between bidder and borrower), the bid transaction reverts:
> 
> * auction can't be won as long as auction is in first phase;
> * interest is not paid;
> * protocol (and credit holders) are forcing to incur a loss when auction enters second phase (100% collateralToBidder, less and less Credit payed back).
> * when auction enters second phase, bidder pays less Credit than it should.
> 
> ## Tools Used
> Manual review
> 
> ## Recommended Mitigation Steps`
> Consider applying same correction to both `principal` and `creditFromBidder`:
> 
> * LendingTerm._call : loans[loanId].callDebt => save rawLoanDebt (principal +interest + openFee) (but do not apply creditMultiplier correction)
> * AuctionHouse.getBidDetail : getBidDetail => apply creditM correction to rawLoanDebt to calculate creditAsked:
>   `creditAsked` = `rawLoanDebt * borrowCreditMultiplier/ bidOnAuctionCreditMultiplier`
> * when `onBid` callback is called the `creditAsked` updated amount is passed and compared with `principal` which has same correction applied.
> 
> ## Assessed type
> Error

***********************************************************************************
# H-03 After a first slashing event, all subsequent SGM `stake` are slashed because of using a non-initialized `userStake` variable

> # Lines of code
> https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L223-L231
> 
> # Vulnerability details
> ## Impact
> After a first slashing event all subsequent SGM staking are considered slashed. Stakers loose principal staked and any potential Guild token rewards.
> 
> ## Proof of Concept
> Inside `SurplusGuildMinter.getRewards() ` , `lastGaugeLoss ` timestamp is [compared](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L229) with the default, uninitialized `userStake.lastGaugeLoss`.
> 
> ```solidity
>   function getRewards(
>         address user,
>         address term
>     )
>         public
>         returns (
>             uint256 lastGaugeLoss, // GuildToken.lastGaugeLoss(term)
>             UserStake memory userStake, // stake state after execution of getRewards()
>             bool slashed // true if the user has been slashed
>         )
>     {
>         bool updateState;
>         lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
>         if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) {
>             slashed = true;
>         }
>         ...
> ```
> 
> Variable `slashed` is not updated afterwards and users guild reward is [deleted](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L253-L255) and their staked balance [cleared](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L274-L283) as well.
> 
> Coded PoC: Add folowing test in SurplusGuildMinter.t.sol file
> 
> ```solidity
>     function testSubsequentStakersAreSlashed_AfterBadDebtLossEvent() public {
> 
>         address alice = makeAddr("alice");
>         address bob = makeAddr("bob");
>         credit.mint(alice, 100e18);
>         credit.mint(bob, 100e18);
> 
>         vm.startPrank(alice);
>         credit.approve(address(sgm), 100e18);
>         sgm.stake(term, 100e18);
>         
>         assertEq(credit.balanceOf(alice), 0, "alice should have no credit");
>         assertEq(profitManager.surplusBuffer(), 0, "should have no surplus");
>         assertEq(profitManager.termSurplusBuffer(term), 100e18, "should have term surplus");
>         assertEq(guild.balanceOf(address(sgm)), 200e18, "sgm guild balance");// because of mintRation = 2:1
>         assertEq(guild.getGaugeWeight(term), 250e18, "term gauge weight");// 200 + 50 from setUp
>         assertEq(sgm.getUserStake(alice, term).credit, 100e18,"alice stake amount");
>         vm.stopPrank();
> 
>         vm.prank(governor);
>         profitManager.setProfitSharingConfig(
>             0.25e18, // surplusBufferSplit
>             0.25e18, // creditSplit
>             0.5e18, // guildSplit
>             0, // otherSplit
>             address(0) // otherRecipient
>         );
> 
>         // add bad debt
>         profitManager.notifyPnL(term, -30e18);
>         assertEq(profitManager.surplusBuffer(), 100e18 - 30e18); // 70e18
>         assertEq(profitManager.termSurplusBuffer(term), 0);
> 
>         // cannot stake if there was just a loss
>         // next block
>         vm.warp(block.timestamp + 13);
>         vm.roll(block.number + 1);
> 
>         // apply loss to alice
>         sgm.getRewards(alice, term);
>         // alice got no rewards because term accrued no interest;
>         // alice got slashed
>         assertEq(credit.balanceOf(alice), 0, "alice credit rewards after slashing");
>         assertEq(sgm.getUserStake(alice, term).credit, 0, "alice staked credit after slashing");
>         assertEq(guild.balanceOf(alice), 0, "alice guild rewards after slashing");
> 
>         // bob stakes
>         vm.startPrank(bob);
>         credit.approve(address(sgm), 100e18);
>         sgm.stake(term, 100e18);
> 
>         assertEq(credit.balanceOf(bob), 0, "bob should have no credit");
>         assertEq(guild.balanceOf(bob), 0, "bob should have no guild");
>         assertEq(profitManager.surplusBuffer(), 70e18, "surplusBuffer amount");// from previous slashing event
>         assertEq(profitManager.termSurplusBuffer(term), 100e18, "termSurplusBuffer amount");// bob staked amount
>         assertEq(guild.balanceOf(address(sgm)), 200e18 + 200e18, "sgm guild balance");
>         assertEq(guild.getGaugeWeight(term), 250e18 + 200e18, "term gauge weight");// 200 + 50 from setUp + 200 from alice
>         assertEq(sgm.getUserStake(bob, term).credit, 100e18,"bob stake amount");
>         vm.stopPrank();
> 
>         // accumulate profit
>         profitManager.notifyPnL(term, 45e18);
>         // 
>         (, , bool isBobSlashed) = sgm.getRewards(bob, term);
>         assertEq(isBobSlashed, false, "bob should not be slashed");
>         assertEq(credit.balanceOf(bob), 10e18, "bob credit rewards after PnL");
>         assertEq(sgm.getUserStake(bob, term).credit, 100e18, "bob staked credit after PnL");
>         assertGt(guild.balanceOf(bob), 0, "bob guild rewards after PnL");
>         
>         vm.prank(bob);
>         sgm.unstake(term, 100e18);
>         assertEq(credit.balanceOf(bob), 110e18, "bob credit balance after unstake");// initial staking amount + rewards
>         assertEq(guild.balanceOf(bob), 0, "bob guild balance after unstake");
>         
>     }
> ```
> 
> ## Tools Used
> Manual review
> 
> ## Recommended Mitigation Steps
> Cache user `_stakes` first and use it afterwards:
> 
> ```solidity
>         bool updateState;
>         lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
> +        userStake = _stakes[user][term];
>         if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) {
>             slashed = true;
>         }
> 
>         // if the user is not staking, do nothing
> -        userStake = _stakes[user][term];
> ```
> 
> 
> ## Assessed type
> 
> Error
> 

