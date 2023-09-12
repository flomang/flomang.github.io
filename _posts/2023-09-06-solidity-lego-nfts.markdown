---
layout: post
title:  "Exploring a Lego NFT marketplace"
date:   2023-09-06 2:53:00 -0700
categories: jekyll update
---

I'm an AFOL (Adult Fan of Lego) with a modest Lego collection that I've decided to make available for purchase in exchange for cryptocurrency. To bring this idea to life, I've created a smart contract for a Lego NFT marketplace. Users can seamlessly engage with this smart contract through their MetaMask wallets, enabling them to buy and sell Lego sets using ETH. It's important to emphasize that this contract's primary function is to facilitate an NFT marketplace where each Lego item is uniquely represented as a non-fungible token, and ownership is tracked securely by Ethereum addresses.

Nevertheless, it's crucial to emphasize that while this smart contract serves as the foundation of the Lego marketplace, several vital functions essential for the real-world operation of such a marketplace, including logistics, shipping, escrow services, and dispute resolution, lie beyond its primary purview.

Therefore, this smart contract will only include the following simple features:

1. **Listing Lego Lots for Sale:** Sellers can easily list their Lego sets for sale, providing detailed information about each item.

1. **Purchasing Lego Lots:** Buyers can acquire Lego sets listed on the marketplace, completing transactions using Ethereum.

1. **Reselling/Updating Lego Lots:** The ownable asset can be resold. Owners can relist their token, offering them to potential buyers with updated prices.

1. **Canceling Lot Listings:** Sellers have the option to remove their Lego sets from the marketplace if they decide not to sell them.

1. **Viewing All Lots for Sale:** Users can access a comprehensive list of all Lego sets currently available for purchase.

1. **Discovering Owned Lots:** Lot owners can conveniently locate and manage all the Lego sets they own within the marketplace.


### Contract Overview
I've named this contract the KisxPortal. I'm kissing my Legos goodbye so I felt this would make a suitable project name for now. The KisxPortal contract is an Ethereum smart contract that inherits from the openzepplin library, specifically ERC721URIStorage and Ownable. 

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.21;

import {ERC721URIStorage, ERC721} from "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Counters} from "@openzeppelin/contracts/utils/Counters.sol";

contract KisxPortal is ERC721URIStorage, Ownable {
    ...
     constructor(
        string memory _name,
        string memory _symbol,
        uint256 _initialMintPrice
    ) ERC721(_name, _symbol) {
        mintPrice = _initialMintPrice;
    }
}
```

ERC721URIStorageility gives us the ability to set a token URI for our tokens. We'll be using this in the future to store our token metadata. Ownable will be used to add access control for a contract withdrawBalance function as well as provide admin control to cancel a forsale listing owned by another user. 

#### Important Enums and Structs
```solidity
enum LotStatus { OnSale, Sold, OffMarket }
enum LotType { Set, Minifig, Part, MOC, Box, Instruction, Other }

struct Lot {
    uint256 id;
    string title;
    string description;
    uint256 price;
    string date;
    address payable owner;
    LotStatus status;
    LotType lotType;
    string uri;
}

// define a struct that will be used to log the transaction
struct LotTxn {
    uint256 lotId;
    uint256 price;
    address seller;
    address buyer;
    uint txnDate;
    LotStatus status;
}

// emit lot sold event 
event LotSold(
    uint _tokenId,
    string _title,
    uint256 _price,
    address _current_owner,
    address _buyer
);

// emit lot created event
event LotCreate(
    uint _tokenId,
    string _title,
    string _description,
    uint256 _price,
    address _author
);

// emit lot cancelled event
event LotCancel(uint _tokenId);

// emit lot resell event
event LotResell(uint _tokenId, uint _status, uint256 _price);

// emit lot updated event
event LotUpdated(Lot _updated);

// emit unauthorized withdraw failure error 
error UnauthorizedWithdrawFailure();
```
The purpose of the events listed above should be self-evident, as they serve as notifications generated during the contract's regular operations. These events carry valuable information related to transaction details.

* **LotTxn** is a structured data type designed to record and retain information pertaining to a sales transaction between a buyer and a seller. 

* **LotStatus** will define possible states a Lot can be in. For example, OnSale should indicate to the public that the item is for sale. 

* **LotType** shall describe a lot category. This category will describe what the lot item is: a part, set, etc.

* **Lot** is a struct that is used to store the relevant details pertaining to what is being sold/owned. A Lego Lot should have an ID (i.e same as token ID), title, description, price (denominated in ETH), date time as a UTC string, a payable owner address, status of lot item (OnSale, OffMarket, etc), type of lot, and uri that points to some json meta data containing the image of our lot - to be implemented.

    For example:
    Lot(1, "Set 1234", "Some awesome set", 0.0035, "2023-09-05T22:27:54Z", ...)

#### Counters and Variables
To efficiently manage essential aspects of the contract, I've chosen to integrate OpenZeppelin's Counter utility. Here's a breakdown of the key variables:

```solidity
Counters.Counter public pendingLotCount; // Keeps track of pending lots (items on sale).
Counters.Counter public index; // Manages the total number of NFTs created by the contract. 
uint256 public mintPrice; // Stores the cost associated with minting an NFT.

