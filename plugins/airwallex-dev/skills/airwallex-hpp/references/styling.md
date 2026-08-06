# Airwallex Hosted Payment Page (HPP): Styling & Saved Payment Method Options

> **Overview, frontend implementation & scenarios**: see [scenarios.md](scenarios.md)

## Table of Contents

- [Custom Styling](#custom-styling)
  - [Appearance](#appearance)
  - [Layout](#layout)
  - [Logo](#logo)
  - [Billing Collection](#billing-collection)
  - [Payment Method Filtering](#payment-method-filtering)
  - [Apple Pay / Google Pay](#apple-pay--google-pay)
- [SavedPaymentMethod Options](#savedpaymentmethod-options)
- [References](#references)

---

## Custom Styling

### Appearance

HPP supports `appearance` with mode and color variables:

```javascript
payments.redirectToCheckout({
  intent_id: intentId,
  client_secret: clientSecret,
  currency: 'USD',
  appearance: {
    mode: 'light',   // 'light' or 'dark'
    variables: {
      colorBrand: '#612FFF',       // accent: buttons, links, focus borders
      colorText: '#14171A',        // primary text color
      colorBackground: '#FFFFFF',  // primary background color
    },
  },
  successUrl: 'https://yoursite.com/payment-success',
});
```

#### Default color values

| Variable | Light Mode | Dark Mode |
|----------|-----------|-----------|
| `colorBrand` | `#612fff` | `#ABA8FF` |
| `colorText` | `#14171a` | `#F5F6F7` |
| `colorBackground` | `#ffffff` | `#14171A` |

> Each color auto-generates a complete set of derived colors (secondary, disabled, hover, etc.)

#### Dark theme example

```javascript
appearance: {
  mode: 'dark',
  variables: {
    colorBrand: '#ABA8FF',
    colorText: '#F5F6F7',
    colorBackground: '#14171A',
  },
},
```

### Layout

```javascript
payments.redirectToCheckout({
  // ...other options
  layout: {
    type: 'accordion',   // 'accordion' (default desktop) or 'tab' (default mobile)
  },
});
```

| Layout | Best for | Description |
|--------|----------|--------------|
| `accordion` | Desktop | Stacked sections, expand to reveal payment form |
| `tab` | Mobile | Tabbed navigation for compact screens |

### Logo

Display your company logo on the HPP header:

```javascript
payments.redirectToCheckout({
  // ...other options
  logoUrl: 'https://yoursite.com/logo.png',
});
```

### Billing Collection

Collect billing information to improve 3DS frictionless checkout rates:

```javascript
payments.redirectToCheckout({
  // ...other options
  requiredBillingContactFields: ['name', 'email', 'address', 'phone'],
});
```

Available fields: `'name'`, `'email'`, `'country_code'`, `'address'`, `'phone'`

### Payment Method Filtering

Control which payment methods are shown and their order:

```javascript
payments.redirectToCheckout({
  // ...other options
  methods: ['card', 'applepay', 'googlepay', 'wechatpay'],
  country_code: 'US',  // some methods are country-specific
});
```

> By default, all activated payment methods are shown. `applepay`, `googlepay`, `paypal` appear at top if supported.

### Apple Pay / Google Pay

```javascript
payments.redirectToCheckout({
  // ...other options
  applePayRequestOptions: {
    countryCode: 'US',            // required
    buttonType: 'buy',
    buttonColor: 'white-outline',
  },
  googlePayRequestOptions: {
    countryCode: 'US',            // required
    merchantInfo: {
      merchantName: 'Your Store',
    },
    buttonType: 'buy',
  },
});
```

> Apple Pay and Google Pay require prior setup in Airwallex dashboard (Payments > Settings > domain registration).

---

## SavedPaymentMethod Options

Control how saved cards are displayed and stored:

```javascript
payments.redirectToCheckout({
  // ...other options
  savedPaymentMethod: {
    displayMode: 'auto',       // 'auto' (show if available) or 'never'
    saveMode: 'auto',          // see below
  },
  autoSaveCardForFuturePayments: true,  // default; the "save card" checkbox is pre-checked (set false to uncheck)
});
```

#### saveMode behavior (from the `@airwallex/components-sdk` `SaveMode` type)

| Value | Behavior |
|-------|----------|
| `'auto'` (default) | When `next_triggered_by` is **not set** and `customer_id` is present → shows checkbox, user chooses to save or not. When `next_triggered_by` is `'customer'` or `'merchant'` → **no checkbox**, card is automatically saved. |
| `'enable'` | Always store the card, no checkbox shown |
| `'disable'` | Never store (guest checkout) |
| `'collect_consent'` | Always show checkbox; only store if customer opts in. Use this to override the auto behavior when `next_triggered_by` is set |

---

## References

- [Hosted Payment Page overview](https://www.airwallex.com/docs/payments/online-payments/hosted-payment-page)
- [HPP — Guest user checkout](https://www.airwallex.com/docs/payments/online-payments/hosted-payment-page/guest-user-checkout)
- [HPP — Registered user checkout](https://www.airwallex.com/docs/payments/online-payments/hosted-payment-page/registered-user-checkout)
- [Test card numbers](https://www.airwallex.com/docs/payments/test-and-go-live/test-card-numbers)
