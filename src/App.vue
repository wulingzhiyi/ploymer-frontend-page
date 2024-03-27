<template>
  <div id="app">
    <div class="body">
      <p class="title">Polymer Bridge</p>
      <div class="container">
        <button class="connect-button btn" v-show="!metamaskConnected" @click="onWalletConnect">
          Connect metamask
        </button>
        <span class="account-info" v-show="metamaskConnected">
          {{ currentAccount }}
        </span>
      </div>
      <div class="body-center">
        <div class="input-container" v-show="metamaskIsInstalled">
          <input type="text" v-model="bridgeAmount" class="bridge-input" placeholder="Transfer amount from op or base">
        </div>
        <div class="input-container" v-show="metamaskIsInstalled">
          <button style="margin-right:20px;" class="transfer-button btn" :disable="isLoading"
            @click="transfer('op')">Transfer to Base</button>
          <button class="transfer-button btn" :disable="isLoading" @click="transfer('base')">Transfer to
            Optimism</button>
        </div>
        <div class="loading-container" v-show="isLoading">
          Loading...
        </div>
        <div class="chain-container">
          <span class="balance-container">
            Optimism balance: {{ currentAccountOpBalance }} {{ srcTokenSymbol }}
          </span>
          <span class="balance-container">
            Base balance: {{ currentAccountBaseBalance }} {{ dstTokenSymbol }}
          </span>
        </div>
      </div>


    </div>
  </div>
</template>

<script>
import Web3 from 'web3';
import { ABI } from './abi';
import BigNumber from "bignumber.js";
const contractAddress = '0x844a848Ed12be1D39518d6706d66Bed24c9d52cA'

export default {
  name: 'App',
  data() {
    return {
      metamaskIsInstalled: false,
      metamaskConnected: false,
      currentAccount: '',
      currentAccountOpBalance: '',
      currentAccountBaseBalance: '',
      bridgeAmount: '',
      isLoading: false,
      srcTokenSymbol: '',
      dstTokenSymbol: '',
    }
  },
  async mounted() {
    if (typeof window.ethereum !== 'undefined') {
      this.metamaskIsInstalled = true;
      if (window.ethereum.isConnected()) {
        const accounts = await ethereum.request({ method: 'eth_accounts' })
        if (accounts && accounts.length > 0) {
          this.currentAccount = accounts[0];
          this.metamaskConnected = true;
          this.queryBalance();
        }
      }
    }

  },
  methods: {

    async transfer(from) {

      console.log(this.bridgeAmount)
      try {
        this.isLoading = true
        const chainId = await ethereum.request({ method: 'eth_chainId' });
        if (from === 'op' && chainId !== '0xaa37dc') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0xaa37dc' }],
          });
        } else if (from === 'base' && chainId !== '0x14a34') {
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0x14a34' }],
          });
        }
        const web3 = new Web3(window.ethereum);
        await window.ethereum.enable();
        const accounts = await web3.eth.getAccounts();
        const myAccount = accounts[0];
        const constract = new web3.eth.Contract(ABI, contractAddress);
        let channelId = (from === 'op' ? 'channel-10' : 'channel-11')
        const channel = web3.utils.padRight(web3.utils.asciiToHex(channelId), 64);
        const receipt = await constract.methods.bridgePolymerToken(
          contractAddress,
          channel,
          this._multiplyBy10Pow18(this.bridgeAmount)
        ).send({ from: myAccount });

        setTimeout(() => {
          this.isLoading = false;
          this.queryBalance()
        }, 20000)

        console.log('send universal packet successful', receipt);
      } catch (error) {
        this.isLoading = false;
        console.error('send universal packet failed', error);
      }
    },

    _multiplyBy10Pow18(number) {
      const num = new BigNumber(number);
      const result = num.multipliedBy(new BigNumber(10).pow(18));
      return result.toNumber();
    },
    async onWalletConnect() {
      try {
        await window.ethereum.enable();
        const accounts = await new Web3(window.ethereum).eth.getAccounts();
        this.currentAccount = accounts[0];
        this.metamaskConnected = true;
        this.queryBalance();
      } catch (error) {
        console.error('connect wallet failed', error);
      }

    },
    async queryBalance() {
      try {
        const opContract = new (new Web3('https://sepolia.optimism.io')).eth.Contract(ABI, contractAddress);
        const opBalance = await opContract.methods.balanceOf(
          this.currentAccount,
        ).call();
        const opSymbol = await opContract.methods.symbol().call();
        let opNum = new BigNumber(Number(opBalance));
        this.currentAccountOpBalance = opNum.dividedBy('1e18');
        this.srcTokenSymbol = opSymbol

        const baseContract = new (new Web3('https://sepolia.base.org')).eth.Contract(ABI, contractAddress);
        const baseBalance = await baseContract.methods.balanceOf(
          this.currentAccount,
        ).call();
        const baseSymbol = await baseContract.methods.symbol().call();
        let baseBigNumber = new BigNumber(Number(baseBalance));
        this.currentAccountBaseBalance = baseBigNumber.dividedBy('1e18');
        this.dstTokenSymbol = baseSymbol
      } catch (error) {
        console.error(error);
      }
    },
  }

}
</script>

<style lang="scss">
html,
body {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
}

#app {
  background-color: #101112;
  width: 100%;
  color: #fff;
  height: 100%;

  .body {
    width: 100%;
    height: 100%;
    padding-top: 20px;
    display: flex;
    flex-direction: column;
    align-items: center;

    .btn {
      background-color: #409eff;
      color: #fff;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      outline: none;
      transition: .3s;
      min-height: 28px;

      &:hover {
        background-color: #66b1ff;
      }
    }

    .title {
      font-size: 28px;

    }

    .container {
      margin-bottom: 10px;
      width: 100%;
      display: flex;
      justify-content: flex-end;
      padding-right: 40px;

      .connect-button {
        padding: 3px 6px;

      }

    }

    .body-center {
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      min-height: 400px;
      border: 1px solid yellow;
      ;
      background: #101112;
      border-radius: 10px;
    }

    .input-container {
      margin: 10px 0;
      display: flex;


      .bridge-input {
        width: 300px;
        height: 22px;
        margin: 0 20px;
      }

    }

    .chain-container {
      display: flex;
      flex-direction: column;
      .chain-common-wrap {
        width: 300px;
        height: 40px;
        border-radius: 5px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
      }

      .chain-line {
        display: flex;
        align-items: center;
        margin: 0 20px;
      }
    }


  }
}
</style>
