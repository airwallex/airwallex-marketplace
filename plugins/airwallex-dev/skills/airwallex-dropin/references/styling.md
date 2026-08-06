# Airwallex Drop-in Element: Styling, Saved Payment Methods & Full Init

> **Canonical source:** the styling and appearance content here distills [Customize style and appearance](https://www.airwallex.com/docs/payments/integration-options/web-checkout/customize-appearance). That page is authoritative. Prefer it if anything here drifts. Full link list in [References](#references).

> **Overview, frontend implementation & scenarios**: see [scenarios.md](scenarios.md)

## Table of Contents

- [Custom Styling](#custom-styling)
  - [Appearance](#appearance)
  - [CSS Rules](#css-rules)
  - [Layout](#layout)
  - [Billing Collection](#billing-collection)
  - [Payment Method Filtering](#payment-method-filtering)
  - [Apple Pay / Google Pay](#apple-pay--google-pay)
- [SavedPaymentMethod Options](#savedpaymentmethod-options)
- [Full Page Initialization Flow](#full-page-initialization-flow)
- [References](#references)

---

## Custom Styling

### Appearance

Drop-in supports `appearance` with mode, color variables, and CSS rules:

```javascript
const element = createElement('dropIn', {
  // ...required options
  appearance: {
    mode: 'light',   // 'light' or 'dark'
    variables: {
      colorBrand: '#612FFF',       // accent: buttons, links, focus borders
      colorText: '#14171A',        // primary text color
      colorBackground: '#FFFFFF',  // primary background color
    },
  },
});
```

For the default light and dark values of each variable, see [Customize style and appearance](https://www.airwallex.com/docs/payments/integration-options/web-checkout/customize-appearance). Each color auto-generates derived colors (secondary, disabled, hover, and so on).

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

### CSS Rules

Drop-in supports fine-grained CSS customization via `appearance.rules`:

```javascript
appearance: {
  mode: 'light',
  variables: { colorBrand: '#0066FF' },
  rules: {
    '.Button': {
      borderRadius: '8px',
      fontSize: '16px',
      fontWeight: '600',
    },
    '.Button:hover': {
      opacity: '0.9',
    },
    '.Input': {
      borderRadius: '6px',
      fontSize: '14px',
    },
    '.Input:hover': {
      borderColor: '#0066FF',
    },
  },
},
```

#### Supported CSS selectors

| Selector | Target |
|----------|--------|
| `.Button` | Submit/pay button |
| `.Button:hover` | Button hover state |
| `.Input` | All input fields |
| `.Input:hover` | Input hover state |
| `.Input:active` | Input active/focused state |
| `.GooglePayButton` | Google Pay button |
| `.GooglePayButton:hover` | Google Pay hover |
| `.ApplePayButton` | Apple Pay button |
| `.ApplePayButton:hover` | Apple Pay hover |

### Layout

```javascript
const element = createElement('dropIn', {
  // ...required options
  layout: {
    type: 'accordion',               // 'accordion' or 'tab'
    alwaysShowMethodLabel: true,      // show icons even with single method
  },
});
```

| Layout | Best for | Description |
|--------|----------|-------------|
| `accordion` | Desktop (default) | Stacked sections, expand to reveal payment form |
| `tab` | Mobile (default) | Tabbed navigation for compact screens |

### Billing Collection

Collect billing information to improve 3DS frictionless checkout rates:

```javascript
const element = createElement('dropIn', {
  // ...required options
  requiredBillingContactFields: ['name', 'email', 'address', 'phone'],
});
```

Or provide billing directly from your existing data:

```javascript
const element = createElement('dropIn', {
  // ...required options
  billing: {
    first_name: 'John',
    last_name: 'Doe',
    email: 'john@example.com',
    address: {
      city: 'San Francisco',
      country_code: 'US',
      postcode: '94107',
      state: 'CA',
      street: '123 Market St',
    },
  },
});
```

> **Note**: When passing `billing` directly, do NOT also set `requiredBillingContactFields`, the merchant-provided billing overrides input fields.

### Payment Method Filtering

Control which payment methods are shown and their order:

```javascript
const element = createElement('dropIn', {
  // ...required options
  methods: ['card', 'applepay', 'googlepay', 'wechatpay'],
  country_code: 'US',  // some methods are country-specific
});
```

> By default, all activated payment methods are shown. `applepay`, `googlepay`, `paypal` appear at top if supported.

### Apple Pay / Google Pay

```javascript
const element = createElement('dropIn', {
  // ...required options
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
const element = createElement('dropIn', {
  // ...required options
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

## Full Page Initialization Flow

```javascript
async function initPaymentPage(intentId, clientSecret, currency, options = {}) {
  // 1. Initialize SDK
  await init({ env: 'demo', enabledElements: ['payments'] });

  // 2. Build Drop-in options
  const dropInOptions = {
    intent_id: intentId,
    client_secret: clientSecret,
    currency,
    ...options,  // appearance, layout, payment_consent, etc.
  };

  // 3. Create and mount
  const element = createElement('dropIn', dropInOptions);
  element.mount('dropIn');

  // 4. Handle events
  element.on('ready', () => {
    document.getElementById('loading').style.display = 'none';
  });

  element.on('success', (e) => {
    const { intent } = e.detail;   // verify server-side before showing success
    window.location.href = '/payment-success?id=' + intent.id;
  });

  element.on('error', (e) => {
    showErrorMessage(e.detail.error.message);
  });

  return element;
}

// Usage examples:

// Guest checkout
initPaymentPage(intentId, clientSecret, 'USD');

// Save card (CIT) — just pass a PaymentIntent that has customer_id.
// Drop-in will render a "Save card" checkbox; no extra options needed.
initPaymentPage(intentId, clientSecret, 'USD');

// With styling
initPaymentPage(intentId, clientSecret, 'USD', {
  appearance: {
    mode: 'dark',
    variables: { colorBrand: '#0066FF' },
  },
  layout: { type: 'accordion' },
});
```

---

## References

- [Drop-in Element overview](https://www.airwallex.com/docs/payments/integration-options/web-checkout/drop-in-element)
- [Drop-in — Guest user checkout](https://www.airwallex.com/docs/payments/integration-options/web-checkout/drop-in-element/guest-user-checkout)
- [Drop-in — Save and reuse payment details](https://www.airwallex.com/docs/payments/integration-options/web-checkout/save-and-reuse-payment-details)
- [Test card numbers](https://www.airwallex.com/docs/payments/test-and-go-live/test-card-numbers)
```
