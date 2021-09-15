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
        style="color: green; font-weight: bold"
        >PolyBlades</a
      >. This tracker only needs your wallet address. Never give your seed / key
      phrase to anyone.
    </v-alert>
    <Dashboard />
    <Accounts class="mt-10" />
  </div>
</template>

<script lang="ts">
import Accounts from '@/components/Accounts.vue'
import Dashboard from '@/components/Dashboard.vue'
import { defineComponent } from '@vue/composition-api'
export default defineComponent({
  components: { Dashboard, Accounts },
  setup() {
    function copyText(text: any) {
      navigator.clipboard.writeText(text)
    }
    return { copyText }
  },
})
</script>
