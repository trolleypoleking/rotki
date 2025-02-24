<template>
  <v-overlay opacity="1" color="grey lighten-4">
    <div class="account-management__loading" />
    <div v-if="!premiumVisible">
      <v-card class="account-management__card pb-6" width="500px" light>
        <div
          class="pt-6 pb-3 text-h2 font-weight-black white--text account-management__card__title"
        >
          <span class="px-6">{{ $t('app.name') }}</span>
          <span class="d-block mb-3 pl-6 text-caption">
            {{ $t('app.moto') }}
          </span>
        </div>
        <connection-loading v-if="!connectionFailure" :connected="connected" />
        <connection-failure v-else />
        <v-slide-y-transition>
          <login
            :loading="loading"
            :displayed="!accountCreation && connected"
            :sync-conflict="syncConflict"
            :errors="errors"
            @touched="errors = []"
            @login="login($event)"
            @backend-changed="backendChanged($event)"
            @new-account="accountCreation = true"
          />
        </v-slide-y-transition>
        <v-slide-y-transition>
          <create-account
            :loading="loading"
            :displayed="accountCreation && connected"
            :error="accountCreationError"
            @error:clear="accountCreationError = ''"
            @cancel="accountCreation = false"
            @confirm="createAccount($event)"
          />
        </v-slide-y-transition>
      </v-card>
      <privacy-notice />
      <div v-if="$interop.isPackaged" class="account-management__log-level">
        <log-level
          :value="loglevel"
          @input="startBackendWithLogLevel($event)"
        />
      </div>
      <div v-if="!$interop.isPackaged" class="account-management__about">
        <v-tooltip open-delay="400" top>
          <template #activator="{ on, attrs }">
            <v-btn
              icon
              color="primary"
              v-bind="attrs"
              @click="$emit('about')"
              v-on="on"
            >
              <v-icon>mdi-information</v-icon>
            </v-btn>
          </template>
          <span>{{ $t('account_management.about_tooltip') }}</span>
        </v-tooltip>
      </div>
    </div>

    <premium-reminder
      :display="displayPremium"
      @login-complete="loginComplete()"
      @upgrade="upgrade()"
    />
  </v-overlay>
</template>

<script lang="ts">
import { Component, Emit, Prop, Vue } from 'vue-property-decorator';
import { mapActions, mapGetters, mapMutations, mapState } from 'vuex';
import ConnectionFailure from '@/components/account-management/ConnectionFailure.vue';
import ConnectionLoading from '@/components/account-management/ConnectionLoading.vue';
import CreateAccount from '@/components/account-management/CreateAccount.vue';
import Login from '@/components/account-management/Login.vue';
import PremiumReminder from '@/components/account-management/PremiumReminder.vue';
import {
  deleteBackendUrl,
  getBackendUrl
} from '@/components/account-management/utils';
import LogLevel from '@/components/helper/LogLevel.vue';
import PrivacyNotice from '@/components/PrivacyNotice.vue';
import { SyncConflict } from '@/store/session/types';
import { ActionStatus, Message } from '@/store/types';
import { Credentials, UnlockPayload } from '@/typing/types';
import {
  CRITICAL,
  currentLogLevel,
  DEBUG,
  Level,
  LOG_LEVEL
} from '@/utils/log-level';

@Component({
  components: {
    ConnectionFailure,
    PrivacyNotice,
    ConnectionLoading,
    PremiumReminder,
    LogLevel,
    Login,
    CreateAccount
  },
  computed: {
    ...mapState(['connectionFailure']),
    ...mapState('session', ['syncConflict', 'premium']),
    ...mapState(['message', 'connected']),
    ...mapGetters(['updateNeeded', 'message'])
  },
  methods: {
    ...mapActions('session', ['unlock']),
    ...mapMutations(['setMessage'])
  }
})
export default class AccountManagement extends Vue {
  accountCreation: boolean = false;
  premium!: boolean;
  loading: boolean = false;
  message!: Message;
  connected!: boolean;
  syncConflict!: SyncConflict;
  unlock!: (payload: UnlockPayload) => Promise<ActionStatus>;
  setMessage!: (message: Message) => void;
  errors: string[] = [];
  accountCreationError: string = '';

  loglevel: Level = process.env.NODE_ENV === 'development' ? DEBUG : CRITICAL;

  premiumVisible = false;

  @Prop({ required: true, type: Boolean })
  logged!: boolean;
  connectionFailure!: boolean;

  get displayPremium(): boolean {
    return !this.premium && !this.message.title && this.premiumVisible;
  }

  async created() {
    this.loglevel = currentLogLevel();
    if (this.connected) {
      return;
    }

    const { sessionOnly, url } = getBackendUrl();
    if (!!url && !sessionOnly) {
      await this.backendChanged(url);
    } else {
      await this.startBackendWithLogLevel(this.loglevel);
    }
  }

  async mounted() {
    const { sessionOnly } = getBackendUrl();
    if (sessionOnly) {
      deleteBackendUrl();
      await this.startBackendWithLogLevel(this.loglevel);
    }
  }

  @Emit()
  loginComplete() {
    this.dismiss();
  }

  async backendChanged(url: string | null) {
    await this.$store.commit('setConnected', false);
    const { success } = await this.$store.dispatch('connect', url);
    if (!success) {
      this.setMessage({
        success: false,
        description: this.$t(
          'account_management.custom_backend.failed.description',
          { url }
        ).toString(),
        title: this.$t(
          'account_management.custom_backend.failed.title'
        ).toString()
      });
      deleteBackendUrl();
      await this.startBackendWithLogLevel(this.loglevel);
    }
  }

  async startBackendWithLogLevel(level: Level) {
    localStorage.setItem(LOG_LEVEL, level);
    await this.$store.commit('setConnected', false);
    await this.$interop.restartBackend(level);
    await this.$store.dispatch('connect');
  }

  async login(credentials: Credentials) {
    const { username, password, syncApproval } = credentials;
    this.loading = true;
    const { message } = await this.unlock({
      username,
      password,
      syncApproval
    });

    if (message) {
      this.errors = [message];
    }
    this.loading = false;
    if (this.logged) {
      this.showPremiumDialog();
      this.showGetPremiumButton();
    }
  }

  async createAccount(credentials: Credentials) {
    const {
      username,
      password,
      apiKey,
      apiSecret,
      submitUsageAnalytics
    } = credentials;
    this.loading = true;
    this.accountCreationError = '';
    const { message, success } = await this.unlock({
      username,
      password,
      create: true,
      syncApproval: 'unknown',
      apiKey,
      apiSecret,
      submitUsageAnalytics
    });
    this.loading = false;

    if (success) {
      this.accountCreation = false;

      if (this.logged) {
        this.showGetPremiumButton();
        this.loginComplete();
      }
    } else {
      this.accountCreationError =
        message ?? this.$t('account_management.creation.error').toString();
    }
  }

  upgrade() {
    this.$interop.navigateToPremium();
    this.loginComplete();
  }

  dismiss() {
    this.premiumVisible = false;
  }

  private showPremiumDialog() {
    if (this.premium) {
      this.loginComplete();
      return;
    }
    this.premiumVisible = true;
  }

  private showGetPremiumButton() {
    this.$interop.premiumUserLoggedIn(this.premium);
  }
}
</script>

<style scoped lang="scss">
@import '~@/scss/scroll';

@keyframes scrollLarge {
  0% {
    transform: rotate(-13deg) translateY(0px);
  }

  100% {
    transform: rotate(-13deg) translateY(-600px);
  }
}

.account-management {
  &__card {
    z-index: 5;
    max-height: 90vh;
    overflow: auto;

    &__title {
      background-color: var(--v-primary-base);
    }

    @extend .themed-scrollbar;
  }

  &__loading {
    position: absolute;
    height: calc(100% + 1100px);
    width: calc(100% + 900px);
    left: -450px !important;
    top: -250px !important;
    opacity: 0.5;
    background: url(~@/assets/images/rotkipattern2.svg) repeat;
    background-size: 450px 150px;
    filter: grayscale(0.5);
    -webkit-animation-name: scrollLarge;
    animation-name: scrollLarge;
    -webkit-animation-duration: 35s;
    animation-duration: 35s;
    -webkit-animation-timing-function: linear;
    animation-timing-function: linear;
    -webkit-animation-iteration-count: infinite;
    animation-iteration-count: infinite;
  }

  &__log-level,
  &__about {
    position: absolute !important;
    width: 56px !important;
    right: 12px !important;

    @media (min-width: 701px) {
      bottom: 12px !important;
    }

    @media (max-width: 700px) {
      top: 12px !important;
    }
  }
}

::v-deep {
  .v-overlay {
    &__content {
      display: flex;
      flex-direction: row;
      align-items: center;
      justify-content: center;
      height: 100%;
      width: 100%;
    }
  }
}
</style>
