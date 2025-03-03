<template>
  <div class="tx-container">
    <div
      :class="
        isWaiting
          ? 'recipient-submission-widget shine'
          : 'recipient-submission-widget'
      "
    >
      <loader v-if="isLoading" />
      <wallet-widget class="m2" v-if="!currentUser" />
      <div
        v-if="currentUser"
        :class="
          isWaiting || txError
            ? 'tx-progress-area'
            : 'tx-progress-area-no-notice'
        "
      >
        <loader class="button-loader" v-if="isWaiting" />
        <div v-if="isWaiting" class="tx-notice">
          Check your wallet for a prompt...
        </div>
        <div v-if="hasTxError || isTxRejected" class="warning-icon">⚠️</div>
        <div v-if="hasTxError" class="warning-text">
          Something failed: {{ txError }}<br />
          Check your wallet or {{ blockExplorerLabel }} for more info.
        </div>
        <div v-if="isTxRejected" class="warning-text">
          You rejected the transaction in your wallet
        </div>
      </div>
      <div class="connected" v-if="currentUser">
        <div class="total-title">
          Total to submit
          <img
            v-tooltip="{
              content:
                'Estimate – this total may be slightly different in your wallet.',
              trigger: 'hover click',
            }"
            src="@/assets/info.svg"
          />
        </div>
        <div class="total">
          {{ depositAmount }}
          <span class="total-currency"> {{ depositToken }}</span>
        </div>
        <div class="warning-text" v-if="hasLowFunds">
          Not enough {{ depositToken }} in your wallet.<br />
          Top up or connect a different wallet.
        </div>
        <div v-if="txHasDeposit" class="checkout-row">
          <p class="m05"><b>Security deposit</b></p>
          <p class="m05">
            {{ depositAmount }} {{ depositToken }}
            <span class="o5"
              >({{ fiatSign
              }}{{
                calculateFiatFee($store.state.recipientRegistryInfo.deposit)
              }})</span
            >
          </p>
        </div>
        <div class="cta">
          <button
            @click="handleSubmit"
            class="btn-action"
            :disabled="!canSubmit || isWaiting || hasLowFunds"
          >
            <div v-if="isWaiting">
              <loader class="button-loader" />
            </div>
            <div v-else>{{ cta }}</div>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import Component from 'vue-class-component'
import { Prop } from 'vue-property-decorator'
import { BigNumber } from 'ethers'
import { Web3Provider } from '@ethersproject/providers'
import { DateTime } from 'luxon'
import { EthPrice, fetchCurrentEthPrice } from '@/api/price'
import { addRecipient } from '@/api/recipient-registry-optimistic'
import { User } from '@/api/user'
import { chain } from '@/api/core'

import Loader from '@/components/Loader.vue'
import Transaction from '@/components/Transaction.vue'
import WalletWidget from '@/components/WalletWidget.vue'

import { formatAmount } from '@/utils/amounts'
import { waitForTransaction } from '@/utils/contracts'
import { RESET_RECIPIENT_DATA } from '@/store/mutation-types'

@Component({
  components: {
    Loader,
    Transaction,
    WalletWidget,
  },
})
export default class RecipientSubmissionWidget extends Vue {
  @Prop() cta!: string
  @Prop() pending!: string
  isLoading = true
  isWaiting = false
  isTxRejected = false
  txHash = ''
  txError = ''
  ethPrice: EthPrice | null = null
  fiatFee = '-'
  fiatSign = '$'

  async created() {
    this.ethPrice = await fetchCurrentEthPrice()
    this.isLoading = false
  }

  get currentUser(): User | null {
    return this.$store.state.currentUser
  }

  get walletProvider(): Web3Provider | undefined {
    return this.$web3.provider
  }

  get blockExplorerLabel(): string {
    return chain.explorerLabel
  }
  get canSubmit(): boolean {
    return (
      !!this.currentUser &&
      !!this.walletProvider &&
      !!this.$store.state.recipientRegistryInfo
    )
  }

  get hasTxError(): boolean {
    return !!this.txError
  }

  get txHasDeposit(): boolean {
    return !!this.$store.state.recipientRegistryInfo?.deposit
  }

  get depositAmount(): string {
    return this.$store.state.recipientRegistryInfo
      ? formatAmount(this.$store.state.recipientRegistryInfo.deposit, 18)
      : '...'
  }

  get hasLowFunds(): boolean {
    const { currentUser, recipientRegistryInfo } = this.$store.state

    if (currentUser?.etherBalance && recipientRegistryInfo?.deposit) {
      return currentUser.etherBalance.lt(recipientRegistryInfo.deposit)
    }
    return false
  }

  get depositToken(): string {
    return this.$store.state.recipientRegistryInfo?.depositToken ?? ''
  }

  handleSubmit(): void {
    this.addRecipient()
  }

  public calculateFiatFee(ethAmount: BigNumber): string {
    if (this.$store.state.recipientRegistryInfo && this.ethPrice) {
      return Number(
        this.ethPrice.ethereum.usd * Number(formatAmount(ethAmount, 18))
      ).toFixed(2)
    }
    return '-'
  }

  private async addRecipient() {
    const {
      currentRound,
      currentUser,
      recipient,
      recipientRegistryAddress,
      recipientRegistryInfo,
    } = this.$store.state

    this.isWaiting = true

    // Reset errors when submitting
    this.txError = ''
    this.isTxRejected = false

    if (
      recipientRegistryAddress &&
      recipient &&
      recipientRegistryInfo &&
      currentUser
    ) {
      try {
        if (currentRound && DateTime.now() >= currentRound.votingDeadline) {
          this.$router.push({
            name: 'join',
          })
          throw { message: 'round over' }
        }

        await waitForTransaction(
          addRecipient(
            recipientRegistryAddress,
            recipient,
            recipientRegistryInfo.deposit,
            currentUser.walletProvider.getSigner()
          ),
          (hash) => (this.txHash = hash)
        )

        // Send application data to a Google Spreadsheet
        if (process.env.VUE_APP_GOOGLE_SPREADSHEET_ID) {
          await fetch('/.netlify/functions/recipient', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify(recipient),
          })
        }

        this.$store.commit(RESET_RECIPIENT_DATA)
      } catch (error) {
        this.isWaiting = false
        this.txError = error.message
        return
      }
      this.isWaiting = false

      this.$router.push({
        name: 'project-added',
        params: {
          hash: this.txHash,
        },
      })
    }
  }
}
</script>

<style scoped lang="scss">
@import '../styles/vars';
@import '../styles/theme';

.tx-container {
  width: 100%;
  display: flex;
  justify-content: center;
}

.recipient-submission-widget {
  display: flex;
  flex-direction: column;
  background: $bg-primary-color;
  border-radius: 1rem;
  border: 1px solid #000;
  align-items: center;
  justify-content: center;
  width: 75%;
  margin-top: 1rem;
  @media (max-width: $breakpoint-m) {
    width: 100%;
    margin-top: 0;
  }
}

.shine {
  background: $bg-primary-color;
  background-image: linear-gradient(
    to right,
    $bg-primary-color 0%,
    $bg-secondary-color 10%,
    $bg-primary-color 40%,
    $bg-primary-color 100%
  );
  background-repeat: repeat;
  position: relative;

  -webkit-animation-duration: 8s;
  -webkit-animation-fill-mode: forwards;
  -webkit-animation-iteration-count: infinite;
  -webkit-animation-name: placeholderShimmer;
  -webkit-animation-timing-function: linear;
}

@-webkit-keyframes placeholderShimmer {
  0% {
    background-position: calc(-100vw) 0;
  }

  100% {
    background-position: calc(100vw) 0;
  }
}

.connected {
  width: 75%;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.tx-progress-area {
  background: $clr-pink-light-gradient-inactive;
  text-align: center;
  border-radius: calc(1rem - 1px) calc(1rem - 1px) 0 0;
  padding: 1.5rem;
  width: -webkit-fill-available;
  margin-bottom: 2rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  justify-content: center;
  font-weight: 500;
}

.tx-progress-area-no-notice {
  background: $clr-pink-light-gradient-inactive;
  text-align: center;
  border-radius: calc(3rem - 1px) calc(3rem - 1px) 0 0;
  width: -webkit-fill-available;
  margin-bottom: 2rem;
  display: flex;
  align-items: center;
  gap: 1rem;
  justify-content: center;
  font-weight: 500;
  height: 1rem;
}

.total {
  font-size: 64px;
  font-weight: 700;
  font-family: 'Glacial Indifference', sans-serif;
  overflow-wrap: break-word;
  width: 100%;
  text-align: center;
  line-height: 100%;
  margin-bottom: 1rem;
}

.checkout-row {
  width: 100%;
  text-align: center;
  display: flex;
  align-items: center;
  justify-content: space-between;
  @media (max-width: $breakpoint-m) {
    flex-direction: column;
    justify-content: flex-start;
  }
}

.m05 {
  margin: 0.5rem 0rem;
}

.m2 {
  margin: 2rem 0rem;
}

.o5 {
  opacity: 0.5;
}

.cta {
  margin: 2rem 0rem;
  margin-bottom: 2rem;
}

.total-currency {
  font-size: 48px;
}

.total-title {
  color: white;
  font-size: 16px;
  font-weight: 400;
  line-height: 100%;
  text-transform: uppercase;
  justify-content: flex-start;
  align-items: center;
  display: flex;
  width: fit-content;
  padding: 0.5rem;
  background: $bg-light-color;
  border-radius: 2rem;
  gap: 0.25rem;
  margin-bottom: 0.5rem;

  img {
    width: 16px;
    height: 16px;
  }
}

.btn-action-disabled {
  @include button(white, $clr-pink-light-gradient, none);
  opacity: 0.4;
  cursor: not-allowed;
}

.warning-icon {
  font-size: 24px;
}

.warning-text {
  font-size: 14px;
}

.warning-text,
.warning-icon {
  line-height: 150%;
  color: $warning-color;
  text-transform: uppercase;
  text-align: center;
}
</style>
