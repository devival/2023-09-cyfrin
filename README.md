# Foundry DeFi Stablecoin CodeHawks Audit Contest - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. `getTokenAmountFromUsd` will return wrong value if the oracle prices are negative ](#M-01)
    - ### [M-02. Cannot liquidate positions if the oracle prices are equal zero](#M-02)
    - ### [M-03. DSCEngine would be unusable if token's Chainlink update interval (heartbeat threshold period) is longer than 3 hours](#M-03)

- ## Gas Optimizations / Informationals
    - ### [G-01. Tautology check for address(0) in `mint` function](#G-01)
    - ### [G-02. Unnecessary check for negative numbers in `burn` and `mint` functions](#G-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Cyfrin

### Dates: Jul 24th, 2023 - Aug 5th, 2023

[See more contest details here](https://www.codehawks.com/contests/cljx3b9390009liqwuedkn0m0)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 3
   - Low: 0
  - Gas/Info: 2


		
# Medium Risk Findings

## <a id='M-01'></a>M-01. `getTokenAmountFromUsd` will return wrong value if the oracle prices are negative             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L345-L347

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L363

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L366

## Summary
In the unlikely event that the price of an asset goes negative, the `getTokenAmountFromUsd` function will convert the `int256` result to `uint256` and return the invalid result, leading to a loss of funds.

## Vulnerability Details
The Chainlink Data Feeds use `int` instead of `uint` because some prices can be negative, like when [oil futures dropped below 0](https://www.ig.com/en/news-and-trade-ideas/what-do-negative-oil-prices-mean--200507). [Source](https://stackoverflow.com/questions/67094903/anybody-knows-why-chainlinks-pricefeed-return-price-value-with-int-type-while)

The usage of off-chain oracle prices in `DSCEngine.sol`:
```
    function getTokenAmountFromUsd(address token, uint256 usdAmountInWei) public view returns (uint256) {
        // price of ETH (token)
        // $/ETH ETH ??
        // $2000 / ETH. $1000 = 0.5 ETH
        AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
@>      (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();
        // ($10e18 * 1e18) / ($2000e8 * 1e10)
@>      return (usdAmountInWei * PRECISION) / (uint256(price) * ADDITIONAL_FEED_PRECISION);
    }
```
```
    function getUsdValue(address token, uint256 amount) public view returns (uint256) {
        AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
@>      (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();
        // 1 ETH = $1000
        // The returned value from CL will be 1000 * 1e8
@>      return ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;
    }
```

## Impact
Positions won't be liquidatable, and DSC system will become insolvent.

## Tools Used
Manual review

## Recommendations
Provide a mechanism for positions to be liquidated even if the price becomes negative.

## <a id='M-02'></a>M-02. Cannot liquidate positions if the oracle prices are equal zero            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DSCEngine.sol#L345-L347

## Summary
In the unlikely event that the price of an asset reaches zero, there is no way to liquidate the position, because the `getTokenAmountFromUsd` function will revert.

## Vulnerability Details
The usage of off-chain oracle prices in `DSCEngine.sol` that can lead to a possible division by zero (an exception that will result in a reverted transaction):
```
    function getTokenAmountFromUsd(address token, uint256 usdAmountInWei) public view returns (uint256) {
        // price of ETH (token)
        // $/ETH ETH ??
        // $2000 / ETH. $1000 = 0.5 ETH
        AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);
@>      (, int256 price,,,) = priceFeed.staleCheckLatestRoundData();
        // ($10e18 * 1e18) / ($2000e8 * 1e10)
@>      return (usdAmountInWei * PRECISION) / (uint256(price) * ADDITIONAL_FEED_PRECISION);
    }
```

## Impact
Positions won't be liquidatable, at an extremely critical moment that they should be liquidatable. Losses and fees will grow and DSC system will become insolvent.

## Tools Used
Manual review

## Recommendations
Provide a mechanism for positions to be liquidated even if the price reaches zero or goes negative.

## <a id='M-03'></a>M-03. DSCEngine would be unusable if token's Chainlink update interval (heartbeat threshold period) is longer than 3 hours            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/libraries/OracleLib.sol#L19

## Summary
The current `TIMEOUT` value in `libraries/OracleLib.sol` would often lead to a temporary freeze of all users' funds after the 3 hours of no price deviation above the threshold.

## Vulnerability Details
There are currently 2 "trigger" parameters that kick off Chainlink nodes to update. 
A deviation parameter: The Chainlink nodes are monitoring the prices of the assets off-chain. If the real-world price of an asset deviates past some interval, it will trigger all the nodes to do an update. Right now, most Ethereum data feeds have a 0.5% deviation threshold.

A time interval: If the price stays within the deviation parameters, it will only trigger an update every X minutes / hours. It is also known as a heartbeat.
[Source: Chainlink](https://blog.chain.link/chainlink-price-feeds-secure-defi/)

Many Chainlink token price have a heartbeat threshold period longer than 3 hours, this includes [BNB/USD](https://data.chain.link/ethereum/mainnet/crypto-usd/bnb-usd) - 4 hours, [DOT/USD](https://data.chain.link/ethereum/mainnet/crypto-usd/dot-usd) 24 hours, [DOGE/USD](https://data.chain.link/ethereum/mainnet/crypto-usd/doge-usd) 24 hours.

However, the [OracleLib.sol](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/libraries/OracleLib.sol#L19) has a `TIMEOUT` constant set to 3 hour period which is used to identify if the price data is stale or not.

## Impact

Lets assume the contracts will use ChainLink price feed with a heartbeat threshold period set to 24 hours and price deviation threshold of 1%. If there is no price deviation more than 1% during that period, DSCEngine would stay frozen for 21 out 24 hours in the day.

## Tools Used
Manual review

## Recommendations
Provide a mechanism for a `TIMEOUT` to be adjusted to each token's price feed separately (can be set in the constructor). Consider creating a mapping priceFeed => timeout if there is more than one token price feed.




# Gas Optimizations / Informationals

## <a id='G/I-01'></a>G/I-01. Tautology check for address(0) in `mint` function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DecentralizedStableCoin.sol#L58

## Summary
In contract `DecentralizedStableCoin.sol`, tautology expression has been detected. Such
expression is of no use since it is already being evaluated in ERC20 abstract contract 
[ERC20.sol#L283](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/19293f3ecdb20a7f44d54279b5c1ddbb84de4a2e/contracts/token/ERC20/ERC20.sol#L283)

## Vulnerability Details
Mint function in `DecentralizedStableCoin.sol`
```
    function mint(address _to, uint256 _amount) external onlyOwner returns (bool) {
        if (_to == address(0)) {
            revert DecentralizedStableCoin__NotZeroAddress();
        }
        if (_amount <= 0) {
            revert DecentralizedStableCoin__MustBeMoreThanZero();
        }
        _mint(_to, _amount);
        return true;
    }
```
Inherited `_mint` function in ERC20 abstract contract 
```
    function _mint(address account, uint256 value) internal {
        if (account == address(0)) {
            revert ERC20InvalidReceiver(address(0));
        }
        _update(address(0), account, value);
    }
```

## Tools Used
Manual Review

## Recommendations
Remove the following lines in [DecentralizedStableCoin.sol#L58-L59](https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DecentralizedStableCoin.sol#L58-L59)

```
    if (_to == address(0)) {
        revert DecentralizedStableCoin__NotZeroAddress();
```

## <a id='G/I-02'></a>G/I-02. Unnecessary check for negative numbers in `burn` and `mint` functions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DecentralizedStableCoin.sol#L48

https://github.com/Cyfrin/2023-07-foundry-defi-stablecoin/blob/d1c5501aa79320ca0aeaa73f47f0dbc88c7b77e2/src/DecentralizedStableCoin.sol#L61

## Summary
Even though the gas cost for comparison operations such as <= or != is the same, it is recommended to abstrain from the check for negative numbers in a uint type variable as they can never be less than zero. 
Source: [soliditylang](https://docs.soliditylang.org/en/v0.8.20/types.html)

## Vulnerability Details
Unnecessary check for negative numbers in `burn` and `mint` functions of `DecentralizedStableCoin.sol`.
Variable `_amount` cannot be less than 0 because passing negative numbers to uint variables is not possible.

## Tools Used
Manual Review

## Recommendations
Replace `_amount <= 0` with `_amount == 0` in `DecentralizedStableCoin.sol#L48` and `DecentralizedStableCoin.sol#L61`.

