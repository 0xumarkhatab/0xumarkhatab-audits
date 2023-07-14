<div>
<p align="center">
    <img src="https://pbs.twimg.com/profile_images/1647972457729409024/O9rTSRkD_400x400.jpg" width="150" height="150" />
</p>
<h1 align="center">Beam Audit Findings Sherlock</h1>
<h2 align="center">GameFi Focused blockchain</h2>
<p align="center">Umar Khatab - Independent Security Researcher</p>  
<p align="center">Date: July 10 to July 13, 2023</p>
<hr/>
</div>

## About Beam 
The beam is a sovereign network focused on gaming brought to you by  @MeritCircle_IO.

The beam is one of its kind Blockchain to focus on Gamification. 

It opens endless possibilities for web3 gamers to come to build or play and take part in this revolution.

## About Auditor

Umar Khatab is an aspiring security researcher

with a good amount of experience in blockchain development and now in security.

He has served the blockchain industry for 2.5 years as of now!

He took his inspiration from Koolex, zachObront, and several Secureum mentors.

Send him a message on [Twitter](https://twitter.com/0xumarkhatab) 

## Report

# [H1] Lack of Admin Role Allocation in MeritNFT.sol can cause the contract to get stuck

## Summary
The MeritNFT contract, responsible for managing the minting and URI setting of NFTs, suffers from a critical vulnerability due to the absence of admin role allocation. This vulnerability can lead to potential operational issues, including excessive minting, the inability to set URIs for legitimate NFTs, and the risk of setting incorrect URIs, thus compromising user trust and experience. Additionally, the lack of an admin role prevents the revocation of compromised addresses or the assignment of new roles, potentially rendering the contract unmanageable.

## Description
The MeritNFT contract employs an access control mechanism using the AccessControlEnumerable contract from the OpenZeppelin library. However, during contract deployment, there is no allocation of the admin role, making it impossible to grant or revoke roles. As a result, the contract remains vulnerable to unauthorized activities, including excessive minting by compromised addresses and incorrect URI settings for NFTs.

## Exploitation Scenario:
If the private keys of addresses holding the `MINTER_ROLE` and `URI_SETTER` roles are compromised, there is no way to revoke their roles or assign new roles. This lack of control can lead to severe consequences:
1. **Excessive Minting**: Compromised addresses can continue minting NFTs indefinitely, exceeding the predefined limits of the contract, such as the maximum cap per address or the total minting cap.
2. **Invalid URI Settings**: Without the ability to assign the `URI_SETTER` role to a trusted address, there is a risk of setting incorrect URIs for legitimate NFTs. This can result in incorrect or misleading information associated with the NFTs, eroding user trust and undermining the value of the tokens.

## Impact
The impact of this vulnerability includes:
- Excessive inflation of the NFT supply, potentially devaluing the tokens.
- Loss of trust from users due to incorrect or misleading URI settings, affecting the NFTs' perceived value and authenticity.
- Inability to manage and control the roles assigned to addresses, compromising the contract's operational stability and security.

## PoC 
To highlight the vulnerability, a foundry test was conducted. The following test case demonstrates the inability to grant the `MINTER_ROLE` to a potential minter due to the lack of an admin role allocation:

```solidity
function testFail_ContractIsNotOwned() external {
    vm.startPrank(owner);

    console.log("-----------------------");
    nft.grantRole(nft.MINTER_ROLE(), potentialMinter);
    assertEq(nft.hasRole(nft.MINTER_ROLE(), potentialMinter), false);

    /**
     * [FAIL. Reason: AccessControl: account 0x0000000000000000000000000000000ffffffff4 is missing role 0x0000000000000000000000000000000000000000000000000000000000000000] 
     */
}
```
![image](https://github.com/sherlock-audit/2023-07-beam-auction-0xumarkhatab/assets/71306738/28c508fa-6fd2-42fe-bf0f-06818d7521ba)

## Recommendation
To address this vulnerability, the following mitigation measures are recommended:

1. **Admin Role Allocation**: In the constructor of the MeritNFT contract, allocate the admin role (`DEFAULT_ADMIN_ROLE`) to a trusted address, typically the contract deployer. This role should have the authority to grant and revoke other roles.
2. **Role Management Functions**: Implement functions in the MeritNFT contract that allow the admin to grant or revoke roles to trusted addresses. These functions should be secured and accessible only to the admin role.

By allocating the admin role and implementing role management functions, you can regain control over role assignments and prevent unauthorized actions, ensuring the secure and intended operation of the MeritNFT contract.

**Severity:**
Based on the potential impact and operational risks associated with this vulnerability, I classify it as **High** severity.

I strongly recommend the immediate implementation of the suggested mitigation measures to safeguard the MeritNFT contract and protect the interests of its users.

**Code Example:**

```solidity
// MeritNFT.sol

// ...

constructor(
    string memory _name,
    string memory _symbol,
    string memory _baseTokenURI,
    address _uriSetter,
    address _minter
) ERC721(_name, _symbol) {
    baseTokenURI_ = _baseTokenURI;
    _setupRole(URI_SETTER, _uriSetter);
    _setupRole(MINTER_ROLE, _minter);
    _setupRole(DEFAULT_ADMIN_ROLE, msg.sender); // Allocate admin role to the contract deployer
}
```

```solidity
// MeritNFT.sol

// ...

function grantAdminRole(address _newAdmin) external onlyAdmin {
    grantRole(DEFAULT_ADMIN_ROLE, _newAdmin);
}

function revokeAdminRole(address _admin) external onlyAdmin {
    revokeRole(DEFAULT_ADMIN_ROLE, _admin);
}
```

By adding the `DEFAULT_ADMIN_ROLE` allocation in the constructor and implementing `grantAdminRole()` and `revokeAdminRole()` functions, you can manage the admin role, allowing for proper role assignment and revocation to ensure the secure operation of the MeritNFT contract.


## Tool used

Foundry, Manual Review

# [H2] Unlimited Minting of Merit NFTs

## Summary
The MeritNFT contract, which is responsible for minting NFTs, suffers from a critical vulnerability that allows unauthorized minting beyond the specified caps. While the AuctionMerit contract implements limits on the number of NFTs that can be minted per address and an overall minting cap, these limits are not enforced in the MeritNFT contract itself. This vulnerability enables an attacker with the `MINTER_ROLE` to bypass the caps and mint an unlimited number of tokens, leading to an inflationary supply and undermining the integrity and value of the protocol.

## Description
The MeritNFT contract defines a maximum number of NFTs that can be minted per address (CAP_PER_ADDRESS) and an overall cap on the total number of NFTs that can be minted (MINT_CAP). However, these caps are not enforced within the MeritNFT contract itself. As a result, an attacker with the `MINTER_ROLE` can directly call the `mint()` function of the MeritNFT contract, bypassing the caps imposed by the AuctionMerit contract.

## Exploitation Scenario
If an attacker obtains the `MINTER_ROLE` and identifies the MeritNFT contract address used by the AuctionMerit contract, they can execute the following steps to exploit the vulnerability:
1. Obtain the MeritNFT contract address from the AuctionMerit contract.
2. Call the `mint()` function of the MeritNFT contract repeatedly, minting NFTs beyond the defined caps.
3. By bypassing the caps, the attacker can inflate the token supply indefinitely, undermining the token's scarcity and diminishing its value.

## Impact
The impact of this vulnerability includes:
- **Inflationary Supply**: Unauthorized minting beyond the defined caps leads to an excessive supply of NFTs, resulting in reduced scarcity and diminishing the value of the tokens.
- **Loss of Trust**: Users may lose trust in the protocol if the minting caps are not upheld, as it undermines the fairness and integrity of the system.
- **Economic Disruption**: The unauthorized minting disrupts the economic balance of the ecosystem, potentially impacting the market value of the tokens and the overall health of the platform.

## Tool used

Foundry, Manual Review


## PoC
To demonstrate the vulnerability, a foundry test was conducted. The following test case showcases the ability of an attacker to mint more NFTs than the specified cap:

```solidity
function test_MintMoreThanCap() public {
    vm.startPrank(minter);
    console.log("-----------------------");
    uint bal = nft.balanceOf(attacker);
    console.log("Attacker balance before: ", bal);

    for (uint i = 1; i <= 20000; i++) {
        nft.mint(i, attacker);
    }
    bal = nft.balanceOf(attacker);
    console.log("Attacker balance After: ", bal);
    vm.stopPrank();
}
```
![image](https://github.com/sherlock-audit/2023-07-beam-auction-0xumarkhatab/assets/71306738/34374411-d44f-49da-9b39-69d12a897462)


The test case shows that an attacker with the `MINTER_ROLE` can mint 20000 NFTs, surpassing the specified cap, and inflate their balance without any enforcement.

## Mitigation
To address this vulnerability, the following mitigation measures are recommended:

1. **Cap Enforcement**: Modify the `mint()` function in the MeritNFT contract to include checks for the minting caps. Before minting an NFT, validate whether the total minted count and the count per address are within the defined limits. If the limits are exceeded, revert the transaction and prevent unauthorized unlimited minting.

By implementing cap enforcement within the MeritNFT contract, you can ensure that minting is restricted to the defined caps, preserving the scarcity and value of the NFTs.

## Severity
Based on the potential impact and the disruption it can cause to the token's value and trust in the protocol, I classify this vulnerability as **High** severity.

It is highly recommended to implement the suggested mitigation measures promptly to prevent unauthorized minting and maintain the integrity of the MeritNFT contract.

Thank you for reading !

# [H3] Single point of Failure of Money , Trust and Reputation due to privatekeys compromise as Merlin Dex hack of 1.82M$

## Motivation: Recent Example: Merlin DEX Exploit

An incident involving the compromise of private keys and its impact on a decentralized exchange platform serves as a cautionary example. Merlin DEX experienced a security breach resulting in the loss of $1.82 million due to a compromised private key. The incident highlighted the importance of considering centralized risks and thoroughly auditing key aspects of a protocol.

For more information, refer to the Twitter post by EvoKid: [Merlin DEX Hack - EvoKid's Twitter Post](https://twitter.com/CryptosUni/status/1672958215875710976?s=20)

### Summary

The AuctionMerit contract is vulnerable to the compromise of private keys associated with key roles, namely the Owner, Minter, and UriSetter. If any of these private keys are compromised, it can lead to severe consequences and compromise the overall security and integrity of the contract.

## Attack scenarios
The following issues can arise due to the private keys Compromise:

#### Owner Key Compromise:
If the private key of the Owner role is compromised, an attacker would gain complete control over the contract. They could potentially drain the funds held in the contract or make unauthorized changes to critical contract parameters.

#### Minter Key Compromise:
In the current implementation, the Minter role allows the minting of NFTs without any limit. If the private key of the Minter role is compromised, an attacker can mint an unlimited number of NFTs, disregarding the predefined minting limits. This would violate the overall accounting and value of the tokens on the platform, potentially leading to a devaluation of the NFTs and defaming the protocol.

#### UriSetter Key Compromise:
The UriSetter role is responsible for setting the tokenURI for each NFT. If the private key of the UriSetter role is compromised, an attacker could manipulate the tokenURI of NFTs, potentially setting them to empty strings (""). This would result in incorrect or missing metadata for the NFTs, negatively impacting the user experience and eroding trust in the platform.


### Impact

The compromise of any of these key roles can have severe consequences:

- Loss of Funds: Compromising the Owner key could result in the unauthorized transfer of funds held in the contract, leading to financial losses for the platform and its users.

- Unlimited Minting: If the Minter key is compromised, an attacker can mint an unlimited number of NFTs, leading to an overabundance of tokens and potentially devaluing the NFTs minted on the platform.

- Incorrect or Missing Metadata: A compromise of the UriSetter key could allow an attacker to manipulate the tokenURI of NFTs, leading to incorrect or missing metadata. This can undermine the trust of users who rely on accurate metadata for making informed decisions.


### Recommendation

To address the vulnerability of private key compromise, i would recommend Adopting a multisignature (multisig) mechanism for critical actions such as fund transfers or changes to key contract parameters. This requires multiple authorized parties to sign off on transactions, reducing the risk of a single compromised key compromising the entire contract.


By implementing these recommendations, particularly the adoption of multisig control and secure key management practices, the AuctionMerit contract can significantly reduce the risk of compromised key roles and protect the funds, minting process, and metadata integrity.

Please note that this vulnerability is classified as **High** severity due to the potential financial losses, devaluation of tokens, and erosion of user trust that can result from a compromise of private keys associated with key roles.

# [M1] A Clever Miner can have an unfair advantage and cause Financial loss to the platform through block timestamp manipulation

## Summary
The Auction Merit contract is vulnerable to block timestamp manipulation,<br/> which can be exploited to gain an unfair advantage and undermine the integrity of the auction process.<br/> The vulnerability allows the minter to manipulate the block timestamp to perform actions that deviate <br/> from the intended behavior of the contract.

## Scenarios
**Early/Late Minting**: <br/> The minter can manipulate the block timestamp to mint NFTs <br/> before the actual auction start time or after the auction time.<br/> By setting the block timestamp to a time in the future, <br/> the minter can bypass the auction restrictions and mint NFTs prematurely,<br/> giving them an unfair advantage over other participants.

**Undervalued Payment**:<br/> The minter can manipulate the block timestamp to pay for NFTs at the initial start price, <br/> even if the actual time is close to the end price. <br/> By setting the block timestamp to an earlier time, the minter can exploit the lower start price,<br/> resulting in an undervalued payment for the NFTs they acquire.

## Impact

The manipulation of the block timestamp in the AuctionMerit contract can have the following consequences:

**Unfair Advantage**: <br/> The minter can gain an unfair advantage over other participants by minting NFTs before the auction starts or paying at a lower price,<br/> distorting the fairness of the auction process.

**Financial Loss**: <br/> Legitimate participants may suffer financial losses due to undervalued payments made by the minter,<br/> as the NFTs are acquired at a lower price than intended.

## Code

```solidity
/// @notice Returns the current price
    function getPrice() public view returns(uint256) {
        // Use the current timestamp to get the current price
->        return getPriceAt(block.timestamp);
    }

    /// @notice Returns the price at a given time
    /// @param time Timestamp
    function getPriceAt(uint256 time) public view returns(uint256) {
        // If auction hasn't started yet, return start price
        if(time < startTime) {
            return startPrice;
        }
        // If auction has ended, return end price
        if(time >= endTime) {
            return endPrice;
        }

...
    }
```

## Tool used
Manual Review

## Recommendation
Try to integrate an external source of time to make the price decisions on.




