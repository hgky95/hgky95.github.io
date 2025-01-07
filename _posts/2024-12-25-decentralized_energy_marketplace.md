---
title: 'Energy NFT Marketplace'
categories: [blockchain]
date: 2025-01-06 21:00:00
tags: [blockchain, solidity, backend, java, spring-boot]
image: "/assets/img/energy_nft_marketplace/poster.png"
---
Energy trading is being transformed by blockchain technology. This article delves into building a decentralized Energy NFT Marketplace on [Base Sepolia](https://www.base.org/), enabling users to tokenize energy production, trade energy NFTs, and earn loyalty points through an integrated rewards program. <br>
The platform also supports seamless asset bridging between L2s and L1 networks that is powered by [Across Protocol](https://across.to/), showcasing the potential of blockchain to revolutionize energy markets.

> The complete source code and installation guide are available on [GitHub](https://github.com/hgky95/energy-marketplace).
{: .prompt-tip }

## Architecture Overview

The project includes three-tier architecture:

![Architecture Diagram](https://github.com/user-attachments/assets/be1520e2-6d44-4546-9d44-e4a1aff797c4)

1. **Frontend Layer**: TypeScript, React application with Web3 integration
2. **Backend Layer**: Spring Boot service with caching and event handling
3. **Blockchain Layer**: Smart contracts deployed on EVM-compatible blockchain

> The SmartGrid in this architecture is for simulation purpose only.

## Key Features

- Energy NFT minting and trading
- Cross-chain bridge
- Real-time blockchain event synchronization
- Loyalty program for user engagement
- Efficient caching system

## Demonstration

Here is the demo video: [Youtube](https://youtu.be/dtCQmJSXIFc)

## Implementation

### Smart Contracts

The blockchain layer consists of three main contracts:
1. EnergyNFT.sol
2. EnergyMarketplace.sol
3. LoyaltyProgram.sol

All of them are solidity code but using javascript markdown for better readability.

#### EnergyNFT.sol

The `EnergyNFT` smart contract enables tokenization of energy as NFTs, allowing users to mint, trade, and transfer energy securely while maintaining individual energy balances

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract EnergyNFT is ERC721URIStorage, Ownable {
    uint256 private _tokenIds;
    mapping(address => uint256) public userEnergyBalances;
    mapping(uint256 => uint256) public tokenEnergyAmount;
    address public marketplaceAddress;

    event EnergyNFTMinted(uint256 tokenId, address owner, string ipfsHash);
    event EnergyBalanceUpdated(address user, uint256 newBalance);
    event EnergyProduced(address user, uint256 amount);
    event MarketplaceAddressUpdated(address newMarketplace);

    constructor() ERC721("Energy NFT", "ENFT") Ownable(msg.sender) {}

    modifier onlyMarketplace() {
        require(
            msg.sender == marketplaceAddress,
            "Only marketplace can call this function"
        );
        _;
    }

    function setMarketplaceAddress(
        address _marketplaceAddress
    ) external onlyOwner {
        require(
            _marketplaceAddress != address(0),
            "Invalid marketplace address"
        );
        marketplaceAddress = _marketplaceAddress;
        emit MarketplaceAddressUpdated(_marketplaceAddress);
    }

    //Note: this method should be interact with trusted oracle, currently it's just for demo purpose
    function produceEnergy(
        address user,
        uint256 _energyAmount
    ) external onlyOwner {
        userEnergyBalances[user] += _energyAmount;
        emit EnergyProduced(user, _energyAmount);
        emit EnergyBalanceUpdated(user, userEnergyBalances[user]);
    }

    function mint(
        address _from,
        string memory _tokenURI,
        uint256 _energyAmount
    ) external returns (uint) {
        require(_energyAmount > 0, "Energy value should be greater than 0");
        require(
            userEnergyBalances[_from] >= _energyAmount,
            "Insufficient energy balance!!!"
        );

        _tokenIds++;
        _safeMint(_from, _tokenIds);
        _setTokenURI(_tokenIds, _tokenURI);

        tokenEnergyAmount[_tokenIds] = _energyAmount;
        userEnergyBalances[_from] -= _energyAmount;

        emit EnergyNFTMinted(_tokenIds, _from, _tokenURI);
        emit EnergyBalanceUpdated(_from, userEnergyBalances[_from]);

        return (_tokenIds);
    }

    function transferEnergy(
        address _from,
        address _to,
        uint256 _tokenId
    ) external onlyMarketplace {
        require(
            ownerOf(_tokenId) == _to,
            "Energy can only be transferred to NFT owner"
        );

        uint256 energyAmount = tokenEnergyAmount[_tokenId];
        require(energyAmount > 0, "No energy associated with this NFT");

        userEnergyBalances[_to] += energyAmount;
        tokenEnergyAmount[_tokenId] = 0;

        emit EnergyBalanceUpdated(_from, userEnergyBalances[_from]);
        emit EnergyBalanceUpdated(_to, userEnergyBalances[_to]);
    }

    function getCurrentEnergy(address user) public view returns (uint256) {
        return userEnergyBalances[user];
    }
}
```

#### EnergyMarketplace.sol

The `EnergyMarketplace` smart contract facilitates the trading of energy NFTs, managing listings, purchases. It includes a commission system that adjusts based on user loyalty points.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import {EnergyNFT} from "./EnergyNFT.sol";
import {ILoyaltyProgram} from "./ILoyaltyProgram.sol";

contract EnergyMarketplace is ReentrancyGuard, Ownable {
    uint8 public constant DEFAULT_BASE_COMMISSION_RATE = 1;
    uint256 private constant PRECISION = 100;

    EnergyNFT public nftContract;
    ILoyaltyProgram public loyaltyProgram;

    uint256 public itemCount;
    uint8 public baseCommissionRate;

    struct Item {
        uint256 tokenId;
        uint256 price;
        uint256 energyAmount;
        address seller;
        bool isActive;
    }

    mapping(uint256 => Item) public items;

    event NFTMintedAndListed(
        uint256 tokenId,
        address seller,
        string ipfsHash,
        uint256 energyValue,
        uint256 price
    );
    event NFTSold(
        uint256 tokenId,
        address seller,
        address buyer,
        uint256 price,
        uint256 fee
    );
    event ListingUpdated(uint256 tokenId, uint256 newPrice);
    event ListingCancelled(uint256 tokenId);
    event CommissionRateUpdated(uint256 newFeePercentage);
    event Withdrawal(address recipient, uint256 amount);
    event LoyaltyProgramUpdated(address newLoyaltyProgram);

    constructor(
        address _nftContract,
        address _loyaltyProgram
    ) Ownable(msg.sender) {
        nftContract = EnergyNFT(_nftContract);
        loyaltyProgram = ILoyaltyProgram(_loyaltyProgram);
        baseCommissionRate = DEFAULT_BASE_COMMISSION_RATE;
    }

    function setLoyaltyProgram(address _loyaltyProgram) external onlyOwner {
        require(_loyaltyProgram != address(0), "Invalid address");
        loyaltyProgram = ILoyaltyProgram(_loyaltyProgram);
        emit LoyaltyProgramUpdated(_loyaltyProgram);
    }

    function calculateFee(
        uint256 price,
        address seller
    ) public view returns (uint256) {
        uint256 sellerPoints = loyaltyProgram.getLoyaltyPoints(seller);
        uint256 commissionRate = loyaltyProgram.getCommissionRate(
            sellerPoints,
            baseCommissionRate
        );
        return (price * commissionRate) / (100 * PRECISION);
    }

    function buyNFT(uint256 _tokenId) external payable nonReentrant {
        Item storage item = items[_tokenId];
        require(item.isActive, "NFT not for sale");
        require(msg.value >= item.price, "Insufficient payment");

        address seller = item.seller;
        uint256 price = item.price;
        uint256 fee = calculateFee(price, seller);
        uint256 sellerProceeds = price - fee;

        // Mark item as inactive before making transfers
        item.isActive = false;

        nftContract.transferFrom(seller, msg.sender, _tokenId);
        nftContract.transferEnergy(seller, msg.sender, _tokenId);

        // Update loyalty points based on energy amount
        loyaltyProgram.addLoyaltyPoints(seller, uint32(item.energyAmount / 10));

        payable(seller).transfer(sellerProceeds);

        emit NFTSold(_tokenId, seller, msg.sender, price, fee);
    }

    function updateCommissionRate(uint8 _newCommissionRate) external onlyOwner {
        require(
            _newCommissionRate > 0,
            "Commission rate cannot be less than 0"
        );
        baseCommissionRate = _newCommissionRate;
        emit CommissionRateUpdated(_newCommissionRate);
    }

    function mintAndList(
        string memory _tokenURI,
        uint256 _energyAmount,
        uint256 _price
    ) external returns (uint256) {
        uint256 newTokenId = nftContract.mint(
            msg.sender,
            _tokenURI,
            _energyAmount
        );
        items[newTokenId] = Item(
            newTokenId,
            _price,
            _energyAmount,
            msg.sender,
            true
        );
        emit NFTMintedAndListed(
            newTokenId,
            msg.sender,
            _tokenURI,
            _energyAmount,
            _price
        );
        itemCount++;
        return newTokenId;
    }

    function withdrawFees(uint256 _amount) external onlyOwner {
        uint256 balance = address(this).balance;
        require(balance >= _amount, "Insufficient balance");

        (bool success, ) = payable(owner()).call{value: _amount}("");
        require(success, "Failed to withdraw fees");
        emit Withdrawal(owner(), _amount);
    }
}
```

#### LoyaltyProgram.sol

The `LoyaltyProgram` smart contract manages user loyalty points and commission rates. It includes a discount tier system that adjusts based on the number of loyalty points.

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import "@openzeppelin/contracts/access/Ownable.sol";
import "./ILoyaltyProgram.sol";

contract LoyaltyProgram is ILoyaltyProgram, Ownable {
    // use higher precision (2 decimal places in this case) - fixed-point arithmetic
    uint256 private constant PRECISION = 100;

    struct Discount {
        uint256 points;
        uint8 discountPercentage;
    }

    Discount[] public discountTiers;
    mapping(address => uint256) private userPoints;
    mapping(address => bool) public authorizedCallers;

    event DiscountTierAdded(uint256 points, uint8 discountPercentage);
    event DiscountTierUpdated(
        uint256 index,
        uint256 points,
        uint8 discountPercentage
    );
    event DiscountTierRemoved(uint256 index);
    event LoyaltyPointsAdded(address user, uint256 points);
    event CallerAuthorized(address caller);
    event CallerRevoked(address caller);

    modifier onlyAuthorized() {
        require(
            authorizedCallers[msg.sender] || msg.sender == owner(),
            "Not authorized"
        );
        _;
    }

    constructor() Ownable(msg.sender) {
        discountTiers.push(Discount(1000, 5));
        discountTiers.push(Discount(5000, 8));
        discountTiers.push(Discount(10000, 10));
    }

    function addAuthorizeCaller(address caller) external onlyOwner {
        authorizedCallers[caller] = true;
        emit CallerAuthorized(caller);
    }

    function revokeCaller(address caller) external onlyOwner {
        authorizedCallers[caller] = false;
        emit CallerRevoked(caller);
    }

    function getCommissionRate(
        uint256 loyaltyPoints,
        uint8 baseCommissionRate
    ) external view override returns (uint256) {
        uint256 discountPercentage = 0;

        for (uint256 i = 0; i < discountTiers.length; i++) {
            if (loyaltyPoints >= discountTiers[i].points) {
                discountPercentage = discountTiers[i].discountPercentage;
            } else {
                break;
            }
        }

        return
            (baseCommissionRate * PRECISION * (100 - discountPercentage)) / 100;
    }

    function addLoyaltyPoints(
        address user,
        uint32 points
    ) external override onlyAuthorized {
        userPoints[user] += points;
        emit LoyaltyPointsAdded(user, userPoints[user]);
    }

    function getLoyaltyPoints(
        address user
    ) external view override returns (uint256) {
        return userPoints[user];
    }

    function updateDiscountTier(
        uint256 index,
        uint256 points,
        uint8 discountPercentage
    ) external override onlyOwner {
        require(index < discountTiers.length, "Invalid tier index");
        require(discountPercentage <= 100, "Discount cannot exceed 100%");
        discountTiers[index] = Discount(points, discountPercentage);
        emit DiscountTierUpdated(index, points, discountPercentage);
    }

    function addDiscountTier(
        uint256 points,
        uint8 discountPercentage
    ) external override onlyOwner {
        require(discountPercentage <= 100, "Discount cannot exceed 100%");
        discountTiers.push(Discount(points, discountPercentage));
        emit DiscountTierAdded(points, discountPercentage);
    }

    function removeDiscountTier(uint256 index) external override onlyOwner {
        require(index < discountTiers.length, "Invalid tier index");
        for (uint256 i = index; i < discountTiers.length - 1; i++) {
            discountTiers[i] = discountTiers[i + 1];
        }
        discountTiers.pop();
        emit DiscountTierRemoved(index);
    }
}
```

## Conclusion

This project demonstrates how blockchain can revolutionize energy trading by enabling secure tokenization, decentralized trading, and rewarding user participation. With features like dynamic commissions, loyalty rewards, and seamless energy transfers, it paves the way for scalable and transparent energy marketplaces, showcasing the power of blockchain in real-world applications.
