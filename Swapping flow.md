V3 Defi contract call flow 
>> Deploy All the contract 
	UniswapV3Factory
	SwapRouter
	NonfungiblePositionManager
	NonfungibleTokenPositionDescriptor
	TokenA
	TokenB

>> go to factory contract and Create the pool >> by calling createPool( tokenA.address, tokenB.address, fee('3000')) [there are three fee slot > 0.05%,.3%,.1%]
>> If Pool Created Successfully then we need to inialize the pool with ratio like  [1:1]
	need to call >>> initialize(uint160 sqrtPriceX96) []
	here ..In Uniswap V3, the pool price is stored as a square root value in Q64.96 fixed-point format.
	sqrtPriceX96 = sqrt(price) * 2^96
	lets calculte price is 	price = token1Amount / token0Amount [we want 1:1]
				price = 1
				sqrtPriceX96 = swrt(1)* 2^96 = 79228162514264337593543950336
		
>> Now give approval to NonFungibalePositionManager contract in order to mint Nft and deposit Liquidity
	tokenA.approve(nonFungi.address, amount)
	tokenB.approve(nonFungi.address, amount)
	
>> Add liquidity and mint Nft positions >> go to NonFungiblePositionManager and call mint function
	mint(
	token0: tokenA.address,
  	token1: tokenB.address,
  	fee: 3000, // [for .3 percet tick value is 60]
  	tickLower: nearestTick - tickSpacing * 10, // -600[0-60*10]
  	tickUpper: nearestTick + tickSpacing * 10, // 600 [0+60*10]
  	amount0Desired: expandTo18Decimals(100), [tokenA amount want to add in liquidity [100 ether]]
  	amount1Desired: expandTo18Decimals(100),[tokenB amount want to add in liquidity [100 ether]]
  	amount0Min: 0,
  	amount1Min: 0,
  	recipient: deployer.address,
  	deadline: Math.floor(Date.now() / 1000) + 60 * 10,) // epoch timestamp
  	)

>> Once liquidity added successfully any user can swap token before swap user need to give approval to router contract and then call 
	tokenA.approve(router.address, amount)
	exactInputSingle({
  		tokenIn: tokenA.address,
  		tokenOut: tokenB.address,
  		fee: 3000,
  		recipient: deployer.address,
  		deadline: Math.floor(Date.now() / 1000) + 60 * 10, // epoch timestamp
  		amountIn: expandTo18Decimals(10), [10 ether want to swap ] 
  		amountOutMinimum: 0,
  		sqrtPriceLimitX96: 0,
		});
	
	
