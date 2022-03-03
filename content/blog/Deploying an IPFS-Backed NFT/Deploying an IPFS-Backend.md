---
title: Deploying an IPFS-Backed NFT
date: "2022-03-02"
description: "Reflections about my experience creating and deploying an NFT"
---

There are many ways to deploy NFTs. You can host assets on your own servers, post images only, have attributes attached, etc. This blog post will go into the most typical setup I’ve seen for a project, that being:
- Images uploaded on IPFS
- JSON metadata hosted on IPFS
- Contract hand-made (in solidity, duh)

## TL;DR

- Write the solidity contract
- Upload images to Pinata
- Upload JSON data to Pinata
- Deploy contract to rinkeby
- Verify contract on rinkeby
- Set your baseTokenURI to ipfs://<your json folder CID>/
- List your project on OpenSea: https://testnets.opensea.io/get-listed

---

Before getting started, I’ve pushed all of my code to this public template here: https://github.com/keyboard-clacker/ERC-721 Feel free to use this as your base starting point — you should probably not have to modify this code beyond a few lines like your IPFS CID or NFT name/ticker, unless you want added functionality in your contract.

# Writing Your Contract

There really isn’t anything special here you have to worry about. I used a typical OpenZeppelin setup that has basically no custom code here. In case you want a boilerplate to start off with, here ya go:

```solidity
pragma solidity ^0.8.0;
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
contract TestContract is ERC721Enumerable, Ownable {
	using SafeMath for uint256;
	using Counters for Counters.Counter;
	Counters.Counter private _tokenIds;
	uint public constant MAX_SUPPLY = 100;
	uint public constant PRICE = 0.01 ether;
	uint public constant MAX_PER_MINT = 5;
	string public baseTokenURI;
	constructor(string memory baseURI) ERC721("Test Contract", "NFTC") {
		setBaseURI(baseURI);
	}
	function reserveNFTs() public onlyOwner {
		uint totalMinted = _tokenIds.current();
		require(totalMinted.add(3) < MAX_SUPPLY, "Not enough NFTs left to reserve");
		for (uint i = 0; i < 3; i++) {
			_mintSingleNFT();
		}
	}
	function _baseURI() internal view virtual override returns (string memory) {
		return baseTokenURI;
	}
	function setBaseURI(string memory _baseTokenURI) public onlyOwner {
		baseTokenURI = _baseTokenURI;
	}
	function mintNFTs(uint _count) public payable {
		uint totalMinted = _tokenIds.current();
		require(totalMinted.add(_count) <= MAX_SUPPLY, "Not enough NFTs left!");
		require(_count >0 && _count <= MAX_PER_MINT, "Cannot mint specified number of NFTs.");
		require(msg.value >= PRICE.mul(_count), "Not enough ether to purchase NFTs.");
		for (uint i = 0; i < _count; i++) {
			_mintSingleNFT();
		}
	}
	function _mintSingleNFT() private {
		uint newTokenID = _tokenIds.current();
		_safeMint(msg.sender, newTokenID);
		_tokenIds.increment();
	}
	function tokensOfOwner(address _owner) external view returns (uint[] memory) {
		uint tokenCount = balanceOf(_owner);
		uint[] memory tokensId = new uint256[](tokenCount);
		for (uint i = 0; i < tokenCount; i++) {
			tokensId[i] = tokenOfOwnerByIndex(_owner, i);
		}
		return tokensId;
	}
	function withdraw() public payable onlyOwner {
		uint balance = address(this).balance;
		require(balance > 0, "No ether left to withdraw");
		(bool success, ) = (msg.sender).call{value: balance}("");
		require(success, "Transfer failed.");
	}
}
```

