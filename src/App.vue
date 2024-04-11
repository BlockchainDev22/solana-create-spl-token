<script setup lang="ts">
import CreateTokenForm from "./components/CreateTokenForm.vue";
import { PhantomWalletAdapter, SolflareWalletAdapter } from '@solana/wallet-adapter-wallets'
import { initWallet, WalletMultiButton } from 'solana-wallets-vue'
import "solana-wallets-vue/styles.css";
const wallets = [
  new PhantomWalletAdapter(),
  new SolflareWalletAdapter(),
]
initWallet({ wallets, autoConnect: true })
</script>

<template>
  <div class="flex lg:px-36 lg:flex-row flex-col justify-center items-center h-screen relative">
    <div class="flex flex-col lg:w-1/2 justify-center items-center w-full">
      <img alt="Vue logo" class="logo" src="./assets/logo.png" width="125" height="125" />
      <div class="mt-10">
        <h1
          class="text-5xl font-sans text-slate-800 font-bold text-center  text-transparent bg-clip-text bg-gradient-to-l from-[#7928ca] to-[#ff0080]">
          Create SPL Tokens</h1>
        <h3 class="text-center">
          You can create spl token using metaplex.<br />
          <span></span>
        </h3>
      </div>
      <div class="mt-5">
        <wallet-multi-button />
      </div>
    </div>
    <div class="flex justify-center items-center lg:w-1/2">
      <CreateTokenForm />
    </div>
  </div>
</template>