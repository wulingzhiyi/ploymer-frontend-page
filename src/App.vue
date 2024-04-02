<template>
  <div id="app">
    <div class="header">
      <span class="connect-button btn" v-show="!metamaskConnected" @click="onMetamaskConnected">
        Connect
      </span>
      <span class="header-account" v-show="metamaskConnected">
        {{ account }}
      </span>
    </div>
    <div class="body">
      <div class="body-content">
        <div class="body-content-top">
          <div class="body-content-top-top">
            <span class="body-content-top-top-from">From</span>
            <div class="body-content-top-top-chain-container">
              <img v-show="fromOp" src="https://optimism-sepolia.blockscout.com/favicon/favicon.ico" alt=""
                class="body-content-top-top-chain-logo">
              <img v-show="!fromOp" src="https://base-sepolia.blockscout.com/favicon/favicon.ico" alt=""
                class="body-content-top-top-chain-logo">
              <span class="body-content-top-top-chain-name">{{ fromOp ? 'Optimism' : 'Base' }}</span>
              <img src="./images/arrow_bottom.svg" class="body-content-top-top-chain-arrow">
            </div>
          </div>
          <div class="body-content-top-center">
            <input type="text" v-model="amount" class="body-content-top-center-input">
            <div class="body-content-top-center-right">
              <img src="https://static.optimism.io/data/USDT/logo.png" alt=""
                class="body-content-top-center-right-logo">
              <span class="body-content-top-center-right-token">
                USDT
              </span>
              <img src="./images/arrow_bottom.svg" alt="" class="body-content-top-center-right-arrow">
            </div>
          </div>
          <p class="body-content-top-bottom">
            Balance: {{ srcBalance }}
          </p>
        </div>
        <div class="body-content-toggle">
          <img src="./images/toggle.svg" @click="toggle" class="body-content-toggle-logo">
        </div>
        <div class="body-content-top">
          <div class="body-content-top-top">
            <span class="body-content-top-top-from">To</span>
            <div class="body-content-top-top-chain-container">
              <img v-show="!fromOp" src="https://optimism-sepolia.blockscout.com/favicon/favicon.ico" alt=""
                class="body-content-top-top-chain-logo">
              <img v-show="fromOp" src="https://base-sepolia.blockscout.com/favicon/favicon.ico" alt=""
                class="body-content-top-top-chain-logo">
              <span class="body-content-top-top-chain-name">{{ fromOp ? 'Base' : 'Optimism' }}</span>
              <img src="./images/arrow_bottom.svg" alt="" class="body-content-top-top-chain-arrow">
            </div>
          </div>
          <p class="body-content-top-bottom body-content-top-bottom-receive">
            You will receive: {{ amount }} {{ dstSymbol }}
          </p>
          <p class="body-content-top-bottom">Balance: {{ dstBalance }}</p>
        </div>
        <span :class="`body-content-transfer-button btn ${isLoading ? 'loading' : ''}`" @click="transfer()">
          {{ transferButtonText }}
        </span>
      </div>
    </div>
  </div>
</template>

<script>
import Web3 from 'web3';
import { ABI } from './abi';
import BigNumber from "bignumber.js";
const contractAddress = '0x844a848Ed12be1D39518d6706d66Bed24c9d52cA'
const opChainId = '0xaa37dc';
const baseChainId = '0x14a34';

