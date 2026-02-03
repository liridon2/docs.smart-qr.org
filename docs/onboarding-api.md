# Onboarding Integration Guide (Landing → SmartQR)

## 📋 Übersicht (aktueller Flow)

1. Landing sendet einen Onboarding-Request an das SmartQR-Backend.
2. Backend antwortet mit `onboarding_id` (32‑hex Zeichen).
3. Landing leitet zum Stripe Payment Link weiter und setzt:
   - `client_reference_id = onboarding_id`
   - `prefilled_email = registrationData.email`
4. Kein weiterer API‑Call von der Landing nach Stripe Checkout.
5. Backend erstellt den Tenant **ausschließlich** über das Stripe Webhook‑Event `checkout.session.completed`.

---

## ✅ Endpoint: Onboarding Request

```
POST https://v2.smart-qr.org/api/onboarding/request
```

### Headers
```
Content-Type: application/json
```

### Body
```json
{
  "restaurant_name": "Pizzeria Da Mario",
  "address": "Musterstraße 123, 12345 Stadt",
  "owner_name": "Max Mustermann",
  "email": "restaurant@example.com",
  "password": "SecurePass123"
}
```

### Response (201 Created)
```json
{
  "onboarding_id": "<32-hex-chars>"
}
```

### Validierung (Landing)
- `onboarding_id` muss 32‑hex sein (Regex: `/^[a-f0-9]{32}$/i`)

### Error Responses (Beispiele)
- `400 VALIDATION_ERROR`
- `409 EMAIL_EXISTS`
- `500 ERROR`

---

## 🔗 Stripe Payment Link (Landing)

Beispielhafte Weiterleitung (Pseudo):
```ts
const url = new URL(PAYMENT_LINK);
url.searchParams.set('client_reference_id', onboardingId);
url.searchParams.set('prefilled_email', registrationData.email);
window.location.href = url.toString();
```

Wichtig: Die Zuordnung **Onboarding ⇔ Stripe** erfolgt nur über
`client_reference_id=onboarding_id`.

---

## 🧾 Stripe Webhook (Backend)

```
POST https://v2.smart-qr.org/webhooks/stripe
```

### Benötigtes Event
- `checkout.session.completed`

Das Backend liest:
- `client_reference_id` → `onboarding_id`
- `customer` → `stripe_customer_id`

Dann wird der Tenant erstellt und die Onboarding‑Anfrage als bezahlt markiert.

---

## ⚠️ Optional/Legacy: Manuelles Tenant‑Onboarding

Der Endpoint `POST /api/onboarding/create-tenant` existiert weiterhin, ist aber
**nicht** Teil des aktuellen Landing‑Flows. Er kann für interne/Legacy‑Integrationen
verwendet werden (API‑Key erforderlich).
