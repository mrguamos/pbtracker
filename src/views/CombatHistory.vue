<template>
  <div>
    <v-alert
      type="warning"
      color="primary darken-4"
      elevation="24"
      dismissible
      class="text-center"
    >
      This covers the latest 1,000 transactions of your account. If it's not on
      the list, try next page.
    </v-alert>
    <v-card class="mx-auto">
      <v-card-title>Combat History</v-card-title>
      <v-card-text>
        <v-data-table
          disable-sort
          :loading="getCombatHistoryLoading"
          :headers="combatHistoryHeaders"
          :items="combats"
          disable-pagination
          hide-default-footer
          item-key="hash"
        >
          <template v-slot:[`item.hash`]="{ item }">
            <div class="font-weight-bold short-address">{{ item.hash }}</div>
          </template>
        </v-data-table>
        <div class="text-center pt-2">
          <v-pagination
            v-model="page"
            :total-visible="10"
            :length="pageCount"
            @input="getCombatHistory()"
          ></v-pagination>
        </div>
      </v-card-text>
    </v-card>
  </div>
</template>

<script lang="ts">
import {
  defineComponent,
  ref,
  getCurrentInstance,
  inject,
} from '@vue/composition-api'
import axios from 'axios'
import { mainAddress, PolyBlades } from '../contracts/contracts'
import Web3 from 'web3'
import { ethers } from 'ethers'

export default defineComponent({
  setup() {
    const web3 = inject('web3') as Web3
    const vm: any = getCurrentInstance()
    const combats = ref([] as any)
    const IPolyblades = new ethers.utils.Interface(PolyBlades)
    const page = ref(1)
    const pageCount = ref(100)
    const getCombatHistoryLoading = ref(false)
    const combatHistoryHeaders = [
      { text: 'Txn Hash', value: 'hash' },
      { text: 'Result', value: 'result' },
      { text: 'Character', value: 'character' },
      { text: 'Enemy Roll', value: 'enemyRoll' },
      { text: 'Player Roll', value: 'playerRoll' },
      { text: 'Sickle Gain', value: 'skillGain' },
      { text: 'Weapon', value: 'weapon' },
      { text: 'XP Gain', value: 'xpGain' },
      { text: 'Date', value: 'date' },
    ]

    async function getCombatHistory() {
      combats.value = []
      getCombatHistoryLoading.value = true
      const response = await axios.get(
        `https://api.polygonscan.com/api?module=account&action=txlist&address=${vm.proxy.$route.params.address}&contractaddress=${mainAddress}&sort=desc&apikey=NZ24CRNHW9PTFG91UG9NXX67YKXE3TN9R6&page=${page.value}&offset=1000`
      )
      const result = response.data.result
      if (result) {
        const transactions = await Promise.all(
          result.map(async (r: any) => {
            const transaction: any = await web3.eth.getTransactionReceipt(
              r.hash
            )
            const t: any = {}
            t.hash = r.hash
            t.date = new Date(r.timeStamp * 1000).toLocaleString()
            if (transaction.logs) {
              try {
                if (
                  transaction.logs[0].topics.indexOf(
                    '0x7a58aac6530017822bf3210fccef7efa31f56277f19966bc887bfb11f40ca96d'
                  ) >= 0
                ) {
                  const data = IPolyblades.parseLog(transaction.logs[0])
                  const { character } = data.args
                  const { enemyRoll } = data.args
                  const { playerRoll } = data.args
                  const { skillGain } = data.args
                  const { weapon } = data.args
                  const { xpGain } = data.args
                  t.character = parseInt(character._hex, 16)
                  t.enemyRoll = enemyRoll
                  t.playerRoll = playerRoll
                  t.skillGain = parseInt(skillGain._hex, 16)
                  t.skillGain /= Math.pow(10, 18)
                  t.weapon = parseInt(weapon._hex, 16)
                  t.xpGain = xpGain
                  t.result = xpGain == 0 ? 'Lose' : 'Win'
                  return t
                }

                //console.log(data)
              } catch (error) {
                console.log(error)
              }
            }
          })
        )

        combats.value = transactions.filter((t) => {
          if (t) return t
        })
      }
      getCombatHistoryLoading.value = false
    }

    getCombatHistory()
    return {
      combats,
      page,
      pageCount,
      combatHistoryHeaders,
      getCombatHistoryLoading,
      getCombatHistory,
    }
  },
})
</script>

<style></style>
