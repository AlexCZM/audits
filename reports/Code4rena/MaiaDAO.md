# H-01 TalosBaseStrategy#init() lacks slippage protection

> ## Lines of code
>  https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L99-L147 
>
>  https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L419-L425
> 
> # Vulnerability details
> `checkDeviation` modifier purpose is to add slippage protection for increase/decrease liquidity operations. It's applied to `deposit/redeem`, `rerange/rebalance` but `init()` is missing it.
> 
> ## Impact
> There is no slippage protection on `init()`.
> 
> ## Proof of Concept
> In the `init()` function of TalosBaseStrategy, the following actions are performed: an initial deposit is made, a tokenId and shares are minted.
> 
> The `_nonfungiblePositionManager.mint()` function is called with hardcoded values of `amount0Min` and `amount1Min`, both set to 0. Additionally, it should be noted that the `init()` function does not utilize the `checkDeviation` modifier, which was specifically designed to safeguard users against slippage.
> 
> ```solidity
>     function init(uint256 amount0Desired, uint256 amount1Desired, address receiver)
>         external
>         virtual
>         nonReentrant
>         returns (uint256 shares, uint256 amount0, uint256 amount1)
>     {
>     ...
>         (_tokenId, _liquidity, amount0, amount1) = _nonfungiblePositionManager.mint(
>             INonfungiblePositionManager.MintParams({
>                 token0: address(_token0),
>                 token1: address(_token1),
>                 fee: poolFee,
>                 tickLower: tickLower,
>                 tickUpper: tickUpper,
>                 amount0Desired: amount0Desired,
>                 amount1Desired: amount1Desired,
>                 amount0Min: 0,
>                 amount1Min: 0,
>                 recipient: address(this),
>                 deadline: block.timestamp
>             })
>         );
>         ...
> ```
> 
> https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L99-L147
> 
> ```solidity
>     /// @notice Function modifier that checks if price has not moved a lot recently.
>     /// This mitigates price manipulation during rebalance and also prevents placing orders when it's too volatile.
>     modifier checkDeviation() {
>         ITalosOptimizer _optimizer = optimizer;
>         pool.checkDeviation(_optimizer.maxTwapDeviation(), _optimizer.twapDuration());
>         _;
>     }
> ```
> 
> https://github.com/Maia-DAO/maia-ecosystem-monorepo/blob/2f6e87348877684aa0c12aec204fea210cfbe6eb/src/scope/talos/base/TalosBaseStrategy.sol#L419-L425
> 
> ## Tools Used
> VS Code, uniswapv3book
> 
> ## Recommended Mitigation Steps
> Apply `checkDeviation` to `init()` function.
> 
> ## Assessed type
> Other


--------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------



# M-01 ERC20Boost.sol An user can be attached to a gauge and have no boost balance.

> # Lines of code
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L150-L172 
>
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L80-L83 
>
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L336-L344
> 
> ## Vulnerability details
> When a user boosted gauge became deprecated, the user can transfer his boost tokens. When the same gauge is reintroduced to the active gauge list, the user will boost it again even if his boost token balance is zero.
> 
> ## Impact
> Same amount of boost tokens can be allocated to gauges by multiple addresses.
> 
> ## Proof of Concept
> Let's take an example:
> 
> 1. Alice calls `attach()` from gaugeA to boost it;
> 
> * `getUserBoost[alice]` is set to balanceOf(alice)
> 
> 2. Owner removes gaugeA and it's added to `_deprecatedGauges`;
> 3. Alice calls `updateUserBoost()`; because gaugeA is now deprecated her allocated boost is set to `userBoost` which is initialized to zero (0) :
> 
> ```solidity
>     function updateUserBoost(address user) external {
>         uint256 userBoost = 0;
>         address[] memory gaugeList = _userGauges[user].values();
>         uint256 length = gaugeList.length;
>         for (uint256 i = 0; i < length;) {
>             address gauge = gaugeList[i];
>             if (!_deprecatedGauges.contains(gauge)) {
>                 uint256 gaugeBoost = getUserGaugeBoost[user][gauge].userGaugeBoost;
>                 if (userBoost < gaugeBoost) userBoost = gaugeBoost;
>             }
>             unchecked {
>                 i++;
>             }
>         }
>         getUserBoost[user] = userBoost;
>         emit UpdateUserBoost(user, userBoost);
>     } 
> ```
> 
> 4. `freeGaugeBoost()` returns the amount of unallocated boost tokens:
> 
> ```solidity
> function freeGaugeBoost(address user) public view returns (uint256) {
> 	return balanceOf[user] - getUserBoost[user];
> }
> ```
> 
> 5. `transfer()` has `notAttached()` modifier that ensures the transferred amount is free (not allocated to any gauge);
> 
> ```solidity
>     /**
>      * @notice Transfers `amount` of tokens from `msg.sender` to `to` address.
>      * @dev User must have enough free boost.
>      * @param to the address to transfer to.
>      * @param amount the amount to transfer.
>      */
>     function transfer(address to, uint256 amount) public override notAttached(msg.sender, amount) returns (bool) {
>         return super.transfer(to, amount);
>     }
> ```
> 
> 6. Alice transfer her tokens
> 7. When gaugeA is added back, `addGauge(gaugeA)` she will continue to boost gaugeA even if her balance is 0
> 
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L150-L172
> 
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L81C1-L83C6
> 
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Boost.sol#L336-L344
> 
> ## Tools Used
> VS Code
> 
> ## Recommended Mitigation Steps
> One solution is `updateUserBoost()` to loop all gauges (active and deprecated) not only the active ones:
> 
> ```solidity
>     function updateUserBoost(address user) external {
>         uint256 userBoost = 0;
>         address[] memory gaugeList = _userGauges[user].values();
> 
> 
>         uint256 length = gaugeList.length;
>         for (uint256 i = 0; i < length;) {
>             address gauge = gaugeList[i];
>             uint256 gaugeBoost = getUserGaugeBoost[user][gauge].userGaugeBoost;
>             if (userBoost < gaugeBoost) userBoost = gaugeBoost;
>             unchecked {
>                 i++;
>             }
>         }
>         getUserBoost[user] = userBoost;
>         emit UpdateUserBoost(user, userBoost);
>     }
> ```
> 
> Even the `updateUserBoost()` comments indicates all `_userGauges` should be iterated over.
> 
> ```solidity
> /**
> * @notice Update geUserBoost for a user, loop through all _userGauges
> * @param user the user to update the boost for.
> */
> function  updateUserBoost(address user) external;
> ```
> 
> ## Assessed type
> Other


