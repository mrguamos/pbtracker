<template>
  <div>
    <v-card class="mx-auto">
      <v-card-title>Accounts</v-card-title>
      <v-card-text>
        <v-row justify="center">
          <v-col cols="12" md="5">
            <v-text-field
              v-model="account.address"
              label="Address"
              clearable
              required
              name="address"
            ></v-text-field>
          </v-col>

          <v-col cols="12" md="5">
            <v-text-field
              v-model="account.name"
              label="Account Name"
              clearable
              required
              name="name"
            ></v-text-field>
          </v-col>
        </v-row>
        <v-row justify="center">
          <v-col cols="12" md="4">
            <v-btn block color="primary" @click="addAccount()"> Submit </v-btn>
          </v-col>
        </v-row>
        <v-data-table
          class="pt-10"
          :loading="accountsLoading"
          :headers="accountHeaders"
          :items="accounts"
          disable-pagination
          hide-default-footer
          key="address"
        >
          <template v-slot:[`item.action`]="{ item, index }">
            <div class="text-no-wrap">
              <v-btn color="primary" outlined @click="removeAccount(index)">
                <v-icon color="grey">mdi-delete</v-icon>
              </v-btn>
              <v-btn
                class="ml-1"
                color="primary"
                outlined
                @click="refreshAccount(item, index)"
              >
                <v-icon color="grey">mdi-refresh</v-icon>
              </v-btn>
            </div>
          </template>
        </v-data-table>
      </v-card-text>
    </v-card>
    <v-snackbar v-model="snackbar" bottom>
      Invalid Address

      <template v-slot:action="{ attrs }">
        <v-btn color="pink" text v-bind="attrs" @click="snackbar = false">
          Close
        </v-btn>
      </template>
    </v-snackbar>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, inject } from '@vue/composition-api'
import Web3 from 'web3'
import { Contract } from 'web3-eth-contract'
import { useAccounts } from '../store/accounts'

export default defineComponent({
  methods: {
    forceUpdate() {
      this.$forceUpdate()
    },
  },
  setup() {
    const sickle: Contract = inject('sickle') as Contract
    const polyblades: Contract = inject('polyblades') as Contract
    const web3 = inject('web3') as Web3
    const accountHeaders = [
      { text: 'Address', value: 'address' },
      { text: 'Name', value: 'name' },
      { text: 'Wallet Sickle', value: 'walletSickle' },
      { text: 'Wallet Matic', value: 'walletMatic' },
      { text: 'Unclaimed Sickle', value: 'unclaimedSickle' },
      { text: 'Action', value: 'action' },
    ]
    const snackbar = ref(false)
    const accountsLoading = ref(false)
    const account = reactive({
      name: '',
      address: '',
    })
    const { accounts } = useAccounts()

    const storedAccounts: any[] = JSON.parse(localStorage.getItem('addresses')!)

    accounts.value = storedAccounts ? storedAccounts : []

    const fetchAccounts = async () => {
      accountsLoading.value = true
      await Promise.all(
        accounts.value.map(async (item, index) => {
          await fetchAccount(item, index)
        })
      )
      accountsLoading.value = false
    }

    fetchAccounts()

    async function refreshAccount(item: any, index: number) {
      accountsLoading.value = true
      await fetchAccount(item, index)
      accountsLoading.value = false
    }

    async function fetchAccount(item: any, index: number | null = null) {
      try {
        let walletSickle = Number(
          await sickle.methods.balanceOf(item.address).call()
        )
        walletSickle = Number(
          parseFloat(
            web3.utils.fromWei(BigInt(walletSickle).toString(), 'ether')
          ).toFixed(4)
        )
        item.walletSickle = walletSickle

        let walletMatic = Number(await web3.eth.getBalance(item.address))
        walletMatic = Number(
          parseFloat(
            web3.utils.fromWei(BigInt(walletMatic).toString(), 'ether')
          ).toFixed(4)
        )
        item.walletMatic = walletMatic

        let unclaimedSickle = await polyblades.methods
          .getTokenRewards()
          .call({ from: item.address })

        unclaimedSickle = Number(
          parseFloat(
            web3.utils.fromWei(BigInt(unclaimedSickle).toString(), 'ether')
          ).toFixed(4)
        )

        item.unclaimedSickle = unclaimedSickle
        if (index != null) {
          accounts.value[index] = item
        } else {
          accounts.value.push(item)
        }
      } catch (error) {
        console.log('Invalid Address', error)
      }
    }

    async function addAccount() {
      if (web3.utils.isAddress(account.address)) {
        const exists = accounts.value.some(
          (a) => a['address'] === account.address
        )
        if (!exists) {
          accountsLoading.value = true
          await fetchAccount({ ...account })
          accountsLoading.value = false
          localStorage.setItem('addresses', JSON.stringify(accounts.value))
        }
        account.name = ''
        account.address = ''
      } else {
        snackbar.value = true
      }
    }

    function removeAccount(index: number) {
      accounts.value.splice(index, 1)
      localStorage.setItem('addresses', JSON.stringify(accounts.value))
    }

    return {
      account,
      addAccount,
      accountsLoading,
      accounts,
      accountHeaders,
      snackbar,
      removeAccount,
      refreshAccount,
    }
  },
})
</script>

<style></style>
