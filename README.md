


// blockchain.js
const { ethers } = require("ethers");
const fs = require("fs");
require("dotenv").config();


const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);

const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

// 3. ABI + عنوان العقد
const landAbi = JSON.parse(fs.readFileSync("./LandRegistry.json", "utf8"));
const landAddress = process.env.LAND_CONTRACT_ADDRESS;


const landContract = new ethers.Contract(landAddress, landAbi, wallet);

module.exports = {
  provider,
  wallet,
  landContract,
};


RPC_URL=http://127.0.0.1:8545
PRIVATE_KEY=0x123456...
LAND_CONTRACT_ADDRESS=0xAbC123...عنوان_العقد


---

ب) استدعاء العقد من الـ API – 

// app.js
const express = require("express");
const app = express();
app.use(express.json());

const { landContract } = require("./blockchain");

// إنشاء أرض جديدة على البلوكشين
app.post("/lands/onchain", async (req, res) => {
  try {
    const { price } = req.body; // بالوي
    const tx = await landContract.createLand(price);
    const receipt = await tx.wait();
    res.json({ message: "Land created on blockchain", txHash: receipt.hash });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Blockchain error" });
  }
});

// تحديث السعر على البلوكشين
app.put("/lands/onchain/:id/price", async (req, res) => {
  try {
    const { newPrice } = req.body;
    const landId = req.params.id;
    const tx = await landContract.updatePrice(landId, newPrice);
    const receipt = await tx.wait();
    res.json({ message: "Price updated", txHash: receipt.hash });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Blockchain error" });
  }
});

// شراء أرض
app.post("/lands/onchain/:id/buy", async (req, res) => {
  try {
    const landId = req.params.id;
    const { price } = req.body; // 

    const tx = await landContract.buyLand(landId, {
      value: price, // 
    });
    const receipt = await tx.wait();
    res.json({ message: "Land bought", txHash: receipt.hash });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Blockchain error" });
  }
});

app.listen(3000, () => console.log("Server running on 3000"));




---

2) ربط منصة توثيق الشهادات بالبلوكشين



// cert-blockchain.js
const { ethers } = require("ethers");
const fs = require("fs");
require("dotenv").config();

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

const certAbi = JSON.parse(fs.readFileSync("./CertificateRegistry.json", "utf8"));
const certAddress = process.env.CERT_CONTRACT_ADDRESS;

const certContract = new ethers.Contract(certAddress, certAbi, wallet);

module.exports = { certContract };

ب) API يسجل شهادة على البلوكشين

// cert-app.js
const express = require("express");
const app = express();
app.use(express.json());

const { certContract } = require("./cert-blockchain");

// إضافة شهادة
app.post("/certificates/onchain", async (req, res) => {
  try {
    const { certId, certHash, ownerName, issuer, issueDate } = req.body;
    const tx = await certContract.addCertificate(
      certId,
      certHash,
      ownerName,
      issuer,
      issueDate
    );
    const receipt = await tx.wait();
    res.json({ message: "Certificate added on chain", txHash: receipt.hash });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Blockchain error" });
  }
});

// التحقق من شهادة
app.get("/certificates/onchain/:id", async (req, res) => {
  try {
    const certId = req.params.id;
    const result = await certContract.verifyCertificate(certId);
    // result = (valid, ownerName, issuer)
    res.json({
      valid: result[0],
      ownerName: result[1],
      issuer: result[2],
    });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Blockchain error" });
  }
});

app.listen(4000, () => console.log("Cert server on 4000"));



---

3) ربط منصة الـNFT / المنتجات الرقمية بالبلوكشين




أ) عقد ذكي بسيط لسوق NFT (Skeleton)

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleNFTMarket {
    struct Item {
        uint256 id;
        address payable seller;
        uint256 price;
        bool sold;
    }

    mapping(uint256 => Item) public items;
    uint256 public itemCount;

    event ItemListed(uint256 id, address seller, uint256 price);
    event ItemBought(uint256 id, address buyer);

    function listItem(uint256 _price) public {
        itemCount++;
        items[itemCount] = Item(itemCount, payable(msg.sender), _price, false);
        emit ItemListed(itemCount, msg.sender, _price);
    }

    function buyItem(uint256 _id) public payable {
        Item storage item = items[_id];
        require(!item.sold, "Already sold");
        require(msg.value == item.price, "Wrong price");
        item.seller.transfer(msg.value);
        item.sold = true;
        emit ItemBought(_id, msg.sender);
    }
}


---




<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>NFT Market</title>
</head>
<body>
  <h1>NFT Market (on-chain)</h1>

  <button id="connect">Connect Wallet</button>
  <button id="list">List Item (1 ETH)</button>
  <button id="buy">Buy Item #1</button>

  <script src="https://cdn.jsdelivr.net/npm/ethers@6.7.0/dist/ethers.min.js"></script>
  <script>
    const contractAddress = "0xCONTRACT_ADDRESS_HERE";
    const abi = [
      "function listItem(uint256 _price) public",
      "function buyItem(uint256 _id) public payable"
    ];

    let provider, signer, contract;

    document.getElementById("connect").onclick = async () => {
      if (!window.ethereum) return alert("Install MetaMask");
      await window.ethereum.request({ method: "eth_requestAccounts" });
      provider = new ethers.BrowserProvider(window.ethereum);
      signer = await provider.getSigner();
      contract = new ethers.Contract(contractAddress, abi, signer);
      alert("Wallet connected");
    };

    // عرض منتج
    document.getElementById("list").onclick = async () => {
      const price = ethers.parseEther("1"); // 1 ETH
      const tx = await contract.listItem(price);
      await tx.wait();
      alert("Item listed!");
    };

    // شراء منتج
    document.getElementById("buy").onclick = async () => {
      const price = ethers.parseEther("1");
      const tx = await contract.buyItem(1, { value: price });
      await tx.wait();
      alert("Item bought!");
    };
  </script>
</body>
</html
