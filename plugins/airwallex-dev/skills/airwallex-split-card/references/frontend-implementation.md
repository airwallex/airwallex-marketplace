# Frontend Implementation

## Table of Contents

- [Step 1: Install and Initialize](#step-1-install-and-initialize)
- [Step 2: HTML Structure](#step-2-html-structure)
- [Step 3: Create and Mount Split Card Elements](#step-3-create-and-mount-split-card-elements)
- [Step 4: Render Saved Card List](#step-4-render-saved-card-list)
- [Step 5: Billing Information](#step-5-billing-information)
- [Step 6: Confirm Payment (Core Logic)](#step-6-confirm-payment-core-logic)
- [Full Page Initialization Flow](#full-page-initialization-flow)

---

## Step 1: Install and Initialize

```bash
npm install @airwallex/components-sdk
```

```javascript
import { init, createElement } from '@airwallex/components-sdk';

// Initialize Airwallex SDK
await init({
  env: 'demo', // 'prod' for production
  enabledElements: ['payments'],
});
```

> **Note on amounts**: Airwallex takes the PaymentIntent `amount` in major currency
> units (ISO 4217), so $49.99 is `49.99`, not `4999`. It is not cents-based, and
> multiplying by 100 overcharges the customer a hundredfold. Zero-decimal currencies
> like JPY and KRW have no minor unit, so send the whole number as-is (`1000` for ¥1000).
> Work the amount out from your server-side order data, not from whatever the browser
> sends. The backend side is covered in
> [Payments integration notes for AI agents](https://www.airwallex.com/docs/developer-tools/ai-agent-payments-integration.md).
>
> ```javascript
> // Send the amount in major units, e.g. $49.99:
> await fetch('/api/create-payment-intent', {
>   method: 'POST',
>   headers: { 'Content-Type': 'application/json' },
>   body: JSON.stringify({ amount: 49.99, currency: 'USD' }), // 49.99, not 4999
> });
> ```

## Step 2: HTML Structure

```html
<!-- Saved cards section (shown when user has saved cards) -->
<div id="saved-cards-section" style="display: none;">
  <h3>Select a saved card</h3>
  <div id="saved-cards-list"></div>
  <div id="saved-card-cvc-section" style="display: none;">
    <label>Security code (CVC)</label>
    <div id="savedCardCvc"></div>
  </div>
  <hr>
  <label>
    <input type="radio" name="payment-method" value="new-card" id="use-new-card"> Use a new card
  </label>
</div>

<!-- New card input section -->
<div id="new-card-section">
  <div>
    <label>Cardholder name</label>
    <input id="cardholder-name" placeholder="Name on card" />
  </div>
  <div>
    <label>Card number</label>
    <div id="cardNumber"></div>
  </div>
  <div>
    <label>Expiry</label>
    <div id="expiry"></div>
  </div>
  <div>
    <label>CVC</label>
    <div id="cvc"></div>
  </div>
  <div id="save-card-section">
    <label>
      <input type="checkbox" id="save-card-checkbox"> Save this card for future payments
    </label>
  </div>
</div>

<!-- Billing Address section -->
<div id="billing-section">
  <h3>Billing address</h3>
  <label>
    <input type="checkbox" id="same-as-shipping" checked> Same as shipping address
  </label>
  <div id="billing-form" style="display: none;">
    <!-- Shown when "same as shipping" is unchecked -->
    <input id="billing-first-name" placeholder="First name" />
    <input id="billing-last-name" placeholder="Last name" />
    <input id="billing-email" placeholder="Email" />
    <input id="billing-phone" placeholder="Phone (optional)" />
    <input id="billing-street" placeholder="Street address" />
    <input id="billing-city" placeholder="City" />
    <input id="billing-state" placeholder="State / Province" />
    <input id="billing-postcode" placeholder="Postal code" />
    <select id="billing-country">
      <option value="US">United States</option>
      <option value="CN">China</option>
      <option value="AU">Australia</option>
      <!-- More countries... -->
    </select>
  </div>
</div>

<script>
  // Toggle billing form visibility
  document.getElementById('same-as-shipping').addEventListener('change', (e) => {
    document.getElementById('billing-form').style.display = e.target.checked ? 'none' : 'block';
  });
</script>

<!-- Submit button -->
<button id="submit">Confirm Payment</button>
```

## Step 3: Create and Mount Split Card Elements

```javascript
// ============================
// Global variables
// ============================
let cardNumber, expiry, cvc, savedCardCvc;
let selectedSavedCardId = null; // ID of user's selected saved card
let intentId;       // set by initPaymentPage()
let clientSecret;   // set by initPaymentPage()
let shippingAddress; // set from your order/checkout data before payment
let isLoggedIn;      // set from your auth logic (true if user has an account)

// ============================
// Initialize new card input elements
// ============================
function initNewCardElements() {
  // The intent goes in a nested `intent` object. `CardNumberElementOptions` has no
  // top-level intent_id / client_secret; those two names belong to confirm() instead,
  // and passing them here is silently ignored.
  cardNumber = createElement('cardNumber', {
    intent: { id: intentId, client_secret: clientSecret },
  });
  expiry = createElement('expiry');
  cvc = createElement('cvc');

  cardNumber.mount('cardNumber');
  expiry.mount('expiry');
  cvc.mount('cvc');
}

// ============================
// Initialize saved card CVC element
// ============================
function initSavedCardCvc(cvcLength = 3) {
  savedCardCvc = createElement('cvc', {
    cvcLength,            // 3 for Visa/MasterCard, 4 for AMEX
    isStandalone: true,   // improves UX when used independently for saved cards
  });
  savedCardCvc.mount('savedCardCvc');
}
```

> **Note**: A mounted Element instance stays bound to the container it was mounted into, and mounting the same instance again has no effect. The new-card CVC and the saved-card CVC are therefore two separate instances in two separate containers. To move one, destroy it and create a fresh instance (see Step 4).

## Step 4: Render Saved Card List

```javascript
/**
 * Render saved card list
 * @param {Array} savedCards - payment_methods list from backend
 */
function renderSavedCards(savedCards) {
  if (!savedCards || savedCards.length === 0) return;

  const container = document.getElementById('saved-cards-list');
  document.getElementById('saved-cards-section').style.display = 'block';

  savedCards.forEach((method, index) => {
    const card = method.card;
    const cvcLength = card.brand === 'amex' ? 4 : 3;

    const label = document.createElement('label');
    label.style.cssText = 'display: block; padding: 8px; cursor: pointer;';

    const radio = document.createElement('input');
    radio.type = 'radio';
    radio.name = 'payment-method';
    radio.value = method.id;
    radio.checked = index === 0;
    // Bind the handler directly. An inline onchange="..." attribute resolves the
    // function against `window`, which throws in a module or bundled build where
    // onSavedCardSelected is module-scoped rather than global.
    radio.addEventListener('change', () => onSavedCardSelected(method.id, cvcLength));

    // Appending strings sets text, so card values are never parsed as HTML.
    label.append(
      radio,
      ` ${card.brand.toUpperCase()} **** ${card.last4}  Exp ${card.expiry_month}/${card.expiry_year}`,
    );
    container.append(label);
  });

  // Select first card by default
  if (savedCards.length > 0) {
    onSavedCardSelected(savedCards[0].id, savedCards[0].card.brand === 'amex' ? 4 : 3);
  }

  // "Use new card" radio toggle
  document.getElementById('use-new-card').addEventListener('change', () => {
    onUseNewCard();
  });
}

/**
 * User selects a saved card
 */
function onSavedCardSelected(paymentMethodId, cvcLength) {
  selectedSavedCardId = paymentMethodId;

  // Hide new card section, show CVC
  document.getElementById('new-card-section').style.display = 'none';
  document.getElementById('saved-card-cvc-section').style.display = 'block';

  // Recreate rather than re-mount: a mounted instance stays bound to its first container
  if (savedCardCvc) {
    savedCardCvc.destroy();
    savedCardCvc = null;
  }
  initSavedCardCvc(cvcLength);
}

/**
 * User switches to "use new card"
 */
function onUseNewCard() {
  selectedSavedCardId = null;

  // Show new card section, hide saved card CVC
  document.getElementById('new-card-section').style.display = 'block';
  document.getElementById('saved-card-cvc-section').style.display = 'none';
}
```

## Step 5: Billing Information

Billing (billing address) is passed via the `payment_method` parameter in `confirm()`. The page typically offers two choices:
- **Same as shipping address** (checkbox checked, reuse existing address)
- **Enter new billing address** (show additional address form)

### Type Definitions (from `@airwallex/components-sdk`)

```typescript
// confirm() parameter type
interface PaymentMethodRequestData {
  client_secret: string;
  intent_id?: string;
  payment_method_id?: string;       // saved card ID (Scenario C)
  customer_id?: string;
  triggered_by?: 'customer';        // CIT subsequent payment with saved card
  payment_consent?: PaymentConsentOptions;
  payment_method?: PaymentMethodObjType;  // ← billing info goes here
  payment_method_options?: PaymentMethodOptionsType;
}

// payment_method object
interface PaymentMethodObjType {
  card: {
    name?: string;    // cardholder name
  };
  billing?: Billing;  // billing address info
}

// Billing information
interface Billing {
  first_name: string;
  last_name: string;
  email: string;                    // billing email
  phone_number?: string;
  date_of_birth?: string;
  address: Address;                 // billing address
}

// Address
interface Address {
  city: string;
  country_code: string;   // ISO 3166 two-letter country code, e.g. "US", "CN", "AU"
  postcode: string;
  state: string;
  street: string;
}
```

### Build Billing Object

```javascript
/**
 * Get billing information
 * @param {boolean} sameAsShipping - whether to reuse shipping address
 * @param {Object} shippingAddress - existing shipping address
 */
function getBillingInfo(sameAsShipping, shippingAddress) {
  if (sameAsShipping && shippingAddress) {
    // Reuse shipping address
    return {
      first_name: shippingAddress.first_name,
      last_name: shippingAddress.last_name,
      email: shippingAddress.email,
      phone_number: shippingAddress.phone_number,
      address: {
        city: shippingAddress.city,
        country_code: shippingAddress.country_code,
        postcode: shippingAddress.postcode,
        state: shippingAddress.state,
        street: shippingAddress.street,
      },
    };
  }

  // Get from billing form
  return {
    first_name: document.getElementById('billing-first-name').value,
    last_name: document.getElementById('billing-last-name').value,
    email: document.getElementById('billing-email').value,
    phone_number: document.getElementById('billing-phone').value,
    address: {
      city: document.getElementById('billing-city').value,
      country_code: document.getElementById('billing-country').value,
      postcode: document.getElementById('billing-postcode').value,
      state: document.getElementById('billing-state').value,
      street: document.getElementById('billing-street').value,
    },
  };
}
```

## Step 6: Confirm Payment (Core Logic)

```javascript
document.getElementById('submit').addEventListener('click', async () => {
  try {
    let response;

    // Build billing info
    const sameAsShipping = document.getElementById('same-as-shipping').checked;
    const billing = getBillingInfo(sameAsShipping, shippingAddress);

    if (selectedSavedCardId) {
      // ============================================
      // Scenario C: Pay with saved card (CIT subsequent)
      // ============================================
      response = await savedCardCvc.confirm({
        intent_id: intentId,
        client_secret: clientSecret,
        payment_method_id: selectedSavedCardId,
        triggered_by: 'customer',
      });

    } else {
      const saveCard = document.getElementById('save-card-checkbox').checked;

      if (saveCard) {
        // ============================================
        // Scenario B: New card + save card (CIT save)
        // ============================================
        response = await cardNumber.confirm({
          intent_id: intentId,
          client_secret: clientSecret,
          payment_consent: {
            next_triggered_by: 'customer',
          },
          payment_method: {
            card: {
              name: document.getElementById('cardholder-name').value,
            },
            billing,
          },
        });

      } else {
        // ============================================
        // Scenario A: New card one-time payment (Guest)
        // ============================================
        response = await cardNumber.confirm({
          intent_id: intentId,
          client_secret: clientSecret,
          payment_method: {
            card: {
              name: document.getElementById('cardholder-name').value,
            },
            billing,
          },
        });
      }
    }

    // Handle success
    console.log('Payment success:', response);
    handlePaymentSuccess(response);

  } catch (error) {
    // Handle failure — see error-handling.md
    console.error('Payment failed:', error);
    handlePaymentError(error);
  }
});
```

## Full Page Initialization Flow

```javascript
async function initPaymentPage() {
  // 1. Initialize Airwallex SDK
  await init({ env: 'demo', enabledElements: ['payments'] });

  // 2. Get intentId, clientSecret from backend (created with customer_id).
  //    The backend builds the PaymentIntent from trusted order data. Any amount
  //    the client sends is in major units per ISO 4217 (49.99 for $49.99), never
  //    cents. See the note on amounts under Step 1.
  ({ intentId, clientSecret } = await fetchPaymentIntent());

  // 3. Initialize new card input elements (always needed as fallback)
  initNewCardElements();

  // 4. If user is logged in, fetch saved card list
  if (isLoggedIn) {
    const savedCards = await fetchSavedCards(); // call backend API
    if (savedCards.length > 0) {
      // renderSavedCards() auto-selects the first card, which creates and mounts
      // the saved-card CVC instance. Do not mount it again here, or both paths
      // compete for the same container.
      renderSavedCards(savedCards);
    }
  }
}
```
