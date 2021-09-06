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
  </div>
</template>

<script lang="ts">
import { computed, defineComponent } from '@vue/composition-api'
import Card from './Card.vue'
import { useAccounts } from '../store/accounts'

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
      return `${total}`
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
      return `${total}`
    })
    const totalMatic = computed(() => {
      let total = 0
      accounts.value.map((item) => {
        if (!isNaN(item.walletMatic)) {
          total = total + item.walletMatic
        }
      })
      return `${total}`
    })

    return { totalSickle, totalUnclaimedSickle, walletSickle, totalMatic }
  },
})
</script>
