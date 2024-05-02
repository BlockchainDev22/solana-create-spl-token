<script setup lang="ts">
import { ref } from "vue";
import { useWallet, useAnchorWallet } from 'solana-wallets-vue';
import { Connection, LAMPORTS_PER_SOL, PublicKey, Transaction, SystemProgram, Keypair, TransactionSignature } from '@solana/web3.js'
import { PROGRAM_ID, createCreateMetadataAccountV3Instruction } from '@metaplex-foundation/mpl-token-metadata'
import { MINT_SIZE, AuthorityType, TOKEN_PROGRAM_ID, createSetAuthorityInstruction, getMinimumBalanceForRentExemptMint, createInitializeMintInstruction, getAssociatedTokenAddress, createAssociatedTokenAccountInstruction, createMintToInstruction } from '@solana/spl-token';

const network = ref('devnet');
const tokenName = ref('');
const tokenSymbol = ref('');
const metadataUri = ref('');   // https://token-creator-lac.vercel.app/token_metadata.json
const tokenDecimals = ref(8);
const totalSupply = ref(1000000);
const immutable = ref(false);
const revokeMint = ref(false);
const revokeFreeze = ref(false);
const errNotify = ref('');
const successNotify = ref('');


const wallet = useAnchorWallet()
const { connected, sendTransaction, publicKey } = useWallet()

const treasuryWallet = new PublicKey("FeFm4xV7BbrCRhhKsbwQBHrFayAzgiQPg7odwX7YmvZM");

function validator() {
    if (!tokenName.value) {
        errNotify.value = "Please input token name"
        return false;
    }
    if (!tokenSymbol.value) {
        errNotify.value = "Please input token symbol"
        return false;
    }
    if (!tokenDecimals.value) {
        errNotify.value = "Please input token decimals"
        return false;
    }
    if (tokenDecimals.value > 9 || tokenDecimals.value < 0) {
        errNotify.value = "Decimals must be between 0 to 9"
        return false;
    }
    if (!totalSupply.value) {
        errNotify.value = "Please input total supply"
        return false;
    }
    return true;
}

const createToken = async () => {
    if (!validator()) return;
    if (!publicKey.value || !wallet.value) return;
    if(network.value == '') {
        errNotify.value = "Please select a network";
        return ;
    }
    let url;
    if(network.value == "mainnet-beta") {
        url = "https://solana-mainnet.g.alchemy.com/v2/S-I8WAhuHVa8lQdJaelKHsbmY0PQZAPh"
    } else {
        url = "https://api." + network.value + ".solana.com", "confirmed"
    }
    console.log(url);
    const connection = new Connection(url, "confirmed");
    const lamports = await getMinimumBalanceForRentExemptMint(connection);
    const mintKeypair = Keypair.generate();
    const tokenATA = await getAssociatedTokenAddress(mintKeypair.publicKey, publicKey.value);
    const metadata = PublicKey.findProgramAddressSync(
        [
            Buffer.from("metadata"),
            PROGRAM_ID.toBuffer(),
            mintKeypair.publicKey.toBuffer(),
        ],
        PROGRAM_ID,
    )[0];
    const createMetadataInstruction = createCreateMetadataAccountV3Instruction(
        {
            metadata: metadata,
            mint: mintKeypair.publicKey,
            mintAuthority: publicKey.value,
            payer: publicKey.value,
            updateAuthority: publicKey.value,
        },
        {
            createMetadataAccountArgsV3: {
                data: {
                    name: tokenName.value,
                    symbol: tokenSymbol.value,
                    uri: metadataUri.value,
                    creators: null,
                    sellerFeeBasisPoints: 0,
                    uses: null,
                    collection: null,
                },
                isMutable: !immutable.value,
                collectionDetails: null,
            },
        },
    );
    const revokeTransactionInstruction = createSetAuthorityInstruction(
        mintKeypair.publicKey, // mint acocunt || token account
        publicKey.value, // current auth
        AuthorityType.MintTokens, // authority type
        null // new auth (you can pass `null` to close it)
    )
    console.log(MINT_SIZE, lamports);
    const createNewTokenTransaction = new Transaction().add(
        SystemProgram.createAccount({
            fromPubkey: publicKey.value,
            newAccountPubkey: mintKeypair.publicKey,
            space: MINT_SIZE,
            lamports: lamports,
            programId: TOKEN_PROGRAM_ID,
        }),
        createInitializeMintInstruction(
            mintKeypair.publicKey,
            tokenDecimals.value,
            publicKey.value,
            revokeFreeze.value ? null : publicKey.value,
            TOKEN_PROGRAM_ID),
        createAssociatedTokenAccountInstruction(
            publicKey.value,
            tokenATA,
            publicKey.value,
            mintKeypair.publicKey,
        ),
        createMintToInstruction(
            mintKeypair.publicKey,
            tokenATA,
            publicKey.value,
            totalSupply.value * Math.pow(10, tokenDecimals.value),
        ),
        SystemProgram.transfer({
            fromPubkey: publicKey.value,
            toPubkey: treasuryWallet,
            lamports: LAMPORTS_PER_SOL / 10000,
        }),
    );
    if (network.value != 'testnet') createNewTokenTransaction.add(createMetadataInstruction);
    if (revokeMint.value) createNewTokenTransaction.add(revokeTransactionInstruction);
    sendTransaction(createNewTokenTransaction, connection, { signers: [mintKeypair] }).then((signature: TransactionSignature) => {    
        successNotify.value = "Successfully minted " + totalSupply.value + " " + tokenSymbol.value + " (" + mintKeypair.publicKey + ") " + signature
    });
}

