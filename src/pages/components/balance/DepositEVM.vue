<template>
  <q-dialog
    v-model="showDlg"
    :full-height="false"
    transition-show="slide-up"
    transition-hide="slide-down"
  >
    <div class="popupCard">
      <div class="exitBtn">
        <q-btn
          round
          flat
          dense
          v-close-popup
          class="text-grey-6"
          icon="close"
        />
      </div>
      <div class="popupHeading">
        <div />
        <div class="text-h5 text-weight-medium text-center q-mt-lg">
          {{$t('components.evm_deposit')}}
        </div>
        <div />
      </div>
      <div class="text-center">
        <div class="text-subtitle2 text-grey-4 text-center">
          {{$t('components.deposit_1')}}<br />{{$t('components.deposit_2')}}
        </div>
        <div class="text-center q-mt-md">
          <div class="inputAmount row items-center">
            <input
              type="text"
              class="col text-weight-regular text-right no-border no-outline transparent text-white"
              v-model="depositAmount"
              @focus="
                depositAmount = depositAmount === '0' ? '' : depositAmount
              "
              @blur="inputBlur"
            />
            <label class="text-weight-medium q-ml-sm text-left"> TLOS </label>
          </div>
          <div
            @click="depositAmount = nativeTLOSBalance"
            class="depositAddressToggle"
          >
          {{$t('components.max')}}: {{ nativeTLOSBalance }}
          </div>
        </div>
        <div class="depositInput row justify-center">
          <q-input
            v-model="recipientAddress"
            label="Recipient (Metamask Address)"
            rounded
            outlined
            class="q-pt-md col-11"
            standout="bg-transparent text-white"
            label-color="white"
            color="white"
            input-class="text-white"
            @input="checkRecipientExist"
            @focus="checkRecipientExist"
            :rules="[
              (val) =>
                (val.slice(0, 2) === '0x' && val.length === 42) ||
                'Invalid address',
            ]"
          />
        </div>
        <div class="row justify-center">
          <q-btn
            :disabled="noDepositInputAmount || !recipientAddress"
            class="purpleGradient depositBtn"
            no-caps
            rounded
            label="Deposit"
            @click="deposit"
          />
        </div>
        <div class="row justify-center" v-if="!recipientAddress && !evmAddress">
          <q-btn
            class="purpleGradient depositBtn"
            no-caps
            rounded
            label="Generate Linked EVM address"
            @click="generateAddress"
          />
        </div>
        <div class="row justify-center">
          <div
            class="lightBlue depositAddressToggle q-mt-xs"
            @click="addEvmNetwork"
          >
          {{$t('components.add_evm_network')}}
          </div>
        </div>
        <div class="row justify-center q-mt-md">
          <div v-if="!evmAddress && depositOwnAddress" class="note">
            {{$t('components.first_deposit')}}
          </div>
          <div
            v-if="
              !recipientAddressExists &&
              !depositOwnAddress &&
              recipientAddress != ''
            "
            class="note"
          >
            {{$t('components.address_not_exist')}}
          </div>

          <q-card-section class="q-pt-sm text-warning">
            {{$t('components.dont_send_to_exchanges')}}
          </q-card-section>
        </div>
      </div>
    </div>
  </q-dialog>
</template>

<script>
import { mapGetters, mapActions } from "vuex";
import BigNumber from "bignumber.js";

export default {
  props: ["showDepositEVMDlg", "nativeTLOSBalance"],
  data() {
    return {
      depositAmount: "0",
      depositOwnAddress: false,
      recipientAddress: "",
      recipientAddressExists: true,
    };
  },
  computed: {
    ...mapGetters("account", ["isAuthenticated", "accountName", "evmAddress"]),
    showDlg: {
      get() {
        return this.showDepositEVMDlg;
      },
      set(value) {
        this.$emit("update:showDepositEVMDlg", value);
      },
    },
    noDepositInputAmount(){
      return (parseFloat(this.depositAmount) > 0) ? null: true;
    }
  },
  methods: {
    ...mapActions("account", [
      "setEvmState"
    ]),
    addEvmNetwork() {
      let params = [];
      if (this.chainName !== "telos") {
        params = [
          {
            chainId: "0x29",
            chainName: "Telos EVM Testnet",
            nativeCurrency: {
              name: "Telos",
              symbol: "TLOS",
              decimals: 4,
            },
            rpcUrls: ["https://testnet.telos.net/evm"],
            blockExplorerUrls: ["https://testnet.teloscan.io"],
          },
        ];
      } else {
        params = [
          {
            chainId: "0x28",
            chainName: "Telos EVM Mainnet",
            nativeCurrency: {
              name: "Telos",
              symbol: "TLOS",
              decimals: 4,
            },
            rpcUrls: ["https://mainnet.telos.net/evm"],
            blockExplorerUrls: ["https://teloscan.io"],
          },
        ];
      }

      window.ethereum
        .request({ method: "wallet_addEthereumChain", params })
        .then(() => console.log("Success"))
        .catch((error) => console.log("Error", error.message));
    },
    inputBlur() {
      if (isNaN(this.depositAmount)) this.depositAmount = "0";
      else this.depositAmount = Number(this.depositAmount).toString();
    },
    async checkRecipientExist() {
      try {
        _ = await this.$store.$evmApi.telos.getEthAccount(
          this.recipientAddress.toLowerCase()
        );
        this.recipientAddressExists = true;
      } catch (error) {
        this.recipientAddressExists = false;
      }
    },
    async generateAddress(){
      const actions = [];
      if (!this.evmAddress) {
          actions.push({
            account: "eosio.evm",
            name: "create",
            data: {
              account: this.accountName.toLowerCase(),
              data: "create",
            },
          });
        }
      try {
        const transaction = await this.$store.$api.signTransaction(
          actions,
          this.$t('components.create_evm_for', {account: this.accountName})
        );
        await this.setEvmState();
        this.$q.notify({
          type: "primary",
          message: this.$t('components.created_evm_for', {account: this.accountName}),
        });
        this.depositAmount = "0";
        this.depositOwnAddress = false;
        this.recipientAddress = this.evmAddress;
        this.recipientAddressExists = true;

      } catch (error) {
        this.$errorNotification(error);
      }
    },
    async deposit() {
      let amount = parseFloat(this.depositAmount);
      if (amount > parseFloat(this.nativeTLOSBalance)) {
        this.$q.notify({
          type: "negative",
          message: this.$t('components.cant_deposit_more', {balance: this.nativeTLOSBalance}),
        });
        return;
      }

      let quantityStr = `${amount.toFixed(4)} TLOS`;
      let actions = [];
      let memo = "";
      memo = this.recipientAddress.toLowerCase();
      await this.checkRecipientExist();
      if (!this.recipientAddressExists) {
        actions.push({
          account: process.env.EVM_CONTRACT,
          name: "openwallet",
          data: {
            account: this.accountName.toLowerCase(),
            address: this.recipientAddress.slice(2),
          },
        });
      }

      actions.push({
        account: "eosio.token",
        name: "transfer",
        data: {
          from: this.accountName.toLowerCase(),
          to: "eosio.evm",
          quantity: quantityStr,
          memo: memo,
        },
      });

      try {
        const transaction = await this.$store.$api.signTransaction(
          actions,
          this.$t('components.deposit_to_evm', {quantity: quantityStr})
        );

        await this.setEvmState();

        this.depositAmount = "0";
        this.depositOwnAddress = false;
        this.recipientAddress = "";
        this.recipientAddressExists = true;
        this.showDlg = false;

        this.$q.notify({
          type: "primary",
          message: this.$t('components.deposited_to_evm', {quantity: quantityStr}),
        });
      } catch (error) {
        this.$errorNotification(error);
      }
    },
  },
  watch: {
    showDepositEVMDlg() {
      if (this.showDlg) {
        this.$emit("addEvmNetwork");
      };
      if (this.evmAddress){
        this.recipientAddress = this.evmAddress
      }
    },
  },
};
</script>

<style lang="scss" scoped>
.depositAddressToggle {
  font-size: .8rem;
  cursor: pointer;
}
.lightBlue {
  color: $lightBlue;
}
.note {
  max-width: 25rem;
}
.depositBtn {
  flex-basis: 15rem;
  height: 3rem;
  margin-top: 1rem;
  margin-bottom: 1rem;
}

.exitBtn {
  position: absolute;
}

.q-input{
  font-size: .75rem;
}
</style>
