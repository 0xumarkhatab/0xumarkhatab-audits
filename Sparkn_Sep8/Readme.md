# Sparkn  - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Low Risk Findings
    - ### [L-01. Dos Attack to setContest by a miner](#L-01)
    - ### [L-02. Proxy Implementation address can be overwritten by Implementation itself in future](#L-02)
    - ### [L-03. Invariant failure : Proxy's _distribute Function is susceptible to donation attack with Rounding problem](#L-03)
    - ### [L-04. Signature Replay in Proxy Factory](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 21st, 2023 - Aug 29th, 2023

[See more contest details here](https://www.codehawks.com/contests/cllcnja1h0001lc08z7w0orxx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - Low: 4



# Low Risk Findings

## <a id='L-01'></a>L-01. Dos Attack to setContest by a miner            



## Summary
SetContest function can be front-runned and denied to provide service due to block.timestamp manipulations by miners.
## Vulnerability Details
Although it does not give much incentive to the miners there might be the cases in future where some miners 
can prevent the execution of the `setContest` method for their benefit.

The function uses block.timestamp which can be manipulated by the miners.
They can either set it to very low or very high value.
This will make the following condition get True and execute revert.

`ProxyFactory#setContest`

```solidity
        if (closeTime > block.timestamp + MAX_CONTEST_PERIOD || closeTime < block.timestamp) {
            revert ProxyFactory__CloseTimeNotInRange();
        }

```

## Impact

- The owner will be unable to set the details of any contest causing a denial of the service of the platform

## Tools Used

Manual review

## Recommendations
Keep in mind that block.timestamp can be manipulated and craft the closeTime condition accordingly.

One thing we can do is cache the last timestamp value of successful deployment.

And compare the last one with the close Time.


If the last value is greater than that of the current blocktimestamp , we surely know that it has been manipulated and we just allow the contest to be created.

For the case when block.timestamp is manipulated to a really high value like months, we can set a threshold like 5 days, if the last block.timestamp was 5 days ago or even 10( set this value based upon the perceived usage or maybe make a setter function to set its price later according to usage history)

and then allow contest creation if the current timestamp is far more than estimated.
## <a id='L-02'></a>L-02. Proxy Implementation address can be overwritten by Implementation itself in future            



## Vulnerability Details
Looking at the mechanism of storing the implementation address in a proxy contract,
We can infer two things:

- It is a non-standard method
- vulnerable to being overwritten by the logic( implementation ) contract.

Although the current implementation of the `Distributor` contract uses immutables and constants to store additional data inside the logic contract, in the future if there is a need to add some non-constant data inside the logic contract, this might cause an issue

```solidity
address private immutable _implementation;`
```


## Impact
The implementation address can be overwritten by the logic contract inside the proxy's storage and the contest will be lost along with its associated tokens which will incur a loss to users and protocol.

## Tools Used
Manual review
## Recommendations
We should use a standard method of storing the implementation address at a storage slot that is very random and there are negligible chances of it being overwritten.

here is the standard calculation by Openzeppelin:

```solidity

bytes32 private constant implementationPosition = bytes32(uint256(
  keccak256('eip1967.proxy.implementation')) - 1
));