I recommend using hardhat to deploy this, since you can also use it to verify your contract pretty easily (easier a time than I had with truffle). For this, I used `@nomiclabs/hardhat-etherscan`, which requires you to create an etherscan API key, but this can be done very easily and freely here, as long as you have an etherscan account: [https://etherscan.io/myapikey](https://etherscan.io/myapikey)

As for your hardhat configuration, you need very little special stuff here as well. Here is what I’m using, I really only added that etherscan import to the top, which registers the `verify` command, but that’s basically it.

```solidity
require("@nomiclabs/hardhat-waffle");
require("@nomiclabs/hardhat-etherscan");
require('dotenv').config();
const { API_URL, PRIVATE_KEY, ETHERSCAN_API_KEY } = process.env;
// This is a sample Hardhat task. To learn how to create your own go to
// <https://hardhat.org/guides/create-task.html>
task("accounts", "Prints the list of accounts", async (taskArgs, hre) => {
  const accounts = await hre.ethers.getSigners();
  for (const account of accounts) {
    console.log(account.address);
  }
});
// You need to export an object to set up your config
// Go to <https://hardhat.org/config/> to learn more
/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
  solidity: "0.8.4",
  defaultNetwork: "rinkeby",
  networks: {
    rinkeby: {
      url: API_URL,
      accounts: [PRIVATE_KEY]
    }
  },
  etherscan: {
    apiKey: ETHERSCAN_API_KEY
  }
};
```

# Upload Images to Pinata

I recommend doing this and the following step of uploading these two folders to Pinata (or your IPFS portal of your choice) because the base URI you pass in to your contract constructor when deploying your contract will be the ipfs URL for your JSON folder, and all of those files will have `image` attributes that point to images in the IPFS folder with all of the images themselves. So you can really do this whenever you want, hell you can set your base URI to whatever random string you want really, just when you want to go publish on opensea, probably change it to your JSON folder IPFSURI or nothing will work obviously. That being said, let’s get into uploading images to Pinata.

This step is deceptively simple. Basically what you want to do is put all of your image assets in one folder, and upload that entire folder itself to Pinata. The reason you want to do this is because when you upload an entire folder to Pinata, it creates a CID, or content ID hash that is a hash resultant from the actual contents of all of the files within your folder. This will let you do something like `ipfs://<your cid>/1.png` or whatever file you want to reference in that folder. This isn’t as useful for your images folder, since your JSON will be the one pointing directly to each file so it really doesn’t matter, but might as well keep it consistent.

Back to uploading to Pinata. Here’s pretty much the steps you want to do:

- Put all of your images in a folder. Doesn’t matter the folder name, you can give it whatever name you like later.
- Go to your Pinata pin manager dashboard here: [https://app.pinata.cloud/pinmanager](https://app.pinata.cloud/pinmanager)
- Click the Upload button
- Select the “Folder” option
- Select the folder with all of the image files in it
- At the next step, name it whatever you want, it really doesn’t matter. I chose the name of my contract, skewer case, and then for the next step I did the same and just appended -json to the end of it so I knew which was which easily. Plus then they’re in order in the list when sorting by name.

That’s it for uploading images to Pinata. Pretty easy, made harder only by my lack of conviction for concision.

# Upload JSON Metadata to Pinata

This was a part that I have found is poorly descripbed/documented online. And is also the keystone piece for actually connecting your contract tokens to your IPFS assets.

In order to understand this step, you need to know two things.

1. When opensea fetches data for a token, it’ll call `tokenURI` on your contract with the given ID. If you navigate to this URI in your browser, this should eventually be a JSON object that abides by [the metadata spec described by opensea](https://docs.opensea.io/docs/metadata-standards). If it’s an IPFS URI, you might need a chrome extension to check it out.
2. That JSON file is nothing special — all it is is a file that abides by opensea’s metadata spec, served through IPFS. OpenSea takes care of all of the IPFS fetching and all the fancy stuff, for both the JSON file as well as the image itself (which also has an IPFS URI, if you’re following this guide).

I’d imagine there’s some way here to auto-generate these JSON files, but I like doing things by hand before telling a computer to do them for me. Gotta build empathy for our future robot overlords.

Back to business. First things first, you want to create one JSON file who’s name is just the token number for each and every token. So you will have a file called `0`, then `1`, `2`, `3`, I think you get the picture. However many tokens you have, you need one JSON file per token. The reason you _do not_ need the `.json` file extension is because if you try uploading one of these extension-less files and navigate to the IPFS URI, you will see in the headers of the response that IPFS is actually automatically declaring the content type, so we gucci.

Now that you have one JSON file per token all in a folder together. Note: this is a different folder than your image assets. I guess it doesn’t have to be, but seems better that way. Separation of concerns something something something. From here, you want to upload this new folder to Pinata the exact same way you uploaded your images folder. Easy peasy lemon squeezy, step done.

This is a good time to check in. How are you doing, you good? If you have any questions, tweet me @kyleholzinger, I am always down to help a fellow dev in need. Right about now you should have two folders in your IPFS pin manager dashboard, one with all of your image assets, and one with all of the corresponding JSON files per asset. If you have this, you are sitting pretty. On to the contract stuff.

# Deploy Contract to Rinkeby

After uploading your assets and JSON files to IPFS, make sure that when you’re deploying your contract, you set the baseURI to `ipfs://<your JSON folder CID>/`. You don’t necessarily have to, since you probably wrote a `setBaseURI` method on your contract (please do this), but you know what they say, measure twice, deploy once. Learned that one the hard way.

In order to deploy your contract to rinkeby, run the following command:

```bash
npx hardhat run scripts/run.js --network rinkeby
```

I made my life easier by aliasing `h` to `npx hardhat`, but it’s your life. Also, do not clear your terminal history after running this command.

# Verify Contract on Rinkeby

After running `run.js`, it should have spit out the contract address of your newly deployed contract. In order to verify this contract on etherscan, run this command:

```bash
npx hardhat verify --network rinkeby "<your contract address>" "ipfs://<your json CID>/"
```

This might take a second, but should eventually spit out a success message after a minute or two. Once it is verified, you can go to [rinkeby.etherscan.io](https://rinkeby.etherscan.io/) and see your verified contract at that address with all its pretty viewable source code. Ya love to see it.

# Set your Base URI

You might not need to do this. Just make sure your `baseURI` is set properly to the IPFS link to the folder with your JSON files in them. I repeat, JSON files, not your image assets. The JSON files should each link to the corresponding image asset— the top-level asset that opensea is requesting for should be this JSON data for each token.

# List Your Project on OpenSea

I’m assuming you’re using OpenSea here because it’s kind of standard, but do what you want. This is what I’m doing. In order to list a project that’s deployed to rinkeby on opensea, you have to actually “import” the existing token. It’s quite hidden, and used to be in the page that you get to when you click “Create”, but they’re greedy bastards and want to take all your money so they hide all the good stuff. The page to import your own custom contract ERC-721 token to OpenSea is here: [https://testnets.opensea.io/get-listed](https://testnets). You can also use this page to list something on mainnet, idk why they don’t know you want to publish to the rinkeby instance given you’re on `testnets.opensea.io`, but whatever.

All you do here is just give them the address of your deployed contract, bada bing bada boom.

# Done

You should see any tokens that you have minted already from the contract (if you did, that is) populating your collection page now. They might not have any images showing yet, I usually test it out by going to one of them and clicking the “refresh” button in the upper right. If you wait a few minutes and it’s still not showing anything, try calling `tokenURI` on your contract with one of the token IDs and navigating to that URI it gives you (something like `ipfs://lkasjdlkasjd/1`). It should be your JSON file for that token, and if you navigate to the URI specified under the `image` attribute in the JSON, that should be the image you want to display on opensea.

That’s pretty much it, let me know if there’s an easier way to do any of this. Congrats on deploying your contract, now you have to do the hard part and convince people to like your art.

