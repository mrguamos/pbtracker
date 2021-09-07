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
          disable-sort
          class="pt-10 table-cursor"
          :loading="accountsLoading"
          :headers="accountHeaders"
          :items="accounts"
          disable-pagination
          hide-default-footer
          item-key="address"
          show-expand
          :single-expand="singleExpand"
          :expanded.sync="expanded"
          @click:row="expandRow"
        >
          <template v-slot:[`item.address`]="{ item }">
            <div class="font-weight-bold ellipsis">{{ item.address }}</div>
          </template>
          <template v-slot:[`item.action`]="{ item, index }">
            <div class="text-no-wrap">
              <v-btn
                color="primary"
                outlined
                @click.stop.prevent="removeAccount(index)"
              >
                <v-icon color="grey">mdi-delete</v-icon>
              </v-btn>
              <v-btn
                class="ml-1"
                color="primary"
                outlined
                @click.stop.prevent="refreshAccount(item, index)"
              >
                <v-icon color="grey">mdi-refresh</v-icon>
              </v-btn>
            </div>
          </template>
          <template v-slot:[`expanded-item`]="{ item, headers }">
            <td :colspan="headers.length" style="width: 1000px" class="px-0">
              <v-data-table
                disable-sort
                class="table-cursor"
                :headers="characterHeaders"
                :items="item.characters"
                disable-pagination
                hide-default-footer
                dense
                @click:row="showDialog"
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

    <div class="text-center">
      <v-dialog v-model="dialog" width="600">
        <v-card>
          <v-card-title class="text-h5 primary lighten-2">
            Combat Simulator - {{ selectedCharId }}
          </v-card-title>

          <v-card-text>
            <div class="pa-2">
              <v-select
                v-model="selectedWeapon"
                :items="weapons"
                label="Select Weapon"
                item-value="id"
                @change="simulate"
              >
                <template slot="selection" slot-scope="data">
                  {{ data.item.stars + 1 }}
                  <v-icon color="yellow">mdi-star</v-icon
                  >{{ data.item.element }} - {{ data.item.id }}
                </template>
                <template slot="item" slot-scope="data">
                  {{ data.item.stars + 1 }}
                  <v-icon color="yellow">mdi-star</v-icon
                  >{{ data.item.element }} - {{ data.item.id }}
                </template>
              </v-select>

              <div style="text-align: center">
                <p class="text-h6 font-weight-black" style="color: black">
                  Enemies
                </p>
              </div>
              <v-data-table
                disable-sort
                :headers="chanceHeaders"
                :items="chances"
                disable-pagination
                hide-default-footer
                dense
                :loading="chancesLoading"
              >
                <template v-slot:[`item.traitName`]="{ item }">
                  <span :class="getElement(item.traitName)"></span>
                </template>
              </v-data-table>
            </div>
          </v-card-text>

          <v-divider></v-divider>

          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn
              color="primary"
              text
              @click="simulate(selectedWeapon)"
              v-if="selectedWeapon != 0"
            >
              Refresh
            </v-btn>
            <v-btn color="primary" text @click="dialog = false"> Close </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-dialog v-model="loadingDialog" hide-overlay persistent width="300">
        <v-card color="green" dark>
          <v-card-text>
            Please stand by
            <v-progress-linear
              indeterminate
              color="white"
              class="mb-0"
            ></v-progress-linear>
          </v-card-text>
        </v-card>
      </v-dialog>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, inject } from '@vue/composition-api'
