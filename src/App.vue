<template>
  <v-app>
    <v-app-bar app color="black" dark>
      <div class="d-flex align-center">
        <v-img
          alt="Logo"
          contain
          src="img/poly.png"
          transition="scale-transition"
          width="120"
        />
      </div>
    </v-app-bar>
    <v-main>
      <v-container class="py-16" fluid>
        <router-view />
      </v-container>
    </v-main>
    <v-footer color="primary lighten-1" padless>
      <v-row justify="center" no-gutters>
        <v-col class="primary lighten-2 py-4 text-center white--text" cols="12">
          {{ new Date().getFullYear() }} —
          <strong
            >Powered by
            <a href="https://github.com/mrguamos" target="_blank"
              >iSkramz</a
            ></strong
          >
        </v-col>
      </v-row>
    </v-footer>
  </v-app>
</template>
<script lang="ts">
import { defineComponent } from '@vue/composition-api'
import { provide } from '@vue/composition-api'
import Web3 from 'web3'
import {
  Characters,
  charAddress,
  sickleAddress,
  IERC20,
  PolyBlades,
  mainAddress,
} from './contracts/contracts'

export default defineComponent({
  setup() {
    const web3 = new Web3('https://polygon-rpc.com/')
    provide('web3', web3)
    const character = new web3.eth.Contract(Characters as any, charAddress)
    provide('character', character)
    const sickle = new web3.eth.Contract(IERC20 as any, sickleAddress)
    provide('sickle', sickle)
    const polyblades = new web3.eth.Contract(PolyBlades as any, mainAddress)
    provide('polyblades', polyblades)
  },
})
</script>
