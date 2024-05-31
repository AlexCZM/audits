# H-01 Players can reRoll more times than they are allowed to because of user selectable fighterType parameter

> # Lines of code
> https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L370-L372
> 
> # Vulnerability details
> ## Impact
> Players can reroll more times than they are allowed to. By rerolling fighters can get different weight, element and physicalAttributes. First 2 impact the playing style, while the last contribute to nft rarity.
> 
> ## Proof of Concept
> The game has a reRoll mechanism which allow fighters to get new random traits. The number of reroll allowed per NFT is set by `maxRerollsAllowed[fighterType]`. Let's suppose the game owner wants to increase the champion (fighterType 0) generation:
> 
> * he first [calls](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L131-L139) `addAttributeProbabilities` to add the probabilities for the new generation (`attributeProbabilities` is [used](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L174) in `dnaToIndex` to get attribute probability index);
> * then he [calls](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L129-L133) `incrementGeneration`.
> 
> A player wants to reroll a dendroid NFT (fighter type 1) but calls `reRoll` with fighterType == 0 (champion). By selecting the other fighter type (than his tokenId type) he benefits the extra reroll champion benefit. The dendroid nft gets new pseudo-random traits and physical attributes, even if dendroids have all physycal attributes [hardcoded](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L93-L95) to 99.
> 
> ## Tools Used
> ## Recommended Mitigation Steps
> Consider removing `fighterType` function attribute from `reroll` and use `fighters[tokenId].dendroidBool` as index for fighter type :
> 
> ```solidity
> -    function reRoll(uint8 tokenId, uint8 fighterType) public {
> +    function reRoll(uint8 tokenId) public {
> +        uint8 fighterType = uint8(fighters[tokenId].dendroidBool);
>         require(msg.sender == ownerOf(tokenId));
>         require(numRerolls[tokenId] < maxRerollsAllowed[fighterType]);
> ...  
> ```
> 
> ## Assessed type
> Invalid Validation

***********************************************************************************

# M-01 Insufficient `customAttributes` validation allow players to set any values for weight and element when `mintFromMergingPool()`