export default {
  name: 'App',
  data() {
    return {
      metamaskConnected: false,
      fromOp: true,
      chainId: '',
      account: '',
      opBalance: '',
      baseBalance: '',
      amount: 0.0,
      isLoading: false,
      srcSymbol: '',
      dstSymbol: '',

    }
  },
  computed: {
    srcBalance() {
      return this.fromOp ? `${this.opBalance} ${this.srcSymbol}` : `${this.baseBalance} ${this.dstSymbol}`
    },
    dstBalance() {
      return this.fromOp ? `${this.baseBalance} ${this.dstSymbol}` : `${this.opBalance} ${this.srcSymbol}`
    },
    transferButtonText() {
      if (this.metamaskConnected) {
        console.log('computed chainId', this.chainId);
        if (this.fromOp && this.chainId !== opChainId) {
          return 'Switch to Optimism'
        } else if (!this.fromOp && this.chainId !== baseChainId) {
          return 'Switch to Base'
        } else {
          return 'Transfer'
        }
      } else {
        return 'Connect Wallet'
      }
    }
  },
  async mounted() {
    setTimeout(async () => {
      if (typeof window.ethereum !== 'undefined') {
        console.log('ethereum is defined');
        if (window.ethereum.isConnected()) {
          console.log('ethereum is connected');
          window.ethereum.on('chainChanged', (chainId) => {
            console.log('Network changed to: ' + chainId);
            this.chainId = chainId;
          });
          const accounts = await ethereum.request({ method: 'eth_accounts' })
          const chainId = await ethereum.request({ method: 'eth_chainId' });
          console.log('mounted chainId: ' + chainId);
          this.chainId = chainId;
          if (accounts && accounts.length > 0) {
            this.account = accounts[0];
            this.metamaskConnected = true;
            this.queryBalance();
          }
        }
      }
    }, 100);

  },
  methods: {
    toggle() {
      this.fromOp = !this.fromOp
    },
    async transfer() {
      if (this.metamaskConnected) {
        try {
          const chainId = await ethereum.request({ method: 'eth_chainId' });
          if (this.fromOp && this.chainId !== opChainId) {
            await window.ethereum.request({
              method: 'wallet_switchEthereumChain',
              params: [{ chainId: '0xaa37dc' }],
            });
          } else if (!this.fromOp && this.chainId !== baseChainId) {
            await window.ethereum.request({
              method: 'wallet_switchEthereumChain',
              params: [{ chainId: '0x14a34' }],
            });
          } else {
            if (this.isLoading) return;
            this.isLoading = true
            const web3 = new Web3(window.ethereum);
            await window.ethereum.enable();
            const accounts = await web3.eth.getAccounts();
            const myAccount = accounts[0];
            const constract = new web3.eth.Contract(ABI, contractAddress);
            let channelId = (this.fromOp ? 'channel-10' : 'channel-11')
            const channel = web3.utils.padRight(web3.utils.asciiToHex(channelId), 64);
            const receipt = await constract.methods.bridgePolymerToken(
              contractAddress,
              channel,
              this.toDecimal(this.amount)
            ).send({ from: myAccount });

            setTimeout(() => {
              this.isLoading = false;
              this.queryBalance()
            }, 36000)

            console.log('send universal packet successful', receipt);
          }
        } catch (error) {
          this.isLoading = false;
          console.error('send universal packet failed', error);
        }
      } else {
        this.onMetamaskConnected();
      }
    },

    toDecimal(number) {
      const num = new BigNumber(number);
      const result = num.multipliedBy(new BigNumber(10).pow(18));
      return result.toNumber();
    },
    async onMetamaskConnected() {
      try {
        await window.ethereum.enable();
        const accounts = await new Web3(window.ethereum).eth.getAccounts();
        const chainId = await ethereum.request({ method: 'eth_chainId' });
        console.log('Current chainId: ' + chainId);
        window.ethereum.on('chainChanged', (chainId) => {
          console.log('Network changed to: ' + chainId);
          this.chainId = chainId;
        });
        this.chainId = chainId;
        this.account = accounts[0];
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
          this.account,
        ).call();
        const opSymbol = await opContract.methods.symbol().call();
        let opNum = new BigNumber(Number(opBalance));
        this.opBalance = opNum.dividedBy('1e18');
        this.srcSymbol = opSymbol

        const baseContract = new (new Web3('https://sepolia.base.org')).eth.Contract(ABI, contractAddress);
        const baseBalance = await baseContract.methods.balanceOf(
          this.account,
        ).call();
        const baseSymbol = await baseContract.methods.symbol().call();
        let baseBigNumber = new BigNumber(Number(baseBalance));
        this.baseBalance = baseBigNumber.dividedBy('1e18');
        this.dstSymbol = baseSymbol
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

p {
  margin: 0;
}

.btn {
  text-align: center;
  cursor: pointer;
  background: #FF0420;
  color: #fff;
  font-weight: 500;
  border-radius: 4px;
  transition: all ease .2s;

  &:hover {
    background: #B80018;
  }

  &:active {
    background: #B80018;
  }

  &.loading {
    background: #FF0420 linear-gradient(90deg, transparent, #B80018);
    background-size: 200% 100%;
    animation: loading 2s infinite;
    cursor: not-allowed;
  }
}

@keyframes loading {
  0% {
    background-position: 100% 0;
  }

  100% {
    background-position: -100% 0;
  }
}

#app {
  background: url('./images/op_bg.png') no-repeat center center/cover;
  width: 100%;
  color: #0F111A;
  height: 100%;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;

  .header {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    height: 72px;
    flex: 0 0 72px;
    width: 100%;
    border-bottom: 1px solid #E0E2EB;
    padding: 0 20px;
    box-sizing: border-box;

    .connect-button {
      padding: 0 16px;
      height: 48px;
      line-height: 48px;
      border-radius: 8px;
    }

    .header-account {
      font-weight: 600;
      font-size: 18px;
    }
  }

  .body {
    flex: 1;
    display: flex;
    justify-content: center;
    align-items: center;

    .body-content {
      padding: 24px;
      background-color: #FBFCFE;
      border-radius: 24px;
      border: 1px solid #E0E2EB;
      box-sizing: border-box;
      width: 480px;

      .body-content-top {
        display: flex;
        flex-direction: column;
        background-color: #F2F3F8;
        border-radius: 12px;
        padding: 16px;
        box-sizing: border-box;

        .body-content-top-top {
          display: flex;
          align-items: center;

          .body-content-top-top-from {
            font-size: 14px;
            color: #404454;
            margin-right: 12px;
          }

          .body-content-top-top-chain-container {
            flex: 1;
            border-radius: 8px;
            cursor: not-allowed;
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 8px 12px 8px 8px;
            border: 1px solid #E0E2EB;
            background-color: inherit;

            .body-content-top-top-chain-logo {
              width: 20px;
              height: 20px;
            }

            .body-content-top-top-chain-name {
              font-weight: 700;
              vertical-align: middle;
              font-size: 14px;
              margin-left: 8px;
              margin-right: 6px;
            }

            .body-content-top-top-chain-arrow {
              width: 10px;
            }
          }
        }

        .body-content-top-center {
          display: flex;
          justify-content: center;
          align-items: center;
          margin: 12px 0px;
          border: 1px solid #E0E2EB;
          border-radius: 12px;
          overflow: hidden;
          height: 56px;

          .body-content-top-center-input {
            width: 232px;
            height: 100%;
            color: #0F111A;
            border: none;
            padding: 0 12px;
            box-sizing: border-box;
            outline: none;
            flex: 1;
            background: #FBFCFE;
            font-weight: 700;
            font-size: 20px;
          }

          .body-content-top-center-right {
            display: flex;
            justify-content: center;
            width: 140px;
            flex: 0 0 140px;
            align-items: center;
            border-left: 1px solid #E0E2EB;
            background: #FBFCFE;
            cursor: not-allowed;
            height: 100%;

            .body-content-top-center-right-logo {
              width: 20px;
              height: 20px;
            }

            .body-content-top-center-right-token {
              font-weight: 700;
              font-size: 14px;
              margin-left: 8px;
              margin-right: 6px;
            }

            .body-content-top-center-right-arrow {
              width: 10px;
            }
          }


        }

        .body-content-top-bottom {
          font-size: 14px;
          color: #404454;
        }

        .body-content-top-bottom-receive {
          margin: 8px 0 4px 0;
        }


      }

      .body-content-toggle {
        margin: 12px 0;
        display: flex;
        justify-content: center;
        align-items: center;

        .body-content-toggle-logo {
          width: 15px;
          cursor: pointer;
        }
      }

      .body-content-transfer-button {
        width: 100%;
        height: 60px;
        line-height: 60px;
        font-weight: 700;
        border-radius: 12px;
        display: inline-block;
        box-sizing: border-box;
        margin-top: 24px;
      }
    }
  }
}
</style>
