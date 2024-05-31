# M-01 The deposited amount is included in how `rsEthAmountToMint` is calculated and it should not. Second depositors get less rsETH shares than deserved

> # Lines of code
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L136 
>
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L109
>
>  https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTOracle.sol#L70 
> 
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L49
> 
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L79
> 
> # Vulnerability details
> ## Impact
> All deposits, starting with the second one, incur a loss in the received rsETH amount.
> 
> ## Proof of Concept
> `LRTDepositPool::depositAsset` helps users to stake LST in exchange for rsETH shares. First the LST is transferedFrom user to depositPool and rsETH is minted to depositor.
> 
> ```solidity
> ...
>         if (!IERC20(asset).transferFrom(msg.sender, address(this), depositAmount)) {
>             revert TokenTransferFailed();
>         }
>         // interactions
>         uint256 rsethAmountMinted = _mintRsETH(asset, depositAmount);
> ```
> 
> The amount of rsETH to mint is [calculated](https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L109) by `getRsETHAmountToMint` as following:
> 
> ```solidity
>  rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();
> ```
> 
> `lrtOracle.getRSETHPrice` calls `LRTDepositPool::getTotalAssetDeposits` -> `getAssetDistributionData` to get LST assets amounts distributed among depositPool, NDCs and eigenLayer to calculate the rsETH/ETH exchange rate.
> 
> ```solidity
>     function getAssetDistributionData(address asset)
>         public
>         view
>         override
>         onlySupportedAsset(asset)
>         returns (uint256 assetLyingInDepositPool, uint256 assetLyingInNDCs, uint256 assetStakedInEigenLayer)
>     {
>         // Question: is here the right place to have this? Could it be in LRTConfig?
> 
>         // @audit includes the transferred amount from the depositor in the depositAsset() function 
>         assetLyingInDepositPool = IERC20(asset).balanceOf(address(this));
>         uint256 ndcsCount = nodeDelegatorQueue.length;
>         for (uint256 i; i < ndcsCount;) {
>             assetLyingInNDCs += IERC20(asset).balanceOf(nodeDelegatorQueue[i]);
>             assetStakedInEigenLayer += INodeDelegator(nodeDelegatorQueue[i]).getAssetBalance(asset);
>             unchecked {
>                 ++i;
>             }
>         }
>     }
> ```
> 
> So rsETHPrice is calculated as: `rsETHPrice = totalAssetDeposits * assetPrice / rsEthSupply
> 
> The issue rely in including the current deposit amount in the rsETH/ETH price calculation and then in rsETH amounts to mint formula.
> 
> Let's take the following example: Note: to simplify I approximate the LST/ETH ratio to 1e18.
> 
> * Alice (first depositor) deposit 1 ether LST. rsETH totalSupply is 0 => `getRSETHPrice` returns 1 ether.
>   Alice will mint:
>   rsEthAmountToMint = 1 ether * 1e18 / (1 ether * 1e18 / 1 ether)
>   rsEthAmountToMint = 1 ether rsETH
> * Bob deposit 10 ether LST and will mint:
>   rsEthAmountToMint = 10 ether * 1e18 / ( (1 ether + 10 ether) * 1e18 /1 ether)
>   rsEthAmountToMint  = 10/11 ether rsETH
> 
> Alice deposited 1 ether LSD and got 1 ether rsETH Bob deposited 10 ether LSD and got less than 1 ether rsETH.
> 
> ## Tools Used
> Manual Review
> 
> ## Recommended Mitigation Steps
> Update `LRTDepositPool::depositAsset` : save rsethAmountToMint, transfer assets, mint rsETH shares to user:
> 
> ```solidity
>     function depositAsset(
>         address asset,
>         uint256 depositAmount
>     )
>         external
>         whenNotPaused
>         nonReentrant
>         onlySupportedAsset(asset)
>     {
>         // checks
>         if (depositAmount == 0) {
>             revert InvalidAmount();
>         }
>         if (depositAmount > getAssetCurrentLimit(asset)) {
>             revert MaximumDepositLimitReached();
>         }
> +       rsethAmountToMint = getRsETHAmountToMint(_asset, _amount);
> 
>         if (!IERC20(asset).transferFrom(msg.sender, address(this), depositAmount)) {
>             revert TokenTransferFailed();
>         }
> 
>         // interactions
> -        uint256 rsethAmountMinted = _mintRsETH(asset, depositAmount);
> +        address rsethToken = lrtConfig.rsETH();
> +        IRSETH(rsethToken).mint(msg.sender, rsethAmountToMint);
> 
>         emit AssetDeposit(asset, depositAmount, rsethAmountMinted);
>     }
> ```
> 
> ## Assessed type
> Other

# M-02 Inflation attack can cause early users to lose their deposit

> # Lines of code
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L141 
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L152 
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L109 
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTOracle.sol#L56-L58 
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTOracle.sol#L52-L79 
> https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L79
> 
> # Vulnerability details
> ## Impact
> The first user that deposit can artificially increase the `getRSETHPrice` and by doing so he manipulates `getRsETHAmountToMint`. Subsequent depositors will receive much less rsETH shares than expected. On some scenarios users can receive 0 shares.
> 
> ## Proof of Concept
> `LRTDepositPool.sol::depositAsset` helps users to stake LST in exchange for rsETH shares. To get the amount of rsETH to mint, `getRsETHAmountToMint` is [used](https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L95-L110) :
> 
> ```solidity
>     function getRsETHAmountToMint(
>         address asset,
>         uint256 amount
>     )
>         public view override returns (uint256 rsethAmountToMint)
>     {
>         // setup oracle contract
>         address lrtOracleAddress = lrtConfig.getContract(LRTConstants.LRT_ORACLE);
>         ILRTOracle lrtOracle = ILRTOracle(lrtOracleAddress);
> 
>         // calculate rseth amount to mint based on asset amount and asset exchange rate
>         rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();
>     }
> ```
> 
> * `lrtOracle.getAssetPrice(asset)` returns the asset/ETH exchange rate.
> * `lrtOracle.getRSETHPrice` uses LST assets amounts distributed among depositPool, NDCs and eigenLayer to calculate the rsETH/ETH exchange rate.
> 
> Attack scenario:
> 
> * A first depositor (attacker) deposit 1 wei of rEth;
>   a. rsETH totalSupply is 0 => `getRSETHPrice` [returns 1 ether](https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTOracle.sol#L56-L58);
> 
> ```solidity
>         if (rsEthSupply == 0) {
>             return 1 ether;
>         }
> ```
> 
> b. rETH/ETH ratio is > 1 (~ [1.09 * 1e18](https://etherscan.io/address/0x536218f9E9Eb48863970252233c8F271f554C2d0#readContract) more precise); c. attacker mint `rsethAmountToMint = 1 * 1.09 * 1e18 / 1e18 = 1` wei of rsETH.
> 
> * Attacker transfer 1 ether of rETH to `LRTDepostitPool` or directly to any `NodeDelegator`.
> * Since `getAssetDistributionData` is using `balanceOf` ( [1](https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L79) [2](https://github.com/code-423n4/2023-11-kelp/blob/f751d7594051c0766c7ecd1e68daeb0661e43ee3/src/LRTDepositPool.sol#L83)) to get the LST amounts from these 2 contracts the rsEth price calculated by `getRSETHPrice` will be:
>   totalAssetAmt * assetER/ rsEthSupply = (1 wei + 1 ether) * (1.09 * 1e18) / 1 wei ~= 1.09 * 1e36
> * The next user deposit 1 ether of LSD and will mint:
>   `rsethAmountToMint = (amount * lrtOracle.getAssetPrice(asset)) / lrtOracle.getRSETHPrice();` <=>
>   1 ether * (1.09 * 1e18) /(1.09 * 1e36) = 1 wei rsEth.
>   Instead if the second staker deposit only 0.9 ether of LST, the rsETH minted amount rounds down to 0. The attacker has all rsETH shares and victim none.
> 
> ## Tools Used
> Manual review,
> 
> ## Recommended Mitigation Steps
> Since one of the LST tokens used by protocol is stETH (rebase token) an internal account solution to get the balances is excluded. Instead a simple solution is to deposit a small amount of LST when contract is initialized and transfer the minted rsETH to a null address. Even if protocol doesn't use ERC4626 (but a similar approach) [these](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3706) details can be usefull.
> 
> ## Assessed type
> ERC4626