mapping(uint256 => LotTxn[]) private lotTxns; // A private mapping that records lot transactions. 
Lot[] public lots; // Keeps track of all NFTs
```
* **pendingLotCount** and **index** are Counter instances that meticulously monitor pending lots (items available for sale) and the generated token IDs, respectively. 
* **mintPrice** holds the price required for minting an NFT. 
* **lotTxns** is a private mapping dedicated to capturing sales transactions between a buyer and a seller. 
* **lots** is an array that serves as the repository for all NFTs minted by our contract, ensuring a comprehensive record of every token generated. 

### Smart Contract Functions
#### Minting Lego NFTs / Listing Lots for sale 
```solidity
function createLot(
    string memory _title,
    string memory _description,
    string memory _date,
    uint256 _price,
    string memory _uri,
    LotType _lotType
) public payable validSender returns (uint256 tokenId) {
    require(bytes(_title).length > 0, "The title cannot be empty");
    require(bytes(_date).length > 0, "The Date cannot be empty");
    require(
        bytes(_description).length > 0,
        "The description cannot be empty"
    );
    require(_price > 0, "The price cannot be empty");
    require(msg.value >= mintPrice, "The mint price is not paid");

    tokenId = index.current();
    Lot memory _lot = Lot({
        id: tokenId,
        title: _title,
        description: _description,
        price: _price,
        date: _date,
        owner: payable(msg.sender),
        status: LotStatus.OnSale,
        lotType: _lotType,
        uri: _uri
    });
    lots.push(_lot);
    _safeMint(msg.sender, tokenId);
    _setTokenURI(tokenId, _uri);

    emit LotCreate(tokenId, _title, _description, _price, msg.sender);

    // increment our token index counter
    index.increment();
    pendingLotCount.increment();

    // refund extra payment
    if (msg.value > mintPrice)
        payable(msg.sender).transfer(msg.value - mintPrice);
}
  
```
Users can list their Lego sets for sale by utilizing the createLot function, which is fairly self-explanatory in its operation. However, the critical implementation details lie in the outcomes of the _safeMint and _setTokenURI calls. These functions are inherited from our ERC721URIStorage contract and are responsible for minting our NFTs. You can find the complete definition of the internal functions provided by this extension [here](https://docs.openzeppelin.com/contracts/4.x/api/token/erc721#ERC721URIStorage)

Upon a successful NFT minting, the createLot function emits a LotCreate event, and any excess funds sent by the user are promptly returned to their wallet address.


#### Buying Lego Lots
```solidity
function buyLot(uint256 _tokenId) public payable validSender {
      Lot memory lot = findLot(_tokenId);
      // lot owner must not be zero
      require(lot.owner != address(0), "The owner cannot be zero");
      // buyer must not be the owner
      require(msg.sender != lot.owner, "You are the owner");
      // buyer must pay the price
      require(msg.value >= lot.price, "The price is not paid");
      // lot must be on sale
      require(lot.status == LotStatus.OnSale, "The lot is not for sale");

      // transfer ownership of tokenId from lot owner to sender 
      _safeTransfer(lot.owner, msg.sender, _tokenId, "");
      //make a payment to the current owner
      lot.owner.transfer(lot.price);

      // log the transaction
      LotTxn memory _lotTxn = LotTxn({
          lotId: lot.id,
          price: lot.price,
          seller: lot.owner,
          buyer: msg.sender,
          txnDate: block.timestamp,
          status: LotStatus.Sold
      });
      lotTxns[lot.id].push(_lotTxn);

      pendingLotCount.decrement();
      emit LotSold(lot.id, lot.title, lot.price, lot.owner, msg.sender);
      
      // update the lot information
      lots[_tokenId].owner = payable(msg.sender);
      lots[_tokenId].status = LotStatus.OffMarket;

       //return extra payment
      if (msg.value > lot.price)
          payable(msg.sender).transfer(msg.value - lot.price);
}

