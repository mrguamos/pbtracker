<template>
  <v-container>
    <v-alert
      type="warning"
      color="black"
      elevation="24"
      dismissible
      class="text-center"
    >
      <span style="color: RED; font-weight: bold"
        >NEVER USE THIS UNLESS YOU KNOW WHAT YOU ARE DOING!</span
      >
    </v-alert>
    <div class="text-center">
      <v-card class="mx-auto" style="width: 500px">
        <v-card-title>Transfer</v-card-title>
        <v-card-text>
          <v-form ref="form" v-model="valid" lazy-validation>
            <v-text-field
              v-model="to"
              label="Destination Address"
              required
              name="to"
              :rules="addressRules"
            ></v-text-field>
            <v-text-field
              v-model="tokenId"
              label="Token ID"
              required
              name="tokenId"
              :rules="tokenRules"
            ></v-text-field>

            <v-radio-group
              v-model="tokenType"
              row
              required
              :rules="tokenTypeRules"
            >
              <v-radio label="Character" value="c"></v-radio>
              <v-radio label="Weapon" value="w"></v-radio>
            </v-radio-group>

            <v-btn color="primary" @click="submit"> SUBMIT </v-btn>
          </v-form>
        </v-card-text>
      </v-card>
    </div>
    <v-dialog v-model="dialog" width="500">
      <v-card>
        <v-card-title class="primary"> Transfer Result </v-card-title>

        <v-card-text class="text-center">
          {{ txnHash }}
        </v-card-text>

        <v-divider></v-divider>

        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" text @click="dialog = false"> CLOSE </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script lang="ts">
import { inject, ref } from '@vue/composition-api'
import Web3 from 'web3'
import { Contract } from 'web3-eth-contract'
import { weaponAddress, charAddress } from '../contracts/contracts'

export default {
  setup() {
    const to = ref('')
    const tokenId = ref()
    const valid = ref(true)
    const tokenType = ref('')
    const form: any = ref()
    const dialog = ref(false)
    const txnHash = ref('')

    const addressRules = [
      (v: any) => !!v || 'Destination Address is required',
      (v: any) =>
        (v && v.length >= 32) || 'Address must be at least 32 characters',
    ]

    const tokenRules = [
      (v: any) => !!v || 'Token ID is required',
      (v: any) => !isNaN(v) || 'Must be a number',
    ]

    const tokenTypeRules = [(v: any) => !!v || 'Token Type is required']

    function submit() {
      const isValid = form.value.validate()
      if (isValid) {
        transfer()
      }
    }

    const web3 = inject('web3') as Web3
    const character: Contract = inject('character') as Contract
    const weapon: Contract = inject('weapon') as Contract

    async function transfer() {
      const result = await (window as any).ethereum.request({
        method: 'eth_requestAccounts',
      })

      if (result) {
        const netId = await web3.eth.net.getId()
        if (netId !== 137) {
          await (window as any).ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: '0x89' }],
          })
        }

        try {
          let data
          let address
          if (tokenType.value === 'w') {
            address = weaponAddress
            data = weapon.methods.safeTransferFrom(
              (window as any).ethereum.selectedAddress,
              to.value,
              tokenId.value
            )
          } else if (tokenType.value === 'c') {
            address = charAddress
            data = character.methods.safeTransferFrom(
              (window as any).ethereum.selectedAddress,
              to.value,
              tokenId.value
            )
          } else {
            return
          }

          data = await data.encodeABI()

          const params = {
            data: data,
            from: (window as any).ethereum.selectedAddress,
            to: address,
          }
          txnHash.value = await (window as any).ethereum.request({
            method: 'eth_sendTransaction',
            params: [params],
          })
          dialog.value = true
        } catch (error) {
          console.log(error)
        }
      }
    }

    return {
      to,
      valid,
      submit,
      form,
      addressRules,
      tokenId,
      tokenRules,
      tokenType,
      tokenTypeRules,
      dialog,
      txnHash,
    }
  },
}
</script>

<style></style>
