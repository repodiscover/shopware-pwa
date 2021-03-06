<template>
  <div v-if="order" class="sw-order-details">
    <SfHeading
      class="sw-order-details__header full-width"
      :level="3"
      :title="`${$t('Order no.')} ${order.orderNumber}`"
    />
    <div class="sw-order-details__totals">
      <SfTable class="sf-table--bordered table">
        <SfTableHeading class="table__row">
          <SfTableHeader
            v-for="tableHeader in tableHeaders"
            :key="tableHeader"
            class="table__header"
            :class="{
              table__description: tableHeader === 'Item',
              table__quantity: tableHeader === 'Quantity',
              table__amount: tableHeader === 'Amount',
              table__price: tableHeader === 'Price',
            }"
            >{{ tableHeader }}</SfTableHeader
          >
        </SfTableHeading>
        <SwOrderDetailsItem
          v-for="item in order.lineItems"
          :key="item.id"
          :product="item"
        />
      </SfTable>
      <SwTotals :shipping="shippingCosts" :total="total" :subtotal="subtotal" />
    </div>
    <div class="sw-order-details__addresses">
      <SwPersonalDetails :personal-details="personalDetails" class="content" />
      <SwAddress
        v-if="billingAddress"
        :address="billingAddress"
        :address-title="$t('Billing address')"
        class="content"
      />

      <SwPluginSlot
        name="order-details-payment-method"
        :slot-context="paymentMethod"
      >
        <SwCheckoutMethod
          v-if="paymentMethod"
          :method="paymentMethod"
          :label="$t('Payment method')"
          class="content"
        />
      </SwPluginSlot>

      <SwPluginSlot
        name="order-details-shipping-method"
        :slot-context="shippingMethod"
      >
        <SwCheckoutMethod
          v-if="shippingMethod"
          :method="shippingMethod"
          :label="$t('Shipping method')"
          class="content"
        />
      </SwPluginSlot>
      <SfProperty :name="$t('Order status')" :value="status" />
      <SfLoader
        :loading="isPaymentButtonLoading"
        class="sw-order-details__loader"
      >
        <a v-if="paymentUrl" :href="paymentUrl">
          <SwButton
            class="sf-button sf-button--full-width pay-button color-danger"
          >
            {{ $t("Pay for your order") }}
          </SwButton>
        </a>
        <template #loader>{{ $t("Checking payment status...") }}</template>
      </SfLoader>
    </div>
  </div>
</template>

<script>
import { SfTable, SfProperty, SfHeading, SfLoader } from "@storefront-ui/vue"
import { useUser, getApplicationContext } from "@shopware-pwa/composables"
import { ref, onMounted, computed } from "@vue/composition-api"
import SwPluginSlot from "sw-plugins/SwPluginSlot"
import {
  getOrderPaymentMethodId,
  getOrderShippingMethodId,
} from "@shopware-pwa/helpers"
import {
  getShippingMethodDetails,
  getPaymentMethodDetails,
  getOrderPaymentUrl,
} from "@shopware-pwa/shopware-6-client"
import SwButton from "@/components/atoms/SwButton"
import { PAGE_ORDER_SUCCESS } from "@/helpers/pages"
import SwOrderDetailsItem from "@/components/SwOrderDetailsItem"
import SwPersonalDetails from "@/components/SwPersonalDetails"
import SwAddress from "@/components/SwAddress"
import SwCheckoutMethod from "@/components/SwCheckoutMethod"
import SwTotals from "@/components/SwTotals"