import Web3 from 'web3'
import { Contract } from 'web3-eth-contract'
import { useAccounts } from '../store/accounts'
import { WeaponTrait, WeaponElement } from '../contracts/contracts'

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
    const weapon: Contract = inject('weapon') as Contract
    const maxTax = ref(0)
    const web3 = inject('web3') as Web3
    const accountHeaders = [
      { text: 'Address', value: 'address' },
      { text: 'Name', value: 'name' },
      { text: 'Wallet Sickle', value: 'walletSickle' },
      { text: 'Wallet Matic', value: 'walletMatic' },
      { text: 'Unclaimed Sickle', value: 'unclaimedSickle' },
      { text: 'Total Sickle', value: 'totalSickle' },
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

    const chanceHeaders = [
      { text: 'Element', value: 'traitName' },
      { text: 'Power', value: 'power' },
      { text: 'Chance', value: 'chance' },
    ]
    const snackbar = ref(false)
    const accountsLoading = ref(false)
    const account = reactive({
      name: '',
      address: '',
    })
    const { accounts } = useAccounts()

    const storedAccounts: any[] = JSON.parse(localStorage.getItem('addresses')!)

    const fetchAccounts = async () => {
      accountsLoading.value = true
      const accs = await Promise.all(
        storedAccounts.map(async (item) => {
          return await fetchAccount(item)
        })
      )
      accounts.value = accs
      accountsLoading.value = false
    }

    async function init() {
      maxTax.value = await polyblades.methods.REWARDS_CLAIM_TAX_MAX().call()
      fetchAccounts()
    }

    init()

    async function refreshAccount(item: any, index: number) {
      accountsLoading.value = true
      const account = await fetchAccount(item, index)
      accounts.value[index] = account
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

        item.totalSickle = (
          Number(walletSickle) + Number(unclaimedSickle)
        ).toFixed(4)

        const tax = await polyblades.methods
          .getOwnRewardsClaimTax()
          .call({ from: item.address })

        item.tax = ((tax * 0.15) / maxTax.value).toFixed(2)

        item.characters = await getCharacters(item.address)

        return item
      } catch (error) {
        console.log('Invalid Address', error)
      }
    }

    async function getCharacters(address: string) {
      const charIds: number[] = await polyblades.methods
        .getMyCharacters()
        .call({ from: address })

      const chars = await Promise.all(
        charIds.map(async (charId) => {
          const c = await character.methods.get(charId).call()
          const charData = characterFromContract(charId, c)
          const exp = await polyblades.methods.getXpRewards(charId).call()
          const sta = await character.methods.getStaminaPoints(charId).call()
          return {
            ...charData,
            exp: exp,
            sta: `${sta}/200`,
            ownerAddress: address,
          }
        })
      )
      return chars
    }

    async function getWeapons(address: string) {
      const weapIds: any[] = await polyblades.methods
        .getMyWeapons()
        .call({ from: address })
      const weapons = await Promise.all(
        weapIds.map(async (weapId) => {
          const w = await weapon.methods.get(weapId).call()
          const weaponData: any = weaponFromContract(weapId, w)
          return { ...weaponData, ownerAddress: address }
        })
      )
      return weapons
    }

    function getStatPatternFromProperties(properties: any) {
      return (properties >> 5) & 0x7f
    }

    function getStat1Trait(statPattern: number) {
      return statPattern % 5
    }

    function getStat2Trait(statPattern: number) {
      return Math.floor(statPattern / 5) % 5
    }

    function getStat3Trait(statPattern: number) {
      return Math.floor(Math.floor(statPattern / 5) / 5) % 5
    }

    function statNumberToName(statNum: number) {
      switch (statNum) {
        case WeaponTrait.CHA:
          return 'CHA'
        case WeaponTrait.DEX:
          return 'DEX'
        case WeaponTrait.INT:
          return 'INT'
        case WeaponTrait.PWR:
          return 'PWR'
        case WeaponTrait.STR:
          return 'STR'
        default:
          return '???'
      }
    }

    function getWeaponTraitFromProperties(properties: any) {
      return (properties >> 3) & 0x3
    }

    function weaponFromContract(id: number, data: any) {
      const properties = data[0]
      const stat1 = data[1]
      const stat2 = data[2]
      const stat3 = data[3]
      const level = +data[4]
      const blade = data[5]
      const crossguard = data[6]
      const grip = data[7]
      const pommel = data[8]
      const burnPoints = +data[9]
      const bonusPower = +data[10]

      const stat1Value = +stat1
      const stat2Value = +stat2
      const stat3Value = +stat3

      const statPattern = getStatPatternFromProperties(+properties)
      const stat1Type = getStat1Trait(statPattern)
      const stat2Type = getStat2Trait(statPattern)
      const stat3Type = getStat3Trait(statPattern)

      const traitNum = getWeaponTraitFromProperties(+properties)

      const lowStarBurnPoints = burnPoints & 0xff
      const fourStarBurnPoints = (burnPoints >> 8) & 0xff
      const fiveStarBurnPoints = (burnPoints >> 16) & 0xff

      const stars = +properties & 0x7
      return {
        id: +id,
        properties,
        element: traitNumberToName(traitNum),
        stat1: statNumberToName(stat1Type),
        stat1Value,
        stat1Type,
        stat2: statNumberToName(stat2Type),
        stat2Value,
        stat2Type,
        stat3: statNumberToName(stat3Type),
        stat3Value,
        stat3Type,
        level,
        blade,
        crossguard,
        grip,
        pommel,
        stars,
        lowStarBurnPoints,
        fourStarBurnPoints,
        fiveStarBurnPoints,
        bonusPower,
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
          return ''
      }
    }

    async function addAccount() {
      if (web3.utils.isAddress(account.address)) {
        const exists = accounts.value.some(
          (a) => a['address'] === account.address
        )
        if (!exists) {
          accountsLoading.value = true
          const acc = await fetchAccount({ ...account })
          accounts.value.push(acc)
          accountsLoading.value = false
          const sa = accounts.value.map((a) => {
            return { address: a.address, name: a.name }
          })
          localStorage.setItem('addresses', JSON.stringify(sa))
        }
        account.name = ''
        account.address = ''
      } else {
        snackbar.value = true
      }
    }

    function removeAccount(index: number) {
      accounts.value.splice(index, 1)
      const sa = accounts.value.map((a) => {
        return { address: a.address, name: a.name }
      })
      localStorage.setItem('addresses', JSON.stringify(sa))
    }

    const singleExpand = ref(true)
    const expanded = ref([] as any)

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

    const dialog = ref(false)
    const loadingDialog = ref(false)

    async function showDialog(row: any) {
      selectedWeapon.value = 0
      selectedCharId.value = 0
      chances.value = []
      loadingDialog.value = true
      selectedCharId.value = row.id
      weapons.value = (await getWeapons(row.ownerAddress)) as any
      loadingDialog.value = false
      dialog.value = true
    }

    const characters = ref([])
    const weapons = ref([])
    const selectedWeapon = ref(0)

    async function simulate(weaponId: number) {
      try {
        chances.value = []
        chancesLoading.value = true
        const c = await character.methods.get(selectedCharId.value).call()
        const charData = characterFromContract(selectedCharId.value, c)
        const w = await weapon.methods.get(weaponId).call()
        const weaponData: any = weaponFromContract(weaponId, w)
        const targets = await polyblades.methods
          .getTargets(selectedCharId.value, weaponId)
          .call()
        const enemies = await getEnemyDetails(targets)
        chances.value = await Promise.all(
          enemies.map(async (enemy) => {
            const chance = getWinChance(
              charData,
              weaponData,
              enemy.power,
              enemy.trait
            )

            return {
              ...enemy,
              chance: `${(chance * 100).toFixed(2)}%`,
              traitName: traitNumberToName(enemy.trait),
            }
          })
        )
      } catch (error) {
        console.log(error)
      } finally {
        chancesLoading.value = false
      }
    }

    function getEnemyDetails(targets: any[]) {
      return targets.map((data) => {
        const n = parseInt(data, 10)
        return {
          original: data,
          power: n & 0b11111111_11111111_11111111,
          trait: n >> 24,
        }
      })
    }

    const selectedCharId = ref(0)

    function CharacterPower(level: number) {
      return (1000 + level * 10) * (Math.floor(level / 10) + 1)
    }
    function GetTotalMultiplierForTrait(wep: any, trait: number) {
      return (
        1 +
        0.01 *
          (Stat1PercentForChar(wep, trait) +
            Stat2PercentForChar(wep, trait) +
            Stat3PercentForChar(wep, trait))
      )
    }

    function Stat1PercentForChar(wep: any, trait: number) {
      return MultiplierPerEffectiveStat(
        AdjustStatForTrait(wep.stat1Value, wep.stat1Type, trait)
      )
    }

    function Stat2PercentForChar(wep: any, trait: number) {
      return MultiplierPerEffectiveStat(
        AdjustStatForTrait(wep.stat2Value, wep.stat2Type, trait)
      )
    }

    function Stat3PercentForChar(wep: any, trait: number) {
      return MultiplierPerEffectiveStat(
        AdjustStatForTrait(wep.stat3Value, wep.stat3Type, trait)
      )
    }

    function MultiplierPerEffectiveStat(statValue: number) {
      return statValue * 0.25
    }

    function AdjustStatForTrait(
      statValue: number,
      statTrait: number,
      charTrait: number
    ) {
      let value = statValue
      if (statTrait === charTrait) {
        value = Math.floor(value * 1.07)
      } else if (statTrait === WeaponTrait.PWR) {
        value = Math.floor(value * 1.03)
      }
      return value
    }

    function getElementAdvantage(playerElement: number, enemyElement: number) {
      if ((playerElement + 1) % 4 === enemyElement) return 1
      if ((enemyElement + 1) % 4 === playerElement) return -1
      return 0
    }

    function getWinChance(
      charData: any,
      weapData: any,
      enemyPower: number,
      enemyElement: number
    ) {
      const characterPower = CharacterPower(charData.level)
      const playerElement = parseInt(charData.trait, 10)

      const we = WeaponElement[weapData.element as keyof typeof WeaponElement]
      const weaponElement = parseInt(we.toString(), 10)
      const weaponMultiplier = GetTotalMultiplierForTrait(
        weapData,
        playerElement
      )
      const totalPower = characterPower * weaponMultiplier + weapData.bonusPower
      const totalMultiplier =
        1 +
        0.075 * (weaponElement === playerElement ? 1 : 0) +
        0.075 * getElementAdvantage(playerElement, enemyElement)
      const playerMin = totalPower * totalMultiplier * 0.9
      const playerMax = totalPower * totalMultiplier * 1.1
      const playerRange = playerMax - playerMin
      const enemyMin = enemyPower * 0.9
      const enemyMax = enemyPower * 1.1
      const enemyRange = enemyMax - enemyMin
      let rollingTotal = 0
      // shortcut: if it is impossible for one side to win, just say so
      //if (playerMin > enemyMax) return 3;
      if (playerMax < enemyMin) return 0

      // case 1: player power is higher than enemy power
      if (playerMin >= enemyMin) {
        // case 1: enemy roll is lower than player's minimum
        rollingTotal = (playerMin - enemyMin) / enemyRange
        // case 2: 1 is not true, and player roll is higher than enemy maximum
        rollingTotal +=
          (1 - rollingTotal) * ((playerMax - enemyMax) / playerRange)
        // case 3: 1 and 2 are not true, both values are in the overlap range. Since values are basically continuous, we assume 50%
        rollingTotal += (1 - rollingTotal) * 0.5
      } else {
        // case 1: player rolls below enemy minimum
        rollingTotal = (enemyMin - playerMin) / playerRange
        // case 2: enemy rolls above player maximum
        rollingTotal +=
          (1 - rollingTotal) * ((enemyMax - playerMax) / enemyRange)
        // case 3: 1 and 2 are not true, both values are in the overlap range
        rollingTotal += (1 - rollingTotal) * 0.5
        // since this is chance the enemy wins, we negate it
        rollingTotal = 1 - rollingTotal
      }
      return rollingTotal
    }

    function expandRow(value: any) {
      if (expanded.value.indexOf(value) >= 0) {
        expanded.value.splice(expanded.value.indexOf(value), 1)
      } else {
        expanded.value = []
        expanded.value.push(value)
      }
    }

    const chances = ref()
    const chancesLoading = ref(false)

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
      dialog,
      showDialog,
      loadingDialog,
      weapons,
      selectedWeapon,
      simulate,
      selectedCharId,
      chances,
      chanceHeaders,
      chancesLoading,
      expandRow,
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
<style lang="scss">
.table-cursor tbody tr:hover {
  cursor: pointer;
}
.ellipsis {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  -o-text-overflow: ellipsis;
}
</style>