```

## <a id='L-03'></a>L-03. Invariant failure : Proxy's _distribute Function is susceptible to donation attack with Rounding problem            



## Summary

An attacker can donate a certain amount of tokens to the Proxy contract and make the winners fewer tokens than intended.

## Vulnerability Details
Right now, the `_distribute` function checks the latest balance of the tokens of the proxy contract.
which can be increased by donating some tokens to the contract.

```solidity


    function _distribute(address token, address[] memory winners, uint256[] memory percentages, bytes memory data)
        internal
    {
        ..
        ..
        uint256 totalAmount = erc20.balanceOf(address(this));
        // if there is no token to distribute, then revert
        if (totalAmount == 0) revert Distributor__NoTokenToDistribute();

        uint256 winnersLength = winners.length; // cache length
        for (uint256 i; i < winnersLength;) {
            uint256 amount = totalAmount * percentages[i] / BASIS_POINTS;
            erc20.safeTransfer(winners[i], amount);
            unchecked {
                ++i;
            }
        }


```

The thing is when the contract is faced with multiple scenarios of accounting of different variables, the contract does not work as expected.

The contract suffers from `rounding errors` where the invariant fails.

percentageShares of winners + CommissionFee == 10,000

# Proof Of Concept

Here are the things to note about the PoC :

- As the attacker can easily manipulate the token balance of the contract, the fuzzing will include this as a parameter.

- For simplicity, i've just included two winners and their percentages ( fuzz variables )

- I kept the commission Fee constant for now which is 500

- I've just shown two cases here but more can be generated.

## PoC code

```solidity

function testFuzzRounding(uint totalAmount,uint a)external {
    vm.assume(totalAmount>1e18); // any amount of tokens
    vm.assume(totalAmount<10000000000*1e18); // any amount of tokens
    vm.assume(a<9000); // just 10% less than 100%
    vm.assume(a>1000); // at least one percent share for reality check
    // 500 is the commision fee
    uint b=10000-a-500;
    uint256 amount1 = totalAmount * a / 10000;
    uint256 amount2 = totalAmount * b / 10000;
    assertEq(amount1+amount2,totalAmount);


}


```

## PoC results / breaking cases

Upon following values, the contract's invariant fails. 

| TotalTokensAmount| percentageOfWinner1 | percentageOfWinner2 | CommissionFee |
| --- | ---- | --- | ---- |
|400000000000000000000000 | 1098| 8402  | 500 |
|282776882385583792624126 |1506 | 7994 | 500 |

These inputs are verified ✔

## Impact
- Loss of funds for users.
- It is not very hard to exploit so the severity is high

## Tools Used
Foundry, Manual review

## Recommendations
Instead of checking directly the latest or instantaneous balance of the contract, maintain a state variable of `tokenBalance` of tokens initialized to a certain value when the contract is created.

Use `tokenBalance` instead of `totalAmount` in the `_distribute` function.

```solidity


    function _distribute(address token, address[] memory winners, uint256[] memory percentages, bytes memory data)
        internal
    {
      ..
      ..

        // if there is no token to distribute, then revert
        if (tokenBalance== 0) revert Distributor__NoTokenToDistribute();

        uint256 winnersLength = winners.length; // cache length
        for (uint256 i; i < winnersLength;) {
            uint256 amount = tokenBalance* percentages[i] / BASIS_POINTS;
            erc20.safeTransfer(winners[i], amount);
            unchecked {
                ++i;
            }
        }


```



## <a id='L-04'></a>L-04. Signature Replay in Proxy Factory            



## Summary
The protocol is susceptible to signature replay attacks due to the lack of fields on which the message is signed.

## Vulnerability Details

The protocol does not use any `signature timestamp` (after which the signature is not useable) or a nonce ( a number unique to every user) inside `ProxyFactory#deployProxyAndDistributeBySignature`.

In fact, it uses the following method that can for sure prevent the cross-chain signature replay but not across one chain.

```solidity

        bytes32 domainSeparatorV4 = keccak256(
            abi.encode(
                keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
                keccak256(bytes("ProxyFactory")),
                keccak256(bytes("1")),
                block.chainid,
                address(proxyFactory)
            )
        );
        bytes32 randomId_ = keccak256(abi.encode("Jason", "001"));
        bytes memory sendingData = createData();
        bytes32 data = keccak256(abi.encode(randomId_, sendingData));

```

## Impact

Invalid signatures might be accepted in `deployProxyAndDistributeBySignature` and cause invalid proxy contracts to be deployed by the same person even if they intended to deploy only once.

## Tools Used

Manual review

## Recommendations

There are two possible solutions:

- Implement nonce based signing

```solidity

// declare a user->nonce mapping and check the latest nonce on each deployment from the user
mapping(address=>uint) nonces;
.
.
.

function deployProxyAndDistributeBySignature(
        address organizer,
        bytes32 contestId,
        address implementation,
        bytes calldata signature,
        uint nonce,
        bytes calldata data
    ) public returns (address) {

require(nonce==nonces(msg.sender),"nonces does not match").

..
..
..

}

```

- Implement signatures caching so that multiple signatures can't be replayed.
```solidity

// declare a user->nonce mapping and check the latest nonce on each deployment from the user
mapping(address=>bool) signatures;
.
.

function deployProxyAndDistributeBySignature(
        address organizer,
        bytes32 contestId,
        address implementation,
        bytes calldata signature,
        uint nonce,
        bytes calldata data
    ) public returns (address) {
..
require(signatures[signature]==false,"nonces does not match");
signatures[signature]=true;
..
..
..

}

```



