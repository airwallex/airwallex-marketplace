# Error Handling & Payment Verification

## Error Handling

The `confirm()` promise rejects with an error object. Common error scenarios:

```javascript
function handlePaymentError(error) {
  const { code, message } = error;

  switch (code) {
    case 'validation_error':
      // Card number / expiry / CVC format invalid
      showFieldError(message);
      break;
    case 'card_declined':
      // Issuer declined the card (insufficient funds, fraud, etc.)
      showMessage('Payment declined. Please try a different card.');
      break;
    case 'expired_card':
      // Card has expired
      showMessage('This card has expired. Please use a different card.');
      break;
    case 'processor_declined':
      // Payment processor declined
      showMessage('Payment could not be processed. Please try again.');
      break;
    default:
      // Other errors (network, server, etc.)
      showMessage(message || 'An unexpected error occurred.');
  }
}
```

---

## Verify Payment Result (Webhook)

> **Critical**: Never rely solely on client-side `confirm()` response. Always verify payment status server-side.

1. **Webhook** (recommended): Listen for `payment_intent.succeeded` event on your server
2. **API polling**: Call `GET /api/v1/pa/payment_intents/{id}` to check status
3. **Dashboard**: Check Payment Activity in Airwallex web app

```
Client confirm() success → redirect to "processing" page → backend receives webhook → update order status
```

---

## References

- [Guest user checkout — Split Card Element](https://www.airwallex.com/docs/payments/integration-options/web-checkout/embedded-elements/split-card-element/guest-user-checkout)
- [Save and reuse payment details](https://www.airwallex.com/docs/payments/integration-options/web-checkout/save-and-reuse-payment-details)
- [Card Number Element API — confirm()](https://www.airwallex.com/docs/js/payments/card-number)
- [Test card numbers](https://www.airwallex.com/docs/payments/test-and-go-live/test-card-numbers)