> # Lines of code
> https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/MergingPool.sol#L139-L158 https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L329 https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L499-L506
> 
> # Vulnerability details
> ## Impact
> Players can get a dishonest advantage by allowing them to set any values for `weight` and `element`. Game mechanics are broken by having heroes with unusual element/weight values.
> 
> ## Proof of Concept
> One way users are rewarded is by letting them to mint new heroes. After admin `pickWinner`s, the winners can call `MerginPool.claimRewards()`. The issue rely on the fact that winners can set any values for `customAttributes`, which are [passed](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/MergingPool.sol#L154) to `mintFromMergingPool` and further used [to set](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L499-L506) `element` and `weight` attributes.
> 
> ```solidity
> function _createNewFighter(
> ...
>         if (customAttributes[0] == 100) {
>             (element, weight, newDna) = _createFighterBase(dna, fighterType);
>         }
>         else {
>             element = customAttributes[0];
>             weight = customAttributes[1];
>             newDna = dna;
>         }
> ...
>         fighters.push(
>             FighterOps.Fighter(
>                 weight,
>                 element,
>                 attrs,
> ```
> 
> ## Tools Used
> ## Recommended Mitigation Steps
> I see two different solutions.
> 
> * sanitize the player's inputs by adding proper `require` checks;
> * hardcore `customAttribute` to [uint256(100), uint256(100)] when calling `_createNewFighter` in `mintFromMergingPool` as in the other 2 places.
> 
> ```solidity
>    function mintFromMergingPool(
>        address to, 
>        string calldata modelHash, 
>        string calldata modelType, 
>        uint256[2] calldata customAttributes
>    ) 
>        public 
>    {
>        require(msg.sender == _mergingPoolAddress);
>        _createNewFighter(
>            to, 
>            uint256(keccak256(abi.encode(msg.sender, fighters.length))), 
>            modelHash, 
>            modelType,
>            0,
>            0,
> -            customAttributes
> +            [uint256(100), uint256(100)]
>        );
>    }
> ```
> 
> By doing so, element, weight and dna is computed by `_createFighterBase`.
> 
> ## Assessed type
> Invalid Validation

***********************************************************************************

# M-02 Users may be unable to `claimNRN()` rewards due to looping over unbounded roundId

> # Lines of code
> https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L292-L311
> 
> # Vulnerability details
> ## Impact
> If roundId grows big enough, users may be unable to claim NRN rewards due to block gas limit.
> 
> ## Proof of Concept
> `claimNRN()` function includes a [loop](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L298-L305) that iterates over each round from the first round the user has not claimed (initially 0) up to the current roundId. The issue here lies in the fact that the number of iterations is not user controlable.
> 
> If `roundId` grows big enough and if a user has not claimed their rewards for a large number of rounds, iterating over all rounds will became very costly and can result in a gas cost that is over the block gas limit. In this case user can't claim his rewards anymore.
> 
> ```solidity
>     /// @notice Claims NRN tokens for the specified rounds.
>     /// @dev Caller can only claim once per round.
>     function claimNRN() external {
>         require(numRoundsClaimed[msg.sender] < roundId, "Already claimed NRNs for this period");
>         uint256 claimableNRN = 0;
>         uint256 nrnDistribution;
>         uint32 lowerBound = numRoundsClaimed[msg.sender];
>         for (uint32 currentRound = lowerBound; currentRound < roundId; currentRound++) {
>             nrnDistribution = getNrnDistribution(currentRound);
>             claimableNRN += (
>                 accumulatedPointsPerAddress[msg.sender][currentRound] * nrnDistribution   
>             ) / totalAccumulatedPoints[currentRound];
>             numRoundsClaimed[msg.sender] += 1;
>         }
>         if (claimableNRN > 0) {
>             amountClaimed[msg.sender] += claimableNRN;
>             _neuronInstance.mint(msg.sender, claimableNRN);
>             emit Claimed(msg.sender, claimableNRN);
>         }
>     }
> ```
> 
> ## Tools Used
> ## Recommended Mitigation Steps
> Consider limiting the number of roundIds a user can claim rewards for at a time.
> 
> ```solidity
>         uint32 lowerBound = numRoundsClaimed[msg.sender];
> -        for (uint32 currentRound = lowerBound; currentRound < roundId; currentRound++) {
> +        uint256 maxRoundId = lowerBound + 25 < roundId ? lowerBound + 25 : roundId;
> +        for (uint32 currentRound = lowerBound; currentRound < maxRoundId; currentRound++) {
> ...
> }
> ```
> 
> ## Assessed type
> DoS

***********************************************************************************

# M-03 Weak source of randomness

> # Lines of code
> https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L324 https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L254 https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L214
> 
> # Vulnerability details
> ## Impact
> Using on-chain, known attributes as a source of pseudo-randomness opens the door for gamifying the system; players can get a dishonest advantage. Players can see in advance the reroll heroes traits and can decide not to `reRoll`. Protocol doesn't accumulate rerollCost.
> 
> ## Proof of Concept
> `_createNewFighter` and `_createFighterBase` rely on week source of randomness from only on-chain attributes. Users can manipulate the DNA of their hero in different ways depending where `_createNewFighter` is called from:
> 
> * [`claimFighters`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L214) : players can wait other users mint fighters so `fighters.length` get increased. Once a favorable DNA results from hashing the `(abi.encode(msg.sender, fighters.length))`, player can call `claimFighters`.
> 
> ```solidity
> ... 
>        for (uint16 i = 0; i < totalToMint; i++) {
>             _createNewFighter(
>                 msg.sender, 
>                 uint256(keccak256(abi.encode(msg.sender, fighters.length))),
>                 modelHashes[i], 
> ```
> 
> * in `redeemMintPass` case it seems player can [set any](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L254) `mintPassDnas` they desire.
> * in [`mintFromMergingPool`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L324) case it's similar to first case, but `msg.sender` is MerginPool address instead.
> * when `reRoll` is called, `msg.sender, tokenId, numRerolls` are [used](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L379-L380). Players know in advance the reroll results and can decide not to call the function and pay for it. Protocol doesn't accumulate the reroll fees.
> 
> DNA is used to create the base attributes for the fighter:
> 
> * Element determines special abilities and
> * Weight determines the battle attributes
> * DNA itself is used to create physical attributes, which have rarities.
>   Both element and weight must be in a [specific interval](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L470-L471)
> 
> ```solidity
>         numElements[0] = 3;
> ...
>         uint256 element = dna % numElements[generation[fighterType]];// @audit 3 possible values for generation 0;
>         uint256 weight = dna % 31 + 65;// @audit [65, 95]
> ```
> 
> ## Tools Used
> ## Recommended Mitigation Steps
> Use a safe way of getting random numbers (such as ChainLink VRF).
> 
> ## Assessed type
> Other

***********************************************************************************

# M-04 Players can circumvent `fighterStaked` transfer restriction and play the game risk free

> # Lines of code
> https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L543 https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L254
> 
> # Vulnerability details
> ## Impact
> Users can transfer fighters even if they are staked. Further they can participate in battles without the risk of loosing their NRN amount staked or loosing points.
> 
> ## Proof of Concept
> First time when a player stake a NRN balance to one of his heroes, that hero is considered [staked](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L254).
> 
> When player unstake entire balance from a hero token, that here is [not staked](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L286) anymore.
> 
> `FighterFarm` [implements](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L533-L545) `_ableToTransfer()` that require a token to be unstaked before transferring to another address.
> 
> ```solidity
>     /// @notice Check if the transfer of a specific token is allowed.
>     /// @dev Cannot receive another fighter if the user already has the maximum amount.
>     /// @dev Additionally, users cannot trade fighters that are currently staked.
>     /// @param tokenId The token ID of the fighter being transferred.
>     /// @param to The address of the receiver.
>     /// @return Bool whether the transfer is allowed or not.
>     function _ableToTransfer(uint256 tokenId, address to) private view returns(bool) {
>         return (
>           _isApprovedOrOwner(msg.sender, tokenId) &&
>           balanceOf(to) < MAX_FIGHTERS_ALLOWED &&
>           !fighterStaked[tokenId]
>         );
>     }
> ```
> 
> Players can circumvent this restriction in a few steps:
> 
> * call `stakeNrn()` and stake 1_000 NRN; `amountStaked[tokneId]` is increased, FighterFarm `fighterStaked` is true;
> * loose one battle. `stakeAtRisk[roundId][tokenId]` is increased with 0.1% * amountStaked = 1 (for `bpsLostPerLoss` == 10);
> * unstake all NRN tokens left; `hasUnstaked[tokenId][roundId]` is set to true;
>   FighterFarm `fighterStaked` is set to false;
> * player wins a battle. The `bpsLostPerLoss` part from his `stakeAtRisk` is [reclaimed](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L461).
>   `amountStaked[tokenId]` is set to 0.001 * 1e18 (0.1% of 1 NRN).
> 
> `fighterStaked` is false (can be transferred) and starting with the next round the new owner can call `stakeNRN` [without locking token transfer](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L253-L255) (`updateFighterStaking` is not called).
> 
> This is the first part of the issue. Next you can read how a malicious player can benefit from this.
> 
> Before transferring the token, player stake a big amount of NRN. Next he waits to win a battle so `accumulatedPointsPerFighter` get increased. The player transfer the fighter to another address he controls. the following will happen/ are true:
> 
> * `accumulatedPointsPerFighter[tokenId][roundId]` is set to a relatively big value;
> * `accumulatedPointsPerAddress[newFighterOwner][roundId]` is 0. the new newFighterOwner won no battle yes so he has no points.
>   Now, when a battele is won fighter accumulates points or reclaimNRN, like usual
>   but when a battle is lost, because `accumulatedPointsPerFighter > accumulatedPointsPerAddress`, the tx will revert with an arithmeticError panic error.
> See the following coded PoC which can be added in RankedBattle.t.sol file:
>
> <details> 
><summary> Click  here for coded PoC </summary>
> ```solidity
> function testWinPointsWithZeroRisk() public {
>         address staker = makeAddr("staker");
>         address alice = makeAddr("alice");
>         
>         _mintFromMergingPool(staker);
>         _fundUserWith4kNeuronByTreasury(staker);
> 
>         // mint token 1 (not 0) to another player and increase totalAccumulatedPoints
>         // this is required to `setNewRound()`
>         _increaseTotalAccumulatedPoints();
>         
>         // prepare token to circumvent `fighterStaked` transfer restriction
>         _circumventTransferRestriction(staker);
> 
>         // staker stake some more to token 0 and 'waits' for a win
>         // to increase `accumulatedPointsPerFighter`
>         vm.prank(staker);
>         _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, 0);
>         vm.prank(address(_GAME_SERVER_ADDRESS));
>         _rankedBattleContract.updateBattleRecord(0, 0, 0, 1500, true);
>         assertEq(_rankedBattleContract.amountStaked(0) >= 3_000 * 1e18, true);
> 
>         // token can be transferred freely
>         vm.prank(staker);
>         _fighterFarmContract.transferFrom(staker, alice, 0);
>         assertEq(_fighterFarmContract.ownerOf(0), alice);
>                                                                         //~ 1500 * sqrt(3_000)
>         assertEq(_rankedBattleContract.accumulatedPointsPerFighter(0, 1) == 1500 * 54, true, "fighter points < 54 * 1500");
>         assertEq(_rankedBattleContract.accumulatedPointsPerAddress(alice, 1) == 0, true, " alice points != 0");
> 
>         // this should not revert
>         vm.expectRevert(abi.encodeWithSignature("Panic(uint256)", 0x11)); //arithmeticError
>         vm.prank(address(_GAME_SERVER_ADDRESS));
>         _rankedBattleContract.updateBattleRecord(0, 0, 2, 1500, true);
> 
>         // a won battle doesn't revert
>         vm.prank(address(_GAME_SERVER_ADDRESS));
>         _rankedBattleContract.updateBattleRecord(0, 0, 0, 1500, true);
>         assertEq(_rankedBattleContract.accumulatedPointsPerFighter(0, 1) > 1500 * 54, true);
>     }
> 
> // helper functions 
>     function _increaseTotalAccumulatedPoints() private {
>         uint256 tokenId = 1;
>         address hellen = makeAddr("hellen");
> 
>         _mintFromMergingPool(hellen);
>         _fundUserWith11kNeuronByTreasury(hellen);
> 
>         vm.prank(hellen);
>         _rankedBattleContract.stakeNRN(3_000 * 10 ** 18, tokenId);
>         assertEq(_rankedBattleContract.amountStaked(tokenId), 3_000 * 10 ** 18);
> 
>         vm.prank(address(_GAME_SERVER_ADDRESS));
>         // 0 == win
>         _rankedBattleContract.updateBattleRecord(tokenId, 50, 0, 1500, true);
>         assertEq(_rankedBattleContract.totalAccumulatedPoints(0) > 0, true);
>     }
> 
>     function _circumventTransferRestriction(address staker) private {
>         // staker loose one battle 
>         vm.prank(staker);
>         _rankedBattleContract.stakeNRN(1_000 * 10 ** 18, 0);
>         vm.prank(address(_GAME_SERVER_ADDRESS));
>         // battleResult 2 == loose
>         //              1 == ties
>         //              0 == win
>         _rankedBattleContract.updateBattleRecord(0, 0, 2, 1500, true);
>     
>         // staker unstake all his NRN
>         uint256 amountStakedLeft = _rankedBattleContract.amountStaked(0);
>         vm.prank(staker);
>         _rankedBattleContract.unstakeNRN(amountStakedLeft, 0);
>         assertEq(_rankedBattleContract.amountStaked(0), 0, "amountStacked after unstake");
> 
>         
>         vm.prank(address(_GAME_SERVER_ADDRESS));
>         _rankedBattleContract.updateBattleRecord(0, 0, 0, 1500, true);
>         _rankedBattleContract.setNewRound();
> 
>         // token 0 has amountStaked > 0 but hasUnstaked for the new roundId is 'false' -> Not ok
>         assertEq(_rankedBattleContract.amountStaked(0), 0.001 * 1e18, "amountStacked after unstake and new round");
>         assertEq(_fighterFarmContract.fighterStaked(0), false, "token 0 is staked");
>     }
> ```

> </details>
>
> Malicious user can stake more NRN to ensure his `stakingFactor` is increased to mantain the following true: `pointsToLooseThisBattle > accumulatedPointsPerAddress[fighterOwner][roundId]` Or he can always transfer the fighter to new addresses with
> 
> `accumulatedPointsPerAddress[fighterOwner][roundId]` == 0.
> 
> ## Tools Used
> ## Recommended Mitigation Steps
> There are 2 things that lead to this problem
> 
> * a fighter with a `stakeAtRisk` balance was transferred and
> * the fighter had points accumulated `accumulatedPointsPerFighter[owner]{roundId]` > 0;
> 
> To give the owner flexibility to transfer his NFT when he wishes, consider implementing a new function which will set the nft into a transferable state.
> 
> This function should:
> 
> * unstake any amountStake left;
> * clear any stakeAtRisk left for current round;
> 
> When the token is transferred :
> 
> * subtract `accumulatedPointsPerFighter[tokenId][roundId]` from `accumulatedPointsPerAddress` and `totalAccumulatedPoints`;
> * and set `accumulatedPointsPerFighter[tokenId][roundId] = 0`
> 
> ## Assessed type
> Invalid Validation



### L-01 `attributeProbabilities` are set twice in AiArenaHelper constructor;
Remove `addAttributeProbabilities()` [function call](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L45) from `AiArenaHelper`  constructor. 

```solidity
    constructor(uint8[][] memory probabilities) {
        _ownerAddress = msg.sender;

        // Initialize the probabilities for each attribute
        addAttributeProbabilities(0, probabilities);// @audit-qa remove this function call

        uint256 attributesLength = attributes.length;
        for (uint8 i = 0; i < attributesLength; i++) {
            attributeProbabilities[0][attributes[i]] = probabilities[i];// because you are setting attr prob here
            attributeToDnaDivisor[attributes[i]] = defaultAttributeDivisor[i];
        }
    } 
```

### L-02 `MINTER_ROLE` can mint back any burned NRN. 
Neuron contract [expose](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/Neuron.sol#L163-L165) externally the `burn` function, allowing any NRN holder to destroy tokens they hold. The maximum total supply should be decreased by the amount of burned tokens. 
Additionally, in `mint` function use `<=` instead `<` to better reflect what `MAX_SUPPLY` means. 

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/Neuron.sol#L156

### L-03 Once staked, a fighter is considered staked even if its `amountStaked` balance == 0. 
To allow transferring a previously staked fighter, a player must call `unstakeNRN` even if `amountStaked` for that fighter is 0. 
Consider improving the player UX and read the `stakedBalance` directly. 

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L286

### L-04 Create an Enum with all possible battle results
Currently magic numbers 0, 1, and 2 are used as battle results. 
Consider creating and using an enum instead. Moreover avoid using value 0 as a valid battle results:
```solidity
    enum battleResult {
        invalidResult,
        won,
        tie,
        lost
    }

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L440

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L505-L513

### L-05 Allows only voltage spender to consume energy
Currently any addresses can call `spendVoltage` to reduce its energy. 
Even if unlikely, this can open the door for abuses. Consider keeping only selected addresses to spend voltage 
```solidity

    function spendVoltage(address spender, uint8 voltageSpent) public {
-        require(spender == msg.sender || allowedVoltageSpenders[msg.sender]);
+        require(allowedVoltageSpenders[msg.sender]);
...
```
https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/VoltageManager.sol#L106

### L-06 Players can get an advantage and mint game items with daily allowances/  with multiple addresses 
GameItems contract implements a few restrictions related to the amount of items can be minted daily per address. Players can get an advantage by minting items with daily allowance. Moreover, there is an `transferable` item options.
Consider  to allow only fighter owners to mint items which can't be transferred  or which have an daily allowance.  

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/GameItems.sol#L224

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/GameItems.sol#L312-L314

### L-07 Missing length check
`iconsTypes` array length check is missing but all other arrays length is checked. 
https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L233-L248


### NC-01 Missing @param natspec

- `iconsTypes` from `RedeemMintPass`
https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/FighterFarm.sol#L236

 - `generation` from `createPhysicalAttributes`
https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/AiArenaHelper.sol#L85