ANALYZING FRAX CONTRACT 

SUMMARY OF THE PROJECT
Frax is a stablecoin running on the Ethereum chain. Part of its supply is backed by collateral and the other part algorithmic. The ratio of collateralized and algorithmic values depend on the market's pricing of the FRAX stablecoin. If FRAX is trading at above $1, the protocol decreases the collateral ratio. If FRAX is trading at under $1, the protocol increases the collateral ratio.


 IMPORTED CONTRACTS AND LIBRARIES

1.0 Context.sol;
This retrns the details of the actual user of the contract and the amount they are transacting. This is necessary because the contract implements gasless transactions.

2.0 ERC20Custom.sol;
The most notable convention in this contract is the emission of an 'Approval' event on calls to transferFrom. According to the author, this is consumed by the app to reconstruct the allowance for all accounts.

:NOTICE: Potential bug	<RED>

/*

  function _transfer(address sender, address recipient, uint256 amount) internal virtual {
        require(sender != address(0), "ERC20: transfer from the zero address");
        require(recipient != address(0), "ERC20: transfer to the zero address");

        _beforeTokenTransfer(sender, recipient, amount);

        _balances[sender] = _balances[sender].sub(amount, "ERC20: transfer amount exceeds balance");
        _balances[recipient] = _balances[recipient].add(amount);
        emit Transfer(sender, recipient, amount);
    }
	*/

this function does not check that the sender has the amount in its balance before performing the transfer... is this not a potential risk since we are dealing with ERC20 token and not Ether?

/*

 function _approve(address owner, address spender, uint256 amount) internal virtual {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");

        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
	*/

Again, same observation applies here; no checking of balance of the owner before approval.

NOTICE: potential mild risk	<YELLOW>

/*

 function _burnFrom(address account, uint256 amount) internal virtual {
        _burn(account, amount);
        _approve(account, _msgSender(), _allowances[account][_msgSender()].sub(amount, "ERC20: burn amount exceeds allowance"));
    }

*/
---- why call _burn function before _approve? shouldn't an account be approved before it can have access to the tokens to burn?


3.0 Owned.sol;
This simply sets the owner of the contract, and also assigns a nominated owner. So, apparently, the owner of the contract can nominate another account to become the owner of the contract, after the nominated account has accepted ownership though.

NOTICE: Question.
/*
	  function acceptOwnership() external {
        require(msg.sender == nominatedOwner, "You must be nominated before you can accept ownership");
        emit OwnerChanged(owner, nominatedOwner);
        owner = nominatedOwner;
        **nominatedOwner = address(0);
    }

*/
why is the nominated owner set to address(0)?

4.0 FXS.sol;
This serves as a base contract for the frax contract. It holds functions that mint, checks votes, handles transfer of FRAX and burning of FRAX as well.

NOTICE: Clarification

/*

  function transferFrom(address sender, address recipient, uint256 amount) public virtual override returns (bool) {
        if(trackingVotes){
            // Transfer votes
            trackVotes(sender, recipient, uint96(amount));
        }

        _transfer(sender, recipient, amount);
        _approve(sender, _msgSender(), _allowances[sender][_msgSender()].sub(amount, "ERC20: transfer amount exceeds allowance"));

        return true;
    }
*/

why call _transfer function before _approve? Is it not the other way round? and why update _allowances inside the approve function? this can be done in another line of code.

/* 

checkpoints[voter][nCheckpoints] = Checkpoint(blockNumber, newVotes);

*/

why is a struct Checkpoint acting as a function here by taking in parameters?


5.0 UniswapPairOracle.sol;
/*

   price0Average = FixedPoint.uq112x112(uint224((price0Cumulative - price0CumulativeLast) / timeElapsed));
        price1Average = FixedPoint.uq112x112(uint224((price1Cumulative - price1CumulativeLast) / timeElapsed));


	*/
	
 NOTICE: Please what does these lines of code do? I can't understand anything from them.
	
	
 This contract is basically a Uniswap price oracle. Checks for price update after a set period and updates the pair price. To save gas, it first checks if there is any change in price variables, and updates the price if there is any change.
 
 6.0 ChainlinkETHUSDPriceConsumer.sol;
 
 /*
 
    function getLatestPrice() public view returns (int) {
        (uint80 roundID, int price, , uint256 updatedAt, uint80 answeredInRound) = priceFeed.latestRoundData();
        require(price >= 0 && updatedAt!= 0 && answeredInRound >= roundID, "Invalid chainlink price");
        
        return price;
    }

    function getDecimals() public view returns (uint8) {
        return priceFeed.decimals();
    }
 
 */
 
 Chainlink price feed for getting price of assets in real time. I discovered important tweaks to the functions above that will help in managing the quality of the feedback received from the oracle, especially if they contain errors or if they are invalid.
 
 
 7.0  AccessControl.sol;
 This contract handles role management for the protocol. It has functions to set role, grant role, assign role, revoke role, and one for an admin to also renounce role.
 
 >>	THE MAIN CONTRACT ANALYSIS 	<<
 
 FRAX.SOL
 
 The contract has a number of state variables listed at the top. It sets a collateral ration of 1e6 (1_000_000) to handle decimals, and in the constructor it passes the creator's address to the inherited contract Owned so it can set the ownership at compile time. '_msg.sender()' is used in place of 'msg.sender' because the contract implements a gasless transaction mechanism.
 
 
 FUNCTIONS:=
 
 0. oracle_price() 
 This takes in an enum as a parameter to enable the admin select which token they want to query. The getLatestPrice function is called to retrieve the current price of ETH-USD, which is then used to determine the amount of Frax or FXS returned against ETH.
 
 
 1. frax_price()
 Calls oracle_price() and returns the price of Frax.
 
 2. fxs_price()
 Calls oracle_price() and returns the price of Fxs.
 
 3. frax_info()
 Returns states of all variables holding details about either Frax or Fxs: 
 frax_price, fxs_price, totalSupply, global_Collateral_ratio, globalCollateralValue, minting_fee, redemption_fee, Eth_current_price.
 
 4. globalCollateralValue()
  Loops through the entire pools and returns all value of collaterals in all the pools. It using a for loop and adds up all the collaterals in each of the pools as it loops through.
  
 5. refreshCollateralRatio()
 First checks that the collateral_ratio_paused variable is set to false, reverts otherwise. It also checks that the cooldown time has elapsed before it proceeds.
 
 This function balances the collateral ratio of Frax based on its relation to price_target and price_band. If the current price of the coin is greater than the price target and the set bandwidth, and if the collateral ratio is less than or equals frax_step (the amount to change in the collaterization ratio), then the collateral ratio is reduced to zero; on the contrary, if the collateral ratio is greater than the frax_step, then the frax_step is substracted from the collateral ratio.
 
 ...The second condition is if the frax price is lesser than the price target and the set bandwidth. If true, the collateral ratio is increased by adding two mechanisms:
 
 i. If the collateral ratio when added frax-step is greater or equals 1e6 (price precision) the collateral ratio is set to the price precision.
 
 ii. If collateral ratio plus frax_step is less than price precision, then frax_step is added to the collateral ratio.
 
 Finally the time of the update is recorded and event emitted.
 
 6. pool_burn_from()
 
 Takes an address and an amount of the stablecoin to be burnt, removes the amount of the coin from circulation and substracts the amount form totalSupply. Calls an external function _burnFrom() on ERC20Custom.sol
 
 7. pool_mint()
 Serves as the minter for the Frax protocol, uses the _mint function from ERC20Custom.sol
 
 8. addPool()
 Adds assets to the protocol staking pool.
 
 9. removePool()
 Removes assets from the staking pool; loops through the pool array and sets the target asset's address to address zero.
 
 10. setRedemptionFee()
 Sets fee that users pay to redeem stakes on the platform.
 
 11. setMintingFee()
 Sets the amount of fee users pay to mint Frax on the platform. Users can mint Frax if they have FXS and one other token specified on the platform.
 
 12. setFraxStep()
Sets the frax_step... which is the range to change the collaterization ratio when the refreshCollateralRation() is called. 
 
 13. setPriceTarget()
 Sets the target at which refreshCollateralRatio() will be triggered. Keeps the Frax price within the stable range, relatively.
 
 14. setRefreshCooldown()
 Sets the period for which the refreshCollateralRatio() must wait before it can be called again.
 
 15. setFXSAddress()
 Sets the FXS coin address.
 
 16. setETHUSDOracle()
 Sets the address of the Chainlink Eth-USD Aggregator.
 
 17. setTimeLock()
 Sets the timelock governance address. This address has an admin role in the protocol.
 
 18. setController()
 Sets the address of the Controller. This address has the role of setting and updating some parameters in the protocol.
 
 19. setPriceBand()
 Sets the price range within which the collateral ration is bound. The ratio cannot exceed this band.
 
 20. setFRAXEthOracle()
 Sets the addresses for updating the price of Frax against ETH. This used UniswapPairOracle to get the latest price, and then converts based on the decimals of the Frax coin and ETH.
 
 21. setFXSethOracle()
 Same as setFRAXEthOracle(), but for updating the FXS coin.
 
 22. toggleCollateralRatio()
 Used to pause or play collateral ratio. If set to true, then the refreshCollateralRatio() cannot be called. 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 