<template>
  <div class="about">
    <h1 class="content-heading">Recipient guide</h1>
    <p>
      An overview of how things work as a recipient so you can learn what to
      expect throughout the duration of a funding round.
    </p>
    <div v-if="chain.bridge">
      <h2>Get funds on {{ chain.label }}</h2>
      <p>
        You'll need some {{ chain.currency }} on {{ chain.label }} in order to
        submit transactions to the clr.fund smart contracts.
        <span v-if="chain.isLayer2">
          Follow
          <links
            :to="{
              name: 'about-layer-2',
              params: {
                section: 'bridge',
              },
            }"
          >
            these steps
          </links>
          to bridge your funds to {{ chain.label }}
        </span>
        <span v-else>
          <links :to="chain.bridge">Bridge your funds here</links>
        </span>
      </p>
      <p v-if="chain.isLayer2">
        Confused on what {{ chain.label }} is?
        <links to="about/layer-2">
          Read our explanation on how clr.fund uses layer 2 on Ethereum.
        </links>
      </p>
    </div>
    <h2>Register your project</h2>
    <ol>
      <li>Head over to the <links to="/join">Join page</links>.</li>
      <li>
        Click "See round criteria" and familiarize yourself with the criteria
        for projects.
      </li>
      <li>
        Once you're familiar with the criteria and you're sure your project
        meets them, click "Add project." You'll see a series of forms to fill
        out asking for more information about your project.
      </li>
      <li>
        With the forms finished, you can finish your submission by:
        <ol>
          <li>connecting to the right network via your wallet of choice</li>
          <li>
            sending a deposit of {{ depositAmount }} {{ depositToken }} to the
            registry contract.
          </li>
        </ol>
        Projects are accepted by default, but the registry admin may remove
        projects that don't meet the criteria. Either way, your
        {{ depositToken }} will be returned once your application has been
        either accepted or denied. Note that metadata pointing to all your
        project information (but not contact information) will be stored
        publicly on-chain.
      </li>
    </ol>
    <h2>Claim your funds</h2>
    <p>
      After a clr.fund round is finished, it's simple to claim your project's
      share of the funding. Return to your project's page: you will see a "claim
      funds" button if your project received contributions during the round.
      Submit the claim transaction to receive your funds.
    </p>
    <h2>How does clr.fund work?</h2>
    <p>
      Looking for a more general overview?
      <links to="/about/how-it-works">Check out our "How It Works" page</links>.
    </p>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import Component from 'vue-class-component'
import Links from '@/components/Links.vue'
import { chain } from '@/api/core'
import { ChainInfo } from '@/plugins/Web3/constants/chains'
import { formatAmount } from '@/utils/amounts'

@Component({ components: { Links } })
export default class AboutRecipients extends Vue {
  get chain(): ChainInfo {
    return chain
  }

  get depositAmount(): string {
    return this.$store.state.recipientRegistryInfo
      ? formatAmount(this.$store.state.recipientRegistryInfo.deposit, 18)
      : '...'
  }

  get depositToken(): string {
    return this.$store.state.recipientRegistryInfo?.depositToken ?? ''
  }
}
</script>
