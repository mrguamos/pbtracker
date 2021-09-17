<template>
  <div>
    <v-alert
      type="warning"
      color="primary darken-4"
      elevation="24"
      dismissible
      class="text-center"
    >
      Accepting Donations at

      <v-btn
        icon
        @click="copyText('0x816F7A84bA365963270Ca28f27a012Fd24ab0247')"
      >
        <v-icon color="green">mdi-content-copy</v-icon>
      </v-btn>

      <v-snackbar v-model="show" top>
        Copied to clipboard.

        <template v-slot:action="{ attrs }">
          <v-btn color="pink" text v-bind="attrs" @click="show = false">
            Close
          </v-btn>
        </template>
      </v-snackbar>

      Any amount will do &#128536;
    </v-alert>
    <v-alert
      type="warning"
      color="primary darken-4"
      elevation="24"
      dismissible
      class="text-center"
    >
      This tracker is not affiliated with
      <a
        href="https://polyblades.io/"
        target="_blank"
        style="color: #26c281; font-weight: bold"
        >PolyBlades</a
      >. This tracker only needs your wallet address. Never give your seed / key
      phrase to anyone.
    </v-alert>
    <v-alert
      type="warning"
      color="primary darken-4"
      elevation="24"
      dismissible
      class="text-center"
    >
      Prices are based on
      <a
        href="https://app.1inch.io/#/137/swap/0x2dF507f3A084C3e053d57EF418802f56CC1B7CF8/USDT"
        target="_blank"
        style="color: #26c281; font-weight: bold"
        >1INCH SICKLE / USDT</a
      >.
    </v-alert>
    <Dashboard />
    <Accounts class="mt-10" />
  </div>
</template>

<script lang="ts">
import Accounts from '@/components/Accounts.vue'
import Dashboard from '@/components/Dashboard.vue'
import { defineComponent, ref } from '@vue/composition-api'
export default defineComponent({
  components: { Dashboard, Accounts },
  setup() {
    const show = ref(false)
    function copyText(text: any) {
      navigator.clipboard.writeText(text)
      show.value = true
    }
    return { copyText, show }
  },
})
</script>