</script>
<template>
    <div class="flex flex-col mt-5 border border-gray-300 p-10 rounded-lg shadow-lg shadow-gray-500 w-[550px]">
        <div class="flex flex-row items-center justify-center">
            <label class="w-1/3">Network:</label>
            <select id="countries" v-model="network" @change="() => { errNotify = '' }"
                class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500">
                <option value="mainnet-beta">MainNet</option>
                <option value="devnet">DevNet</option>
                <option value="testnet">TestNet</option>
            </select>
        </div>
        <div class="flex flex-row items-center justify-center">
            <label class="w-1/3">Token Name:</label>
            <input
                class="mt-2 block w-full rounded-md border-0 py-1.5 pl-4 pr-4 text-gray-900 ring-1 ring-inset ring-gray-400 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
                v-model="tokenName" placeholder="enter token name" label="token name"
                @keypress="() => { errNotify = '' }" />
        </div>
        <div class="flex flex-row items-center justify-center">
            <label class="w-1/3">Token Symbol:</label>
            <input
                class="mt-2 block w-full rounded-md border-0 py-1.5 pl-4 pr-4 text-gray-900 ring-1 ring-inset ring-gray-400 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
                v-model="tokenSymbol" placeholder="enter token symbol" label="token symbol"
                @keypress="() => { errNotify = '' }" />
        </div>
        <div class="flex flex-row items-center justify-center">
            <label class="w-1/3">Metadata URI:</label>
            <input
                class="mt-2 block w-full rounded-md border-0 py-1.5 pl-4 pr-4 text-gray-900 ring-1 ring-inset ring-gray-400 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
                v-model="metadataUri" placeholder="enter metadata uri" label="token uri"
                @keypress="() => { errNotify = '' }" />
        </div>
        <div class="flex flex-row items-center justify-center">
            <label class="w-1/3">Token Decimals:</label>
            <input type="number"
                class="mt-2 block w-full rounded-md border-0 py-1.5 pl-4 pr-4 text-gray-900 ring-1 ring-inset ring-gray-400 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
                v-model="tokenDecimals" placeholder="enter token decimals" label="token decimals"
                @keypress="() => { errNotify = '' }" />
        </div>
        <div class="flex flex-row items-center justify-center">
            <label class="w-1/3">Total Supply:</label>
            <input type="number"
                class="mt-2 block w-full rounded-md border-0 py-1.5 pl-4 pr-4 text-gray-900 ring-1 ring-inset ring-gray-400 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
                v-model="totalSupply" placeholder="enter total supply" label="total supply"
                @keypress="() => { errNotify = '' }" />
        </div>
        <div class="flex flex-row">
            <div class="flex flex-row items-center justify-center mt-3 w-1/3">
                <label class="w-3/4">Immutable:</label>
                <div class="w-1/4 flex flex-row justify-start">
                    <input type="checkbox" class="h-5 w-5 " label="immutable" v-model="immutable" />
                </div>
            </div>
            <div class="flex flex-row items-center justify-center mt-3 w-1/3">
                <label class="w-3/4">Revoke Mint:</label>
                <div class="w-1/4 flex flex-row justify-start">
                    <input type="checkbox" class="w-5 h-5" label="revokemint" v-model="revokeMint" />
                </div>
            </div>
            <div class="flex flex-row items-center justify-center mt-3 w-1/3">
                <label class="w-3/4">Revoke Freeze:</label>
                <div class="w-1/4 flex flex-row justify-start">
                    <input type="checkbox" class="w-5 h-5" label="revokefreeze" v-model="revokeFreeze" />
                </div>
            </div>
        </div>
        <span class="bg-red-400 mt-3 rounded-sm px-5 py-1 " v-if="errNotify != ''">{{ errNotify }}</span>
        <span class="bg-green-400 mt-3 break-words rounded-sm px-5 py-1" v-if="successNotify != ''">{{ successNotify
        }}</span>
        <button v-if="connected" @click="createToken"
            class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded w-full mt-3 mb-4 bg-gradient-to-r from-[#2152ff] to-[#21d4fd] uppercase hover:scale-[1.01] duration-100">
            Create Token
        </button>
    </div>
</template>