<template>
  <div id="o-order-confirmation">
    <div class="banner">
      <div class="banner__info">
        <SfHeading
          :title="OnlineOnly ? $t('It\'s ordered!') : $t('You are offline')"
          class="sf-heading--no-underline"
        />
        <p v-if="OnlineOnly && lastOrderConfirmation.orderNumber" class="banner__order-number">
          {{ $t('Order No.') }} <strong>{{ lastOrderConfirmation.orderNumber }}</strong>
        </p>
      </div>
    </div>
    <div class="wrapper">
      <template v-if="OnlineOnly">
        <SfHeading
          :title="$t('You\'ve successfully placed the order')"
          class="sf-heading--left"
        />
        <p class="paragraph">
          {{ $t('You can check status of your order by using our delivery status feature. You will receive an order confirmation e-mail with details of your order and a link to track its progress.') }}
        </p>
      </template>
      <template v-else>
        <template v-if="isNotificationSupported">
          <p v-if="isPermissionGranted" class="paragraph">
            <strong>{{ $t('You will receive Push notification after coming back online. You can confirm the order by clicking on it') }}</strong>
          </p>
          <template v-else>
            <p class="paragraph">
              {{ $t("You can allow us to remind you about the order via push notification after coming back online. You'll only need to click on it to confirm.") }}
            </p>
            <p class="paragraph">
              {{ $t(`Or if you will stay on "Order confirmation" page, the order will be placed automatically without confirmation, once the internet connection will be back.`) }}
            </p>
          </template>
        </template>
        <p v-else class="paragraph">
          {{ $t('To finish the order just come back to our store while online. Your order will be sent to the server as soon as you come back here while online and then confirmed regarding the stock quantities of selected items') }}
        </p>
      </template>
      <SfButton
        v-if="!isPermissionGranted && isNotificationSupported"
        class="wrapper__notifications-button"
        @click.native="requestNotificationPermission()"
      >
        {{ $t('Allow order notifications') }}
      </SfButton>
      <SfHeading
        :title="$t('What we can improve?')"
        class="sf-heading--left"
      />
      <p class="paragraph">
        {{ $t('Your feedback is important for us. Let us know what we could improve.') }}
      </p>
      <textarea
        class="feedback"
        v-model="feedback"
        :placeholder="$t('Type your opinion')"
      />
      <div class="wrapper__buttons">
        <SfButton
          class="color-secondary sf-button--full-width"
          @click="sendFeedback"
        >
          {{ $t('Send my feedback') }}
        </SfButton>
        <SfButton
          class="sf-button--outline sf-button--full-width"
          @click="$router.push(localizedRoute('/'))"
        >
          {{ $t('Back to shop') }}
        </SfButton>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState } from 'vuex';
import config from 'config';
import VueOfflineMixin from 'vue-offline/mixin';
import { EmailForm } from '@vue-storefront/core/modules/mailer/components/EmailForm';
import { isServer } from '@vue-storefront/core/helpers';
import { registerModule } from '@vue-storefront/core/lib/modules';
import { MailerModule } from '@vue-storefront/core/modules/mailer';
import { SfHeading, SfButton } from '@storefront-ui/vue';

export default {
  name: 'OOrderConfirmation',
  components: { SfHeading, SfButton },
  mixins: [VueOfflineMixin, EmailForm],
  data () {
    return {
      feedback: ''
    };
  },
  computed: {
    ...mapState({
      lastOrderConfirmation: state => state.order ? state.order.last_order_confirmation.confirmation : {},
      checkoutPersonalEmailAddress: state => state.checkout.personalDetails.emailAddress
    }),
    isNotificationSupported () {
      if (isServer || !('Notification' in window)) return false;
      return 'Notification' in window;
    },
    isPermissionGranted () {
      if (isServer || !('Notification' in window)) return false;
      return Notification.permission === 'granted';
    },
    mailerElements () {
      return config.mailer.contactAddress;
    }
  },
  beforeCreate () {
    registerModule(MailerModule);
  },
  destroyed () {
    this.$store.dispatch('checkout/setThankYouPage', false);
  },
  methods: {
    requestNotificationPermission () {
      if (isServer) return false;
      if ('Notification' in window && Notification.permission !== 'granted') {
        Notification.requestPermission();
      }
    },
    sendFeedback () {
      this.sendEmail(
        {
          sourceAddress: this.checkoutPersonalEmailAddress,
          targetAddress: this.mailerElements,
          subject: this.$t('What we can improve?'),
          emailText: this.feedback
        },
        this.onSuccess,
        this.onFailure
      );
    },
    onSuccess (message) {
      this.$store.dispatch('notification/spawnNotification', {
        type: 'success',
        message,
        action1: { label: this.$t('OK') }
      });
      if (this.mailerElements.sendConfirmation) {
        this.sendEmail({
          sourceAddress: this.mailerElements,
          targetAddress: this.checkoutPersonalEmailAddress,
          subject: this.$t('Confirmation of receival'),
          emailText: this.$t(`Dear customer,\n\nWe have received your letter.\nThank you for your feedback!`),
          confirmation: true
        });
      }
    },
    onFailure (message) {
      this.$store.dispatch('notification/spawnNotification', {
        type: 'error',
        message,
        action1: { label: this.$t('OK') }
      });
    }
  }
};
</script>

<style lang="scss" scoped>
@import "~@storefront-ui/shared/styles/helpers/breakpoints";

#o-order-confirmation {
  box-sizing: border-box;
  @include for-desktop {
    max-width: 1240px;
    margin: auto;
  }
}
.banner {
  background-color: #f1f2f3;
  &__info {
    padding: 2rem;
    text-align: left;
    @include for-desktop {
      text-align: center;
    }
    @include for-mobile {
      margin: 1rem;
    }
  }
  &__order-number {
    font-size: 0.875rem;
  }
}
.wrapper {
  max-width: 586px;
  margin: var(--spacer-extra-big) auto 0;
  padding: 0 var(--spacer-medium) 0 var(--spacer-medium);
  &__notifications-button {
    margin: var(--spacer-extra-big) 0 var(--spacer-extra-big) 0;
  }
  &__buttons {
    width: 100%;
    display: flex;
    margin: var(--spacer-extra-big) 0 var(--spacer-extra-big) 0;
    justify-content: space-between;
    @include for-desktop {
      & > button:not(:last-child) {
        margin: 0 var(--spacer-medium) 0 0;
      }
    }
    @include for-mobile {
      flex-direction: column;
      & > button {
        margin: 0 0 var(--spacer) 0;
      }
    }
  }
  @include for-desktop {
    padding: 0;
  }
}
.paragraph {
  line-height: 1.875rem;
  font-size: var(--font-size-big);
}
.feedback {
  box-sizing: border-box;
  border: 1px solid var(--c-light);
  width: 100%;
  height: 25vh;
  padding: 0.5em;
  font-family: var(--body-font-family-primary);
  resize: vertical;
}
</style>
