# Custom Styling

## Table of Contents

- [Overview](#overview)
- [Type Definitions](#type-definitions)
- [createElement Options](#createelement-options)
- [Styling Examples](#styling-examples)
- [Styling Tips](#styling-tips)

---

## Overview

Split Card Elements (cardNumber, expiry, cvc) render inside iframes, so you cannot style them with regular CSS. Instead, pass a `style` option via `createElement()`.

## Type Definitions

```typescript
// Supported pseudo-classes for split card elements
type PseudoClasses = ':hover' | ':focus' | ':autofill' | '::placeholder' | '::selection' | ':disabled';

type PseudoClassStyle = {
  [K in PseudoClasses]?: CSSProperties;
};

// Any valid CSS property (camelCase)
interface CSSProperties {
  [CSSPropertyName: string]: string | number | undefined;
}

// Style object for cardNumber / expiry / cvc elements
interface InputStyle {
  // Base styling — all other states extend from this
  base?: PseudoClassStyle & CSSProperties;
  // Applied when input passes validation
  valid?: CSSProperties;
  // Applied when input fails validation
  invalid?: CSSProperties;
  // 3DS popup dimensions
  popupWidth?: number;
  popupHeight?: number;
}
```

## createElement Options

Each split card element accepts these options:

```typescript
// CardNumber options
interface CardNumberElementOptions {
  disabled?: boolean;                       // default: false
  placeholder?: string;                     // e.g. '4242 4242 4242 4242'
  style?: InputStyle;
  autoCapture?: boolean;                    // default: true
  authorizationType?: 'final_auth' | 'pre_auth';  // default: 'final_auth'
  allowedCardNetworks?: CardNetwork[];      // filter accepted card brands
  authFormContainer?: string;               // container ID for 3DS auth form
  intent_id?: string;
  client_secret?: string;
}

// Expiry options
interface ExpiryDateElementOptions {
  disabled?: boolean;
  placeholder?: string;                     // e.g. 'MM / YY'
  style?: InputStyle;
}

// CVC options
interface CvcElementOptions {
  disabled?: boolean;
  placeholder?: string;                     // e.g. 'CVC'
  cvcLength?: number;                       // 3 (default) or 4 (AMEX)
  style?: InputStyle;
  authFormContainer?: string;
  isStandalone?: boolean;                   // true for saved card CVC
  isMasked?: boolean;                       // mask CVC input, default: false
}

// Supported card networks
type CardNetwork = 'visa' | 'mastercard' | 'maestro' | 'unionpay' | 'amex' | 'jcb' | 'diners' | 'discover';
```

## Styling Examples

### Basic brand-matching style

```javascript
const brandStyle = {
  base: {
    color: '#333333',
    fontSize: '16px',
    fontFamily: '"Helvetica Neue", Helvetica, Arial, sans-serif',
    fontWeight: '400',
    lineHeight: '24px',
    '::placeholder': {
      color: '#999999',
      fontWeight: '300',
    },
    ':focus': {
      color: '#111111',
    },
  },
  valid: {
    color: '#0a8a00',
  },
  invalid: {
    color: '#e63757',
  },
};

const cardNumber = createElement('cardNumber', {
  intent: { id: intentId, client_secret: clientSecret },
  placeholder: '1234 5678 9012 3456',
  style: brandStyle,
  allowedCardNetworks: ['visa', 'mastercard', 'amex'],
});

const expiry = createElement('expiry', {
  placeholder: 'MM / YY',
  style: brandStyle,
});

const cvc = createElement('cvc', {
  placeholder: 'CVC',
  style: brandStyle,
});
```

### Dark theme style

```javascript
const darkStyle = {
  base: {
    color: '#f5f6f7',
    fontSize: '16px',
    fontFamily: 'Inter, system-ui, sans-serif',
    backgroundColor: 'transparent',  // iframe bg — container bg set via your CSS
    '::placeholder': {
      color: '#6b7280',
    },
    ':focus': {
      color: '#ffffff',
    },
    ':autofill': {
      color: '#f5f6f7',
      backgroundColor: '#1f2937',
    },
  },
  valid: {
    color: '#34d399',
  },
  invalid: {
    color: '#f87171',
  },
};
```

### Restrict card networks

```javascript
// Only accept Visa and MasterCard
const cardNumber = createElement('cardNumber', {
  allowedCardNetworks: ['visa', 'mastercard'],
  style: brandStyle,
});
```

### Pre-authorization (hold without capture)

```javascript
const cardNumber = createElement('cardNumber', {
  autoCapture: false,               // do not auto-capture
  authorizationType: 'pre_auth',    // extend hold beyond 7 days (Visa/MC only)
  style: brandStyle,
});
```

### Standalone CVC for saved cards

```javascript
const savedCardCvc = createElement('cvc', {
  cvcLength: 3,
  isStandalone: true,    // improves UX when used independently for saved cards
  isMasked: true,        // mask input with dots
  placeholder: 'Enter CVC',
  style: brandStyle,
});
```

## Styling Tips

- **Container styling** (border, border-radius, padding, background) should be applied to the parent `<div>` via your own CSS, the iframe only controls the input text itself
- **Font loading**: custom web fonts must be passed via `init()` so the iframe can access them:
  ```javascript
  await init({
    env: 'demo',
    enabledElements: ['payments'],
    fonts: [
      { src: 'https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600', family: 'Inter' },
    ],
  });
  // Then use in style: base: { fontFamily: 'Inter, sans-serif' }
  ```
- All CSS properties use **camelCase** (e.g. `fontSize`, not `font-size`)
- The `base` style is inherited by all states; `valid`/`invalid`/pseudo-class styles override specific properties
- Use `::placeholder` for placeholder text color and style
- Use `:focus` to change style when the user clicks into the field
- `:autofill` controls browser autofill background color (important for dark themes)