export default {
  name: "SwOrderDetails",
  components: {
    SfProperty,
    SfTable,
    SfHeading,
    SfLoader,
    SwButton,
    SwOrderDetailsItem,
    SwPersonalDetails,
    SwAddress,
    SwCheckoutMethod,
    SwTotals,
    SwPluginSlot,
  },
  props: {
    orderId: {
      type: String,
      default: "",
    },
  },
  data() {
    return {
      tableHeaders: [
        this.$t("Item"),
        this.$t("Price"),
        this.$t("Quantity"),
        this.$t("Amount"),
      ],
    }
  },
  // TODO: move this logic into separate service;
  // details: https://github.com/DivanteLtd/shopware-pwa/issues/781
  setup({ orderId }, { root }) {
    const { apiInstance } = getApplicationContext(root, "myComponent")
    const { getOrderDetails, loading, error: userError } = useUser(root)
    const order = ref(null)
    const paymentMethod = ref(null)
    const shippingMethod = ref(null)
    const paymentUrl = ref(null)
    const isPaymentButtonLoading = ref(false)

    const personalDetails = computed(
      () =>
        order.value && {
          email: order.value.orderCustomer.email,
          firstName: order.value.orderCustomer.firstName,
          lastName: order.value.orderCustomer.lastName,
        }
    )
    const billingAddress = computed(
      () =>
        order.value &&
        order.value.addresses &&
        order.value.addresses.find(
          ({ id }) => id == order.value.billingAddressId
        )
    )
    const shippingAddress = computed(
      () =>
        order.value &&
        order.value.addresses &&
        order.value.addresses.find(
          ({ id }) => id == order.value.shippingAddressId
        )
    )
    const paymentMethodId = computed(
      () => order.value && getOrderPaymentMethodId(order.value)
    )
    const shippingMethodId = computed(
      () => order.value && getOrderShippingMethodId(order.value)
    )
    const shippingCosts = computed(
      () => order.value && order.value.shippingCosts.totalPrice
    )
    const subtotal = computed(
      () => order.value && order.value.price.positionPrice
    )
    const total = computed(() => order.value && order.value.price.totalPrice)
    const status = computed(
      () => order.value && order.value.stateMachineState.name
    )

    onMounted(async () => {
      try {
        isPaymentButtonLoading.value = true
        order.value = await getOrderDetails(orderId)
        paymentMethod.value = await getPaymentMethodDetails(
          paymentMethodId.value,
          apiInstance
        )
        shippingMethod.value = await getShippingMethodDetails(
          shippingMethodId.value,
          apiInstance
        )
        const resp = await getOrderPaymentUrl(
          {
            orderId,
            finishUrl: `${window.location.origin}${PAGE_ORDER_SUCCESS}?orderId=${orderId}`,
          },
          apiInstance
        )
        paymentUrl.value = resp.paymentUrl
      } catch (e) {}
      isPaymentButtonLoading.value = false
    })

    return {
      isLoading: loading,
      userError,
      order,
      personalDetails,
      billingAddress,
      paymentMethod,
      shippingMethod,
      shippingCosts,
      subtotal,
      total,
      status,
      paymentUrl,
      isPaymentButtonLoading,
    }
  },
}
</script>

<style lang="scss" scoped>
@import "@/assets/scss/variables";

.sw-order-details {
  padding: 1rem;
  width: 60%;
  display: flex;
  flex-wrap: wrap;

  @include for-mobile {
    flex-direction: column;
    width: 90%;
    padding: var(--spacer-xs);
  }

  &__loader {
    margin-top: var(--spacer-base);
    height: 3rem;
  }

  &__header {
    text-align: left;
    width: 100%;
    margin: 0 0 var(--spacer-base) 0;
  }

  &__totals {
    flex: 4;

    @include for-desktop {
      flex: 2;
      margin-right: var(--spacer-base);
    }

    .sw-totals {
      margin-top: var(--spacer-base);
      @include for-desktop {
        margin-left: auto;
        width: 50%;
        padding: 0;
      }
    }
  }

  &__addresses {
    flex: 1;
    box-sizing: border-box;
    width: 100%;
    background-color: #f1f2f3;
    padding: var(--spacer-xl);
    margin-bottom: var(--spacer-base);
    &:last-child {
      margin-bottom: 0;
    }

    & > .content {
      margin-bottom: var(--spacer-base);
    }
  }

  .sf-divider {
    border: --c-primary;
    width: 50%;
    max-width: 50%;
    display: block;
  }
}

.table {
  &__row {
    flex-wrap: nowrap;

    & > th {
      order: unset;
    }
  }

  &__data {
    flex: 1;
    order: unset;
    text-align: center;
    &:last-of-type {
      text-align: right;
    }
  }

  &__description {
    flex: 2;
  }

  &__quantity {
    text-align: center;
    flex: 1;
  }

  &__price {
    flex: 1;
    order: unset;
  }

  &__amount {
    flex: 1;
    text-align: right;
  }
}

.pay-button {
  display: flex;
  margin-top: var(--spacer-base);
  margin-bottom: 0;
}
</style>