--------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------


# M-02 Incorrect accounting of free weight in `_decrementWeightUntilFree`

> ## Lines of code
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Gauges.sol#L514-L555
> 
> ## Vulnerability details
> Free weight is calculated as `userFreeWeight` (already free weights) plus weights freed from non-deprecated gauges. Non-deprecated gauges condition is not required and leads to freeing weight from more gauges than needed.
> 
> ```solidity
>      * @notice A greedy algorithm for freeing weight before a token burn/transfer
>      * @dev Frees up entire gauges, so likely will free more than `weight`
>      * @param user the user to free weight for
>      * @param weight the weight to free
>      */
>     function _decrementWeightUntilFree(address user, uint256 weight) internal nonReentrant {
>         uint256 userFreeWeight = freeVotes(user) + userUnusedVotes(user);
>         // early return if already free
>         if (userFreeWeight >= weight) return;
>         uint32 currentCycle = _getGaugeCycleEnd();
>         
>         // cache totals for batch updates
>         uint112 userFreed;
>         uint112 totalFreed;
>         
>         // Loop through all user gauges, live and deprecated
>         address[] memory gaugeList = _userGauges[user].values();
> 
>         // Free gauges through the entire list or until underweight
>         uint256 size = gaugeList.length;
>         
>         //@audit use userFreed instead of totalFreed
>         for (uint256 i = 0; i < size && (userFreeWeight + totalFreed) < weight;) {
>             address gauge = gaugeList[i];
>             uint112 userGaugeWeight = getUserGaugeWeight[user][gauge];
>             if (userGaugeWeight != 0) {
>                 // If the gauge is live (not deprecated), include its weight in the total to remove
>                 if (!_deprecatedGauges.contains(gauge)) {
>                     totalFreed += userGaugeWeight;
>                 }
>                 userFreed += userGaugeWeight;
>                 _decrementGaugeWeight(user, gauge, userGaugeWeight, currentCycle);
> 
>                 unchecked {
>                     i++;
>                 }
>             }
>         }
>         getUserWeight[user] -= userFreed;
>         _writeGaugeWeight(_totalWeight, _subtract112, totalFreed, currentCycle);
>     }
> ```
> 
> https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-20/ERC20Gauges.sol#L514-L555
> 
> ## Impact
> More gauges than necessary are freed means the user accumulate rewards from fewer gauges.
> 
> ## Proof of Concept
> Let's suppose Alice has 3 weights in total and allocate them to 3 gauges: A, B and C, where gauge A is deprecated.
> 
> 1. Alice calls `_decrementWeightUntilFree(alice, weight = 2)`
> 
> * `userFreeWeight` = 0
> * gauge A is freed, `totalFreed` = 0, `userFreed` = 1
> * (`userFreeWeight` + `totalFreed`) < `weight`, continue to free next gauge
> 
> 2. gauge B is freed, `totalFreed` = 1, `userFreed` = 2
> 
> * (`userFreeWeight` + `totalFreed`) < weight, continue to free next gauge
> 
> 3. gauge C is freed, `totalFreed` = 2, `userFreed` = 3
> 4. All gauges are freed.
> 
> Now consider following case:
> 
> 1. Alice calls `_decrementWeightUntilFree(alice, 1)`
> 
> * `userFreeWeight` = 0
> * gauge A is freed, `totalFreed` = 0, `userFreed` = 1
> * (`userFreeWeight` + `totalFreed`) < `weight`, continue to free next gauge
> 
> 2. gauge B is freed, `totalFreed` = 1, `userFreed` = 2
> 
> * (`userFreeWeight` + `totalFreed`) >= `weight`, break
> 
> 3. Alice calls `_decrementWeightUntilFree(alice, weight = 2)`
> 
> * `userFreeWeight` = 1
> * gauge B is freed, `totalFreed` = 1, `userFreed` = 1
> * (`userFreeWeight` + `totalFreed`) >= weight, break
> 
> 4. Only 2 gauges are freed.
> 
> ## Tools Used
> VS Code
> 
> ## Recommended Mitigation Steps
> The freed weight should be considered as weight released from all gauges, not just limited to the non-deprecated ones.
> 
> ```solidity
> 	// Free gauges through the entire list or until underweight
> 	uint256 size = gaugeList.length;
> 	for (
> 		uint256 i = 0; i < size && (userFreeWeight + userFreed) < weight;
> 	) {
> 	...
> 	}
> ```
> 
> ## Assessed type
> Other

