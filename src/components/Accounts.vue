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
        </v-data-table>
      </v-card-text>
    </v-card>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, inject } from '@vue/composition-api'
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
    const web3 = inject('web3') as any
    const accountHeaders = [
      { text: 'Address', value: 'address' },
      { text: 'Name', value: 'name' },
      { text: 'Wallet Sickle', value: 'walletSickle' },
      { text: 'Wallet Matic', value: 'walletMatic' },
      { text: 'Unclaimed Sickle', value: 'unclaimedSickle' },
    ]

    const accountsLoading = ref(false)
    const account = reactive({
      name: '',
      address: '',
    })
    const { accounts } = useAccounts()

    const storedAccounts: any[] = JSON.parse(localStorage.getItem('addresses')!)

    accounts.value = storedAccounts ? storedAccounts : []

    const fetchAccounts = async () => {
      let accountsState: any[] = []
      accountsLoading.value = true
      await Promise.all(
        accounts.value.map(async (item) => {
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

            accountsState.push(item)
          } catch (error) {
            console.log('Invalid Address', error)
          }
        })
      )
      accounts.value = accountsState
      accountsLoading.value = false
    }

    fetchAccounts()

    function addAccount() {
      const exists = accounts.value.some(
        (a) => a['address'] === account.address
      )
      if (!exists) {
        accounts.value.push({ ...account })
        fetchAccounts()
        localStorage.setItem('addresses', JSON.stringify(accounts.value))
      }
      account.name = ''
      account.address = ''
    }

    return { account, addAccount, accountsLoading, accounts, accountHeaders }
  },
})
</script>

<style></style>
