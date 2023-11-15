<template>
  <main>
    <div v-if="connectedWallet !== ''">
      <w3m-button />
      <div>Connected Wallet: {{ connectedWallet }}</div>
      <div class="inputs">
        Tier 1: <input type="number" v-model="tier1Amount"/>
        Tier 2: <input type="number" v-model="tier2Amount"/>
        Tier 3: <input type="number" v-model="tier3Amount"/>
      </div>
      <div>
        <button @click="mint">Mint</button>
      </div>
    </div>
    <div v-else>
      <w3m-button />
    </div>
  </main>
</template>

<script>
import { createWeb3Modal, defaultConfig } from '@web3modal/ethers5/vue';
import { ethers } from 'ethers';
import contractABI from './abis/nft.json';

// 1. Get projectId
const projectId = import.meta.env.VITE_WALLET_CONNECT_PROJECT_ID

// 2. Set chains
const mainnet = {
  chainId: 11155111,
  name: 'Sepolia',
  currency: 'ETH',
  explorerUrl: 'https://sepolia.etherscan.io',
  rpcUrl: import.meta.env.SEPOLIA_RPC_URL
}

// 3. Create modal
const metadata = {
  name: 'My Website',
  description: 'My Website description',
  url: 'https://mywebsite.com',
  icons: ['https://avatars.mywebsite.com/']
}

const modal = createWeb3Modal({
  ethersConfig: defaultConfig({ metadata }),
  chains: [mainnet],
  projectId
})

async function getMerkleProofForAddress(address) {
  var myHeaders = new Headers();
  myHeaders.append("Content-Type", "application/json");

  var raw = JSON.stringify({
    "type": "isWalletWhitelisted_test",
    "walletAddress": address
  });

  var requestOptions = {
    method: 'POST',
    headers: myHeaders,
    body: raw,
    redirect: 'follow'
  };

  return fetch(import.meta.env.VITE_MERKLE_PROOF_API_ENDPOINT, requestOptions)
    .then(response => response.json())
    .then(result => result.body.proof)
    .catch(error => {
      console.log('error', error)
      return [];
    });
}

export default {
  data() {
    return {
      connectedWallet: '',
      tier1Amount: 0,
      tier2Amount: 0,
      tier3Amount: 0,
    }
  },
  methods: {
    async mint() {
      const walletProvider = modal.getWalletProvider()
      const contract = new ethers.Contract(import.meta.env.VITE_NFT_CONTRACT_ADDRESS, contractABI, walletProvider.getSigner(this.connectedWallet));
      
      const prices = await contract.tiersCost();
      const proof = await getMerkleProofForAddress(this.connectedWallet);
      const valueToSend = prices.tier1.mul(this.tier1Amount).add(prices.tier2.mul(this.tier2Amount)).add(prices.tier3.mul(this.tier3Amount));

      await contract.mint(this.tier1Amount, this.tier2Amount, this.tier3Amount, this.connectedWallet, proof, { 
        value: valueToSend 
      });
    }
  },
  mounted() {
    this.connectedWallet = modal.getAddress() ?? '';
    modal.subscribeProvider(({ address }) => {
      console.log('connected: ', address);
      this.connectedWallet = address ?? '';
    });
  }
}
</script>

<style scoped>
.inputs {
  display: flex;
  place-items: flex-start;
}
</style>