```
Buyers can acquire Lego lots by specifying the tokenId associated with the desired Lot. The transaction records the status as 'Sold,' and the lot's status transitions to 'OffMarket' to reflect the successful purchase. It's worth noting that this function currently serves as a proof of concept; in a production environment, an escrow mechanism would typically be incorporated to facilitate safe and secure transactions, where the buyer can confirm receipt before funds are released to the seller. For the purposes of this contract, the existing implementation, which transfers funds directly to the seller, should suffice.


#### Canceling Lego Lots
```solidity
function cancelLot(uint256 _tokenId) public validSender {
    // contract owner can cancel the lot
    if (msg.sender != owner()) {
        // sender must own token
        require(isOwnerOf(_tokenId, msg.sender), "You are not the owner");
    }

    // lot must be on sale
    if (lots[_tokenId].status == LotStatus.OnSale) {
        lots[_tokenId].status = LotStatus.OffMarket;
        pendingLotCount.decrement();
        emit LotCancel(_tokenId);
    }
}
```
Sellers have the option to cancel their lot listings by providing the tokenId associated with their listing. Additionally, the contract owner possesses the authority to cancel all listings.

#### Reselling Lego Lots
```solidity
function updateLot(
       uint256 _tokenId,
       string memory _title,
       string memory _description,
       string memory _date,
       uint256 _price,
       string memory _uri,
       LotType _lotType,
       LotStatus _status
) public validSender {
    // must be the owner
    require(isOwnerOf(_tokenId, msg.sender), "You are not the owner");
    Lot memory lot = findLot(_tokenId);

    if (lot.status == LotStatus.OnSale && _status == LotStatus.OffMarket) {
        pendingLotCount.decrement();
    } else if (
        lot.status == LotStatus.OffMarket && _status == LotStatus.OnSale
    ) {
        pendingLotCount.increment();
    }

    if (_status != LotStatus.None) {
        lot.status = _status;
    }
    if (_lotType != LotType.None) {
        lot.lotType = _lotType;
    }
    if (_price > 0) {
        lot.price = _price;
    }
    if (bytes(_title).length > 0) {
        lot.title = _title;
    }
    if (bytes(_description).length > 0) {
        lot.description = _description;
    }
    if (bytes(_date).length > 0) {
        lot.date = _date;
    }
    if (bytes(_uri).length > 0) {
        lot.uri = _uri;
    }

    lots[_tokenId] = lot;

    emit LotUpdated(lot);
}
```
Lot owners have the flexibility to resell their NFTs by utilizing the **updateLot** function. This function accepts a tokenId and permits the owner to modify various aspects of their lot, including the price. Notably, most parameters are optional, ensuring that any empty parameter values will not trigger a change in the lot's state.

#### Discovering Lots
```solidity
function findAllPending() public view returns (uint256[] memory ids) {
    if (pendingLotCount.current() == 0) {
        return (new uint[](0));
    }

    uint256 arrLength = lots.length;
    ids = new uint256[](pendingLotCount.current());
    uint256 idx = 0;
    for (uint i = 0; i < arrLength; ++i) {
        Lot memory lot = lots[i];
        if (lot.status == LotStatus.OnSale) {
            ids[idx] = lot.id;
            idx++;
        }
    }
}

/* Return the token ID's that belong to the caller */
function findMyLots() public view returns (uint256[] memory myLots) {
    require(msg.sender != address(0));
    uint256 numOftokens = balanceOf(msg.sender);
    if (numOftokens == 0) {
        return new uint256[](0);
    } else {
        myLots = new uint256[](numOftokens);
        uint256 idx = 0;
        uint256 arrLength = lots.length;
        for (uint256 i = 0; i < arrLength; i++) {
            if (ownerOf(i) == msg.sender) {
                myLots[idx] = i;
                idx++;
            }
        }
    }
}
```
The **findAllPending** function allows you to retrieve an array of token IDs currently available for purchase, specifically those listed as 'OnSale.' On the other hand, **findMyLots** returns the token IDs corresponding to lots owned by the caller. It's important to note that future iterations of these functions may require pagination for more complex scenarios, but for our current straightforward purposes, they serve their intended functions effectively.


### Conclusion
The full implementation of this contract is accessible on [GitHub](https://github.com/flomang/kisx/blob/main/hardhat-contracts/contracts/KisxPortal.sol), allowing you to explore its inner workings and intricacies in depth.

In summary, we've delved into the KisxPortal smart contract, a robust and versatile tool that enables you to create and oversee a thriving Lego NFT marketplace right on the Ethereum blockchain. It boasts an array of features, including the ability to mint, purchase, resell, and manage Lego lots as NFTs.

Stay tuned for the exciting future of Lego NFTs, including the forthcoming release of a user-friendly frontend designed to complement this contract. As the world of blockchain and collectibles continues to evolve, this platform offers a glimpse into the innovative possibilities that lie ahead.
