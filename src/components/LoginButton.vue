<template>
  <div>
    <!-- Login Button -->
    <div v-if="!isAuthenticated" class="q-px-md">
      <div class="q-mt-md q-mb-sm">
        <q-btn
          @click="showLogin = true"
          no-caps
          label="Connect Wallet"
          class="purpleGradient q-pa-sm"
          rounded
        />
      </div>

      <!-- Signup Button -->
      <div class="q-mt-md">
        <q-btn
          @click="signUp"
          text-color="white"
          no-caps
          outline
          rounded
          label="Create New Account"
          class=" q-pa-sm"
        />
      </div>
    </div>
    <!-- Show Login -->
    <q-dialog v-model="showLogin">
      <div class="column showLoginPopup q-pa-lg popupCard">
        <div class="text-subtitle1">{{$t('login.connect_wallet')}}</div>
        <q-list class="" dark separator>
          <q-item
            class="q-my-sm"
            v-for="(wallet, idx) in $ual.authenticators"
            :key="wallet.getStyle().text"
            v-ripple
          >
            <q-item-section class="cursor-pointer" avatar @click="onLogin(idx)">
              <img :src="wallet.getStyle().icon" width="30" />
            </q-item-section>
            <q-item-section class="cursor-pointer" @click="onLogin(idx)">
              {{ wallet.getStyle().text }}
            </q-item-section>
            <q-item-section class="flex" avatar>
              <q-spinner
                v-if="loading === wallet.getStyle().text"
                :color="wallet.getStyle().textColor"
                size="2em"
              />
              <q-btn
                v-else
                :color="wallet.getStyle().textColor"
                icon="get_app"
                @click="openUrl(wallet.getOnboardingLink())"
                target="_blank"
                dense
                flat
                size="12px"
              >
                <q-tooltip>
                  {{$t('login.get_app')}}
                </q-tooltip>
              </q-btn>
            </q-item-section>
          </q-item>
        </q-list>

        <!-- Close Button -->
        <q-btn
          v-close-popup
          @click="close"
          size="md"
          no-caps
          rounded
          flat
          class="self-center flex-center"
          label="Close"
          :style="`display:flex;`"
        />
        <q-item
          v-if="error"
          :active="!!error"
          active-class="bg-red-1 text-grey-8"
        >
          <q-item-section>
            {{ error }}
          </q-item-section>
        </q-item>
      </div>
    </q-dialog>

    <!-- RAM low dialog -->
    <q-dialog persistent v-model="resLow">
      <q-card class="popupCard">
        <div class="popupHeading">
          <div>
            <q-btn
              round
              flat
              dense
              v-close-popup
              class="text-grey-6"
              icon="close"
            />
          </div>
          <div class="text-subtitle1 text-weight-medium text-center ">
            {{$t('login.resources_low')}}
          </div>
          <div />
        </div>
        <div class="text-center">
          <div class="q-pb-md">
            {{$t('login.recommend_bying')}}
          </div>
          <div class="">{{$t('login.proceed_q')}}</div>
          <div class="text-center q-gutter-x-sm q-pt-sm">
            <q-btn
              no-caps
              rounded
              class="purpleGradient"
              label="Deny"
              v-close-popup
            />
            <q-btn
              no-caps
              rounded
              label="Approve"
              class="purpleGradient"
              @click="buyResources()"
            />
          </div>
        </div>
      </q-card>
    </q-dialog>
  </div>
</template>

<script>
import { mapGetters, mapActions, mapMutations } from "vuex";

