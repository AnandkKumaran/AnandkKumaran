## Impact

In the current implementation of the contract, the buyer's portion of the shareholder rewards from a purchase is not reflected in the buyer's rewards. A token holder who makes purchases may not receive the amount of holder rewards they are entitled to (a portion of their own fee is spent for the purchase that they rightfully earned). This vulnerability causes all the buyer's portions of shareholder rewards to be locked in the contract forever, and nobody can use or withdraw that fund. The amount of unusable funds will accumulate over time.

## Proof of Concept

The `buy()` function is designed in a way that the buyer is not eligible to receive their portion of the shareholder reward. Line numbers [154, 155](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L154C9-L155C132) state that this is a deliberate design decision. However, shareholder rewards are calculated based on the token circulation, which includes a portion of the buyer [Refer to line 290](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L290). All token holders, except the current buyer, can claim their portion of the shareholder reward. However, the buyer's portion of the reward will be locked in the contract forever due to the lack of functionality for the buyer or contract owner to claim it.

```solidity
File: 1155tech-contracts/src/Market.sol

150:    function buy(uint256 _id, uint256 _amount) external {
151:        require(shareData[_id].creator != msg.sender, "Creator cannot buy");
152:        (uint256 price, uint256 fee) = getBuyPrice(_id, _amount); // Reverts for non-existing ID
153:        SafeERC20.safeTransferFrom(token, msg.sender, address(this), price + fee);
154: @>     // The reward calculation has to use the old rewards value (pre fee-split) to not include the fees of this buy
155: @>     // The rewardsLastClaimedValue then needs to be updated with the new value such that the user cannot claim fees of this buy
156:        uint256 rewardsSinceLastClaim = _getRewardsSinceLastClaim(_id);
157:        // Split the fee among holder, creator and platform
158: @>     _splitFees(_id, fee, shareData[_id].tokensInCirculation);
159:        rewardsLastClaimedValue[_id][msg.sender] = shareData[_id].shareHolderRewardsPerTokenScaled;

280:    function _splitFees(
281:        uint256 _id,
282:        uint256 _fee,
283:        uint256 _tokenCount
284:    ) internal {
285:        uint256 shareHolderFee = (_fee * HOLDER_CUT_BPS) / 10_000;
286:        uint256 shareCreatorFee = (_fee * CREATOR_CUT_BPS) / 10_000;
287:        uint256 platformFee = _fee - shareHolderFee - shareCreatorFee;
288:        shareData[_id].shareCreatorPool += shareCreatorFee;
289:        if (_tokenCount > 0) {
290: @>         shareData[_id].shareHolderRewardsPerTokenScaled += (shareHolderFee * 1e18) / _tokenCount;
```

*GitHub* : [150](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L150C5-L159C100), [280](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L280C5-L290C102)

### POC

1. **Alice** is a holder of tokens for a particular share ID.
2. **Alice** decides to buy more tokens of the same share ID. She calls the buy function. [Line: 150](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L150)
3. In the buy function, the contract first calculates the rewards **Alice** has earned since her last claim (`rewardsSinceLastClaim`). This is based on the `rewardsLastClaimedValue` for **Alice**. [Line: 156](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L156)
4. The contract then proceeds to split the fees from **Alice**'s purchase. A portion of this fee is added to `shareHolderRewardsPerTokenScaled` in `_splitFees()`. [Line: 290](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L290)
5. The contract then updates `rewardsLastClaimedValue` for **Alice** to the current `shareHolderRewardsPerTokenScaled`. This will restrict claim her portion of the reward. [Line: 159](https://github.com/code-423n4/2023-11-canto/blob/335930cd53cf9a137504a57f1215be52c6d67cb3/1155tech-contracts/src/Market.sol#L159)

This indicates that the holder fee from **Alice**'s purchase is excluded from her `rewardsSinceLastClaim` when she makes a purchase. Consequently, this holder fee is irreversibly lost, and Alice is not receiving the rightful amount of rewards from her purchase. Moreover, nobody has the capability to withdraw this amount from the contract.

#### Sequence Diagram

```seq
Note left of Alice: Alice holds: 100
Alice->Market.sol: Buy 100 
Note right of Market.sol: Token in circulation: 1,000\n Fee: 10 \n CREATOR_CUT_BPS: 3,300 \n Alice portion: (Alice holdings) * (fee * CREATOR_CUT_BPS )/(10000 * token in circulation)\n = (100) * (10 * 3,300) / (10,000 * 1,000) \n = 0.33
Market.sol-->Alice: Send 100
Note right of Market.sol: 0.33 token will be locked in contract
```

Referencing the aforementioned sequence diagram, Alice, a token holder with 100 tokens, purchases an additional 100 tokens. However, a portion of her fee, specifically 0.33, becomes permanently locked in the contract, unable to be retrieved.

## Tools Used

Manual Review

## Recommended Mitigation Steps

1. Revise the `buy()` function to enable the buyer to claim their holder rewards.
OR
2. Include the buyer's portion of holder rewards in the platform reward, allowing the platform owner to claim it alongside the platform fee.
