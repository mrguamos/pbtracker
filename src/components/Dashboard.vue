<template>
  <div>
    <v-row justify="center">
      <v-col cols="6" sm="6" md="3">
        <Card :content="totalSickle" :title="'Total Sickle'" />
      </v-col>
      <v-col cols="6" sm="6" md="3">
        <Card :content="walletSickle" :title="'Total Sickle in Wallet'" />
      </v-col>
      <v-col cols="6" sm="6" md="3">
        <Card :content="totalUnclaimedSickle" :title="'Total Unclaimed'" />
      </v-col>

      <v-col cols="6" sm="6" md="3">
        <Card :content="totalMatic" :title="'Total Matic'" />
      </v-col>
    </v-row>
    <v-row justify="center">
      <v-col cols="6" sm="6" md="3">
        <Card :content="`$${totalSickleUSD}`" :title="'Total Sickle / USD'" />
      </v-col>
      <v-col cols="6" sm="6" md="3">
        <Card :content="`$${maticUSD}`" :title="'Matic / USD'" />
      </v-col>
      <v-col cols="6" sm="6" md="3">
        <Card :content="`$${sickleUSD}`" :title="'Sickle / USD'" />
      </v-col>
      <v-col cols="6" sm="6" md="3">
        <Card :content="`$${totalMaticUSD}`" :title="'Total Matic / USD'" />
      </v-col>
    </v-row>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from '@vue/composition-api'
import Card from './Card.vue'
import { useAccounts } from '../store/accounts'
import axios from 'axios'

export default defineComponent({
  components: { Card },
  setup() {
    const { accounts } = useAccounts()
    const walletSickle = computed(() => {
      let total = 0
      accounts.value.map((item) => {
        if (!isNaN(item.walletSickle)) {
          total = total + item.walletSickle
        }
      })
      return `${total.toFixed(4)}`
    })
    const totalSickle = computed(() =>
      (Number(totalUnclaimedSickle.value) + Number(walletSickle.value)).toFixed(
        4
      )
    )
    const totalUnclaimedSickle = computed(() => {
      let total = 0
      accounts.value.map((item) => {
        if (!isNaN(item.unclaimedSickle)) {
          total = total + item.unclaimedSickle
        }
      })
      return `${total.toFixed(4)}`
    })
    const totalMatic = computed(() => {
      let total = 0
      accounts.value.map((item) => {
        if (!isNaN(item.walletMatic)) {
          total = total + item.walletMatic
        }
      })
      return `${total.toFixed(4)}`
    })

    const totalSickleUSD = computed(() => {
      return (Number(totalSickle.value) * Number(sickleUSD.value)).toFixed(4)
    })

    const totalMaticUSD = computed(() => {
      return (Number(totalMatic.value) * Number(maticUSD.value)).toFixed(4)
    })

    const sickleUSD = ref('0')
    const maticUSD = ref('0')

    async function getSickleUSD() {
      try {
        const response = await axios.get(
          'https://api.1inch.exchange/v3.0/137/quote?fromTokenAddress=0x2df507f3a084c3e053d57ef418802f56cc1b7cf8&toTokenAddress=0xc2132d05d31c914a87c6611c10748aeb04b58e8f&amount=1000000000000000000'
        )
        let value: number = response.data.toTokenAmount
        value /= Math.pow(10, 6)
        sickleUSD.value = value.toFixed(4)
      } catch (error) {
        console.log(error)
      } finally {
        setTimeout(getSickleUSD, 5000)
      }
    }

    getSickleUSD()

    async function getMaticUSD() {
      try {
        const response = await axios.get(
          'https://api.1inch.exchange/v3.0/137/quote?fromTokenAddress=0x0d500b1d8e8ef31e21c99d1db9a6444d3adf1270&toTokenAddress=0xc2132d05d31c914a87c6611c10748aeb04b58e8f&amount=1000000000000000000'
        )
        let value: number = response.data.toTokenAmount
        value /= Math.pow(10, 6)
        maticUSD.value = value.toFixed(4)
      } catch (error) {
        console.log(error)
      } finally {
        setTimeout(getMaticUSD, 5000)
      }
    }

    getMaticUSD()

    return {
      totalSickle,
      totalUnclaimedSickle,
      walletSickle,
      totalMatic,
      sickleUSD,
      maticUSD,
      totalSickleUSD,
      totalMaticUSD,
    }
  },
})
</script>