export default {
  data() {
    return {
      showLogin: false,
      showAuth: false,
      authType: "signin",
      error: null,
      authInterval: null,
      close: false,
      ramPrice: 0,
      ramAvail: 0,
      cpuAvail: 0,
      netAvail: 0,
      ramThres: 0.9,
      netThres: 0.9,
      cpuThres: 0.9,
      ramMinimum: 5000, //5 kB
      netMinimum: 10000,
      cpuMinimum: 50000,
      ramLow: false,
      netLow: false,
      cpuLow: false,
      resourcePoll: false,
      RAMtoBuy: 0,
      CPUtoBuy: 0,
      NETtoBuy: 0,
      buyAmount: 1, // 1 TLOS
      resLow: false
    };
  },
  computed: {
    ...mapGetters("account", [
      "isAuthenticated",
      "accountName",
      "loading",
      "isAutoLoading"
    ])
  },
  components: {
  },
  methods: {
    ...mapActions("account", [
      "login",
      "logout",
      "autoLogin",
      "getUserProfile",
      "setEvmState"
    ]),
    ...mapMutations("account", [
      "setAccountName",
      "getAccountProfile",
      "setLoadingWallet"
    ]),
    async signUp() {
      this.openUrl("https://app.telos.net/accounts/add");
    },
    async onLogin(idx) {
        const error = await this.login({ idx });
        if (!error) {
          this.showLogin = false;
          await this.$router.push({ path: "/balance" });
        } else {
          this.error = error;
        }
    },
    openUrl(url) {
      window.open(url);
    },
    goToAccountPage() {
      const accountPath = `/account/${this.accountName}`;
      if (this.$router.currentRoute.path !== accountPath) {
        this.$router.push({ path: accountPath });
      }
    },
    async createEvmApi() {
        try {
          await this.setEvmState()
        } catch (e) {
          console.log(e);
        }
    },

    async buyResources() {
      if (this.ramLow) {
        this.RAMtoBuy = this.buyAmount;
      }
      if (this.cpuLow) {
        this.CPUtoBuy = this.buyAmount;
      }
      if (this.netLow) {
        this.NETtoBuy = this.buyAmount;
      }

      let actions = [];
      if (this.ramLow) {
        actions.push({
          account: "eosio",
          name: "buyram",
          data: {
            payer: this.accountName.toLowerCase(),
            receiver: this.accountName.toLowerCase(),
            quant:
              String(parseFloat(this.RAMtoBuy).toFixed(4)) + String(" TLOS")
          }
        });
      }

      if (this.cpuLow || this.netLow) {
        actions.push({
          account: "eosio",
          name: "delegatebw",
          data: {
            from: this.accountName.toLowerCase(),
            receiver: this.accountName.toLowerCase(),
            stake_net_quantity:
              String(parseFloat(this.NETtoBuy).toFixed(4)) + String(" TLOS"),
            stake_cpu_quantity:
              String(parseFloat(this.CPUtoBuy).toFixed(4)) + String(" TLOS"),
            transfer: false
          }
        });
      }

      try {
        const transaction = await this.$store.$api.signTransaction(
          actions,
          this.$t('resources.buying_resources')
        );
        this.$q.notify({
          type: "primary",
          message: this.$t('resources.resources_bought')
        });
        if (transaction) this.ramLow = false;
        if (transaction) this.cpuLow = false;
        if (transaction) this.netLow = false;
        if (transaction) this.resLow = false;
        this.$emitter.emit("resources_bought");
      } catch (error) {
        this.$errorNotification(error);
      }
    },

    async getRamPrice() {
      const res = await this.$store.$api.getTableRows({
        code: "eosio",
        scope: "eosio",
        table: "rammarket",
        limit: 1,
        show_payer: false
      });
      let ramInfo = res.rows[0];
      this.ramPrice =
        ramInfo.quote.balance.split(" ")[0] /
        ramInfo.base.balance.split(" ")[0];
    },

    async checkResources() {
      await this.getRamPrice();
      let account = await this.$store.$api.getAccount(this.accountName);
      this.ramAvail = account.ram_quota - account.ram_usage;
      this.cpuAvail = account.cpu_limit.available;
      this.netAvail = account.net_limit.available;
      if (account.ram_usage / account.ram_quota > this.ramThres || this.ramAvail < this.ramMinimum)
        this.ramLow = true;
      if (1 - this.netAvail / account.net_limit.max > this.netThres || this.netAvail < this.netMinimum)
        this.netLow = true;
      if (1 - this.cpuAvail / account.cpu_limit.max > this.cpuThres || this.cpuAvail < this.cpuMinimum)
        this.cpuLow = true;
      this.resLow = this.ramLow || this.cpuLow || this.netLow;
    }
  },
  async mounted() {
    await this.autoLogin(this.$route.query.returnUrl);
    if (this.isAuthenticated) {
      await this.createEvmApi();
      await this.checkResources();
    }
    this.authInterval = setInterval(() => {
      if (this.$store.$account.needAuth) {
        this.$store.$account.needAuth = false;
        this.authType = "auth";
        this.showAuth = true;
      } else if (this.$store.$account.needConfirm) {
        this.$store.$account.needConfirm = false;
        this.authType = "confirm";
        this.showAuth = true;
      }
    }, 500);
  },
  beforeUnmount() {
    if (this.authInterval) clearInterval(this.authInterval);
  }
};
</script>

<style lang="scss" scoped>

.showLoginPopup {
  width: 30rem;
  height: auto;
  margin-bottom: 5rem;
}
</style>
