# Cardholder & card create templates (CLI)

Copy-this-then-fill payloads for the CLI surface. Replace every `<...>` with real values from the user's request and the row you are about to create. **Always** dry-run the exact payload first, show/confirm it, then execute the same payload with `--confirm`. For batches, do this row-by-row — a prior sample/template preview is not approval for later rows.

**On the MCP surface, do not use these payloads.** Trust the tool manifest's input schema and pass parameters at the surface the tool expects (e.g., the card-create body is **flat** — `limit_amount`, `limit_currency`, `limit_interval`, `program_purpose` at the top level, not nested under `authorization_controls` or `program`).

---

## Cardholders

### INDIVIDUAL — named person

```json
{
  "request_id": "<generate via uuidgen>",
  "email": "<cardholder_email>",
  "type": "INDIVIDUAL",
  "individual": {
    "name": {"first_name": "<first_name>", "last_name": "<last_name>"},
    "date_of_birth": "<YYYY-MM-DD>",
    "address": {
      "line1": "<street_address>",
      "city": "<city>",
      "postcode": "<postcode>",
      "country": "<country_code_2_letter>"
    },
    "express_consent_obtained": "yes"
  }
}
```

### DELEGATE — purpose card, minimal fields

```json
{
  "request_id": "<generate via uuidgen>",
  "email": "<team_or_purpose_email>",
  "type": "DELEGATE"
}
```

---

## Cards

### Virtual card

```json
{
  "request_id": "<generate via uuidgen>",
  "cardholder_id": "<cdh_id>",
  "form_factor": "VIRTUAL",
  "created_by": "<requesting_persons_full_name>",
  "is_personalized": false,
  "nick_name": "<card_purpose>",
  "authorization_controls": {
    "allowed_transaction_count": "MULTIPLE",
    "transaction_limits": {
      "currency": "<currency>",
      "limits": [{"amount": "<amount>", "interval": "<MONTHLY_or_other>"}]
    }
  },
  "program": {"purpose": "COMMERCIAL"}
}
```

### Physical card

Same as virtual, plus `postal_address`, `form_factor: "PHYSICAL"`, `is_personalized: true`.

```json
{
  "request_id": "<generate via uuidgen>",
  "cardholder_id": "<cdh_id>",
  "form_factor": "PHYSICAL",
  "created_by": "<requesting_persons_full_name>",
  "is_personalized": true,
  "nick_name": "<card_purpose>",
  "authorization_controls": {
    "allowed_transaction_count": "MULTIPLE",
    "transaction_limits": {
      "currency": "<currency>",
      "limits": [{"amount": "<amount>", "interval": "<MONTHLY_or_other>"}]
    }
  },
  "program": {"purpose": "COMMERCIAL"},
  "postal_address": {
    "line1": "<street_address>",
    "city": "<city>",
    "state": "<state>",
    "postcode": "<postcode>",
    "country": "<country_code_2_letter>"
  }
}
```

For EXPRESS shipment (or any China destination), also pass `delivery_details` with `preferred_delivery_mode: "EXPRESS"` and an E.164 `mobile_number`. Both `postal_address` and `delivery_details` are create-time only — if either is wrong after creation, close the card and re-issue.
