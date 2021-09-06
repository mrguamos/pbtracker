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
          item-key="address"
          show-expand
          :single-expand="singleExpand"
          :expanded.sync="expanded"
        >
          <template v-slot:[`item.address`]="{ item }">
            <span class="font-weight-bold">{{ item.address }}</span>
          </template>
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
          <template v-slot:[`expanded-item`]="{ item, headers }">
            <td :colspan="headers.length" style="width: 1000px" class="px-0">
              <v-data-table
                :headers="characterHeaders"
                :items="item.characters"
                disable-pagination
                hide-default-footer
                dense
                key="id"
              >
                <template v-slot:[`item.traitName`]="{ item }">
                  <span :class="getElement(item.traitName)"></span>
                </template>
              </v-data-table>
            </td>
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
    const character: Contract = inject('character') as Contract
    const maxTax = ref(0)
    const web3 = inject('web3') as Web3
    const accountHeaders = [
      { text: 'Address', value: 'address' },
      { text: 'Name', value: 'name' },
      { text: 'Wallet Sickle', value: 'walletSickle' },
      { text: 'Wallet Matic', value: 'walletMatic' },
      { text: 'Unclaimed Sickle', value: 'unclaimedSickle' },
      { text: 'Tax', value: 'tax' },
      { text: 'Action', value: 'action' },
    ]

    const characterHeaders = [
      { text: 'Id', value: 'id' },
      { text: 'Element', value: 'traitName' },
      { text: 'Level', value: 'level' },
      { text: 'Stamina', value: 'sta' },
      { text: 'Current XP', value: 'xp' },
      { text: 'Unclaimed XP', value: 'exp' },
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

    async function init() {
      maxTax.value = await polyblades.methods.REWARDS_CLAIM_TAX_MAX().call()
      fetchAccounts()
    }

    init()

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

        const tax = await polyblades.methods
          .getOwnRewardsClaimTax()
          .call({ from: item.address })

        item.tax = ((tax * 0.15) / maxTax.value).toFixed(2)
        const charIds: any[] = await polyblades.methods
          .getMyCharacters()
          .call({ from: item.address })

        item.characters = []

        await Promise.all(
          charIds.map(async (c) => {
            const charData = characterFromContract(
              c,
              await character.methods.get(c).call()
            )
            const exp = await polyblades.methods.getXpRewards(c).call()
            const sta = await character.methods.getStaminaPoints(c).call()

            const char = { ...charData, exp: exp, sta: `${sta}/200` }
            item.characters.push(char)
          })
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

    function characterFromContract(id: number, data: any) {
      const xp = data[0]
      const name = id
      const level = parseInt(data[1], 10)
      const trait = data[2]
      const traitName = traitNumberToName(+data[2])
      const staminaTimestamp = data[3]
      const head = data[4]
      const arms = data[5]
      const torso = data[6]
      const legs = data[7]
      const boots = data[8]
      const race = data[9]
      return {
        id: +id,
        name,
        xp,
        level,
        trait,
        traitName,
        staminaTimestamp,
        head,
        arms,
        torso,
        legs,
        boots,
        race,
      }
    }

    function traitNumberToName(traitNum: number) {
      switch (traitNum) {
        case 0:
          return 'Fire'
        case 1:
          return 'Ice'
        case 2:
          return 'Lightning'
        case 3:
          return 'Dark'
        default:
          return '???'
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

    const singleExpand = ref(true)
    const expanded = ref([])

    const getElement = (element: string) => {
      switch (element) {
        case 'Fire':
          return 'fire-icon'
        case 'Ice':
          return 'earth-icon'
        case 'Lightning':
          return 'lightning-icon'
        case 'Dark':
          return 'water-icon'
        default:
          return ''
      }
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
      expanded,
      singleExpand,
      characterHeaders,
      getElement,
    }
  },
})
</script>

<style lang="scss" scoped>
.water-icon {
  color: #055050;
  content: url(../assets/water.png);
  width: 2em;
  height: 2em;
}
.fire-icon {
  color: #055050;
  content: url(../assets/fire.png);
  width: 2em;
  height: 2em;
}
.lightning-icon {
  color: #055050;
  content: url(../assets/lightning.png);
  width: 2em;
  height: 2em;
}
.earth-icon {
  color: #055050;
  content: url(../assets/earth.png);
  width: 2em;
  height: 2em;
}
</style>
