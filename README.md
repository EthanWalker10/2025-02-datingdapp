# DatingDapp


### Prize Pool

- Starts: February 06, 2025 Noon UTC
- Ends: February 13, 2025 Noon UTC

- nSLOC: 197

[//]: # (contest-details-open)

## About the Project

Roses are red, violets are blue, use this DatingDapp and love will find you.! Dating Dapp lets users mint a soulbound NFT as their verified dating profile. To express interest in someone, they pay 1 ETH to "like" their profile. If the like is mutual, all their previous like payments (minus a 10% fee) are pooled into a shared multisig wallet, which both users can access for their first date. This system ensures genuine connections, and turns every match into a meaningful, on-chain commitment.

[//]: # (contest-details-close)

[//]: # (scope-open)

## Scope (contracts)

```js
src/
├── LikeRegistry.sol
├── MultiSig.sol
├── SoulboundProfileNFT.sol
```

## Compatibilities

Chains:
- Ethereum/EVM Equivalent

Tokens:
- ERC721 standard


[//]: # (scope-close)

[//]: # (getting-started-open)

## Setup

Coming soon!

[//]: # (getting-started-close)

[//]: # (known-issues-open)

## Known Issues

None Reported

[//]: # (known-issues-close)


# Audit
1. userBalance 在用户添加喜欢的对象后未增加
2. 链上数据是公开的, attacker 可以获取 userBalance 的信息, 并选择余额多的用户下手
   1. 可以将其 totalReward 修改为按照余额更少的一方添加



# 中文二
链上数据是公开的, attacker 可以获取 `userBalance` 的信息, 并选择余额多的用户进行攻击; 此外, Rewards 的分配机制也容易导致不公平的现象
## Summary
1. 由于链上数据公开, 攻击者可以针对余额高的用户进行定向攻击, 支付 `1 eth` 调用 `likeUser` 以牟取潜在的利益; 
2. 由于每个用户可以匹配多个用户, 那么可能出现这种情况:
   1. userA likes userB
   2. userA likes userC
   3. userB likes userA
   4. userC likes userA
   5. 显然, 此时 userA 的余额已经为 0, 这对于 userC 来说是不公平的

## Vulnerability Details
无论 `userBalances[from]` 和 `userBalances[to]` 之间相差多大, 总会全部清空并计入 `totalRewards` 中, 这对于高净值用户存在被钓鱼攻击的可能性; 另一方面, 也对后续匹配成功的用户不公平
```solidity
function matchRewards(address from, address to) internal {
    uint256 matchUserOne = userBalances[from];
    uint256 matchUserTwo = userBalances[to];
    userBalances[from] = 0;
    userBalances[to] = 0;

    uint256 totalRewards = matchUserOne + matchUserTwo;
    uint256 matchingFees = (totalRewards * FIXEDFEE ) / 100;
    uint256 rewards = totalRewards - matchingFees;
    totalFees += matchingFees;

    // Deploy a MultiSig contract for the matched users
    MultiSigWallet multiSigWallet = new MultiSigWallet(from, to);

    // Send ETH to the deployed multisig wallet
    (bool success, ) = payable(address(multiSigWallet)).call{value: rewards}("");
    require(success, "Transfer failed");
}
```

## Impact
* 高净值用户可能会被 `attacker` 进行定向攻击, 比如某个余额为 100 eth 的用户被攻击者通过多个 1 eth 的账户吸引并欺骗
* `attacker` 本身也可以在一开始就创造多个账户来钓鱼, 这些账户被喜欢的越多, 攻击者能够牟取的利益就越大
* rewards 的分配机制对后面匹配成功的用户不公平


## Tools Used
* Foundry

## Recommendations
将 `totalRewards` 修改为按照余额更少的一方进行计算, 不仅可以抵御上述攻击, 还可以避免不公平的情况
```diff
function matchRewards(address from, address to) internal {
-   uint256 matchUserOne = userBalances[from];
-   uint256 matchUserTwo = userBalances[to];
+   uint256 matchBalance = userBalances[from] < userBalances[to] ? userBalances[from] : userBalances[to];
-   userBalances[from] = 0;
+   userBalances[from] -= matchBalance;
-   userBalances[to] = 0;
+   userBalances[to] -= matchBalance;

    uint256 totalRewards = matchBalance + matchBalance;
    uint256 matchingFees = (totalRewards * FIXEDFEE ) / 100;
    uint256 rewards = totalRewards - matchingFees;
    totalFees += matchingFees;

    // Deploy a MultiSig contract for the matched users
    MultiSigWallet multiSigWallet = new MultiSigWallet(from, to);

    // Send ETH to the deployed multisig wallet
    (bool success, ) = payable(address(multiSigWallet)).call{value: rewards}("");
    require(success, "Transfer failed");
}
```




# 英文二
On-chain data is public, and an attacker can access the `userBalance` information and choose to attack users with higher balances; furthermore, the reward distribution mechanism can also lead to unfair phenomena.

## Summary
1. Due to the public nature of on-chain data, attackers can conduct targeted attacks on users with higher balances, paying `1 ETH` to call `likeUser` in order to exploit potential gains;
2. Since each user can match with multiple users, the following situation may arise:
   1. userA likes userB
   2. userA likes userC
   3. userB likes userA
   4. userC likes userA
   5. Clearly, at this point userA's balance is 0, which is unfair for userC.

## Vulnerability Details
Regardless of the difference between `userBalances[from]` and `userBalances[to]`, they will always be fully cleared and counted towards `totalRewards`, which poses a potential phishing attack risk for high-net-worth users; on the other hand, this is also unfair to users who successfully match later.
```solidity
function matchRewards(address from, address to) internal {
    uint256 matchUserOne = userBalances[from];
    uint256 matchUserTwo = userBalances[to];
    userBalances[from] = 0;
    userBalances[to] = 0;

    uint256 totalRewards = matchUserOne + matchUserTwo;
    uint256 matchingFees = (totalRewards * FIXEDFEE ) / 100;
    uint256 rewards = totalRewards - matchingFees;
    totalFees += matchingFees;

    // Deploy a MultiSig contract for the matched users
    MultiSigWallet multiSigWallet = new MultiSigWallet(from, to);

    // Send ETH to the deployed multisig wallet
    (bool success, ) = payable(address(multiSigWallet)).call{value: rewards}("");
    require(success, "Transfer failed");
}
```

## impact
* High-net-worth users may be targeted by an `attacker`, for example, a user with a balance of 100 ETH could be attracted and deceived by an attacker using multiple 1 ETH accounts.
* The `attacker` himself can also create multiple accounts from the start to fish, and the more these accounts are liked, the greater the potential gains for the attacker.
* The reward distribution mechanism is unfair to users who match later.


## Tools Used
* Foundry

## Recommendations
Modifying `totalRewards` to be calculated based on the user with the smaller balance not only helps defend against the aforementioned attacks but also prevents unfair situations.
```diff
function matchRewards(address from, address to) internal {
-   uint256 matchUserOne = userBalances[from];
-   uint256 matchUserTwo = userBalances[to];
+   uint256 matchBalance = userBalances[from] < userBalances[to] ? userBalances[from] : userBalances[to];
-   userBalances[from] = 0;
+   userBalances[from] -= matchBalance;
-   userBalances[to] = 0;
+   userBalances[to] -= matchBalance;

    uint256 totalRewards = matchBalance + matchBalance;
    uint256 matchingFees = (totalRewards * FIXEDFEE ) / 100;
    uint256 rewards = totalRewards - matchingFees;
    totalFees += matchingFees;

    // Deploy a MultiSig contract for the matched users
    MultiSigWallet multiSigWallet = new MultiSigWallet(from, to);

    // Send ETH to the deployed multisig wallet
    (bool success, ) = payable(address(multiSigWallet)).call{value: rewards}("");
    require(success, "Transfer failed");
}
```



# 中文一
## Summary
用户 A 调用 `likeUser` 并发送 `value > 1` 的 eth, 按照 DatingDapp 的设想, 应该由 `userBalances` 去累计用户 A 的金额. 否则, 后续的计算中, 每个用户的金额都将是 0

## Vulnerability Details
当用户 A 调用 `likeUser` 时, 并没有执行对 `userBalances` 的累加
```solidity
function likeUser(
    address liked
) external payable {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
    require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

    likes[msg.sender][liked] = true;
    emit Liked(msg.sender, liked);

    // Check if mutual like
    if (likes[liked][msg.sender]) {
        matches[msg.sender].push(liked);
        matches[liked].push(msg.sender);
        emit Matched(msg.sender, liked);
        matchRewards(liked, msg.sender);
    }
}
```
最终将导致 `totalRewards` 总是 0, 并影响接下来的所有计算:
```solidity
uint256 totalRewards = matchUserOne + matchUserTwo;
uint256 matchingFees = (totalRewards * FIXEDFEE ) / 100;
uint256 rewards = totalRewards - matchingFees;
totalFees += matchingFees;
```


## POC
```solidity
function testUserBalanceshouldIncreaseAfterLike() public {
    vm.prank(user1);
    likeRegistry.likeUser{value: 20 ether}(user2);
    assertEq(likeRegistry.userBalances(user1), 20 ether, "User1 balance should be 20 ether");
}
```
Then we will get an error:
```shell
[FAIL: User1 balance should be 20 ether: 0 != 20000000000000000000]
```


## Impact
* 用户将无法获得奖励
* 合约拥有者也无法取回合约中的 eth



## Recommendations
在 `likeUser` 函数中增加对 `userBalances` 的处理: 
```diff
function likeUser(
    address liked
) external payable {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
    require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

    likes[msg.sender][liked] = true;
+   userBalances[msg.sender] += msg.value;
    emit Liked(msg.sender, liked);

    [...]
}
```


# 英文一

## Summary
User A calls `likeUser` and sends `value > 1` ETH. According to the design of DatingDapp, the amount for user A should be accumulated by `userBalances`. Otherwise, in the subsequent calculations, the balance for each user will be 0.

## Vulnerability Details
When User A calls `likeUser`, the accumulation of `userBalances` is not performed.
```solidity
function likeUser(
    address liked
) external payable {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
    require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

    likes[msg.sender][liked] = true;
    emit Liked(msg.sender, liked);

    // Check if mutual like
    if (likes[liked][msg.sender]) {
        matches[msg.sender].push(liked);
        matches[liked].push(msg.sender);
        emit Matched(msg.sender, liked);
        matchRewards(liked, msg.sender);
    }
}
```
This will result in `totalRewards` always being 0, affecting all subsequent calculations:
```solidity
uint256 totalRewards = matchUserOne + matchUserTwo;
uint256 matchingFees = (totalRewards * FIXEDFEE ) / 100;
uint256 rewards = totalRewards - matchingFees;
totalFees += matchingFees;
```

## POC
```solidity
function testUserBalanceshouldIncreaseAfterLike() public {
    vm.prank(user1);
    likeRegistry.likeUser{value: 20 ether}(user2);
    assertEq(likeRegistry.userBalances(user1), 20 ether, "User1 balance should be 20 ether");
}
```
Then we will get an error:
```shell
[FAIL: User1 balance should be 20 ether: 0 != 20000000000000000000]
```


## Impact
* Users will be unable to receive rewards.
* The contract owner will also be unable to withdraw ETH from the contract.

## Recommendations
Add processing for `userBalances` in the `likeUser` function:
```diff
function likeUser(
    address liked
) external payable {
    require(msg.value >= 1 ether, "Must send at least 1 ETH");
    require(!likes[msg.sender][liked], "Already liked");
    require(msg.sender != liked, "Cannot like yourself");
    require(profileNFT.profileToToken(msg.sender) != 0, "Must have a profile NFT");
    require(profileNFT.profileToToken(liked) != 0, "Liked user must have a profile NFT");

    likes[msg.sender][liked] = true;
+   userBalances[msg.sender] += msg.value;
    emit Liked(msg.sender, liked);

    [...]
}
```