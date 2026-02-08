# Subscription Cancellation Feature

## Übersicht

Dieses Feature ermöglicht es Restaurant-Besitzern, ihr Abonnement direkt über die Account Settings zu kündigen und zu reaktivieren. Die Kündigung erfolgt zum Ende der Abrechnungsperiode (`cancel_at_period_end`), nicht sofort.

**Status**: ✅ Implementiert (Februar 2026)

---

## Architektur

### Datenfluss

```
Frontend (User Action)
    ↓
Backend API (Cancel/Reactivate)
    ↓
Stripe API (Update Subscription)
    ↓
Immediate DB Sync (Don't wait for webhook)
    ↓
Stripe Webhook (customer.subscription.updated)
    ↓
Fallback DB Sync (Ensures consistency)
```

**Wichtig**: Die DB-Synchronisation erfolgt **sofort** nach dem Stripe API Call, nicht erst beim Webhook. Der Webhook dient als Fallback für Konsistenz.

---

## Database Schema

### Migration: `20260208__subscription_sync_fields.sql`

```sql
ALTER TABLE tenants
ADD COLUMN stripe_subscription_id VARCHAR(255) NULL AFTER stripe_customer_id,
ADD COLUMN subscription_cancel_at_period_end BOOLEAN NOT NULL DEFAULT FALSE AFTER subscription_plan,
ADD COLUMN subscription_current_period_end TIMESTAMP NULL AFTER subscription_cancel_at_period_end,
ADD COLUMN subscription_cancelled_at TIMESTAMP NULL AFTER subscription_current_period_end;
```

### Migration: `20260208__add_subscription_started_at.sql`

```sql
ALTER TABLE tenants
ADD COLUMN subscription_started_at TIMESTAMP NULL AFTER subscription_current_period_end;
```

### Felder

| Feld | Typ | Beschreibung |
|------|-----|--------------|
| `stripe_subscription_id` | VARCHAR(255) | Stripe Subscription ID |
| `subscription_cancel_at_period_end` | BOOLEAN | Ob Kündigung zum Periodenende vorgemerkt ist |
| `subscription_current_period_end` | TIMESTAMP | Ende der aktuellen Abrechnungsperiode |
| `subscription_started_at` | TIMESTAMP | Wann das Abo begonnen hat |
| `subscription_cancelled_at` | TIMESTAMP | Wann die Kündigung angefragt wurde |

---

## Backend Implementation

### AdminController.php

#### `getSubscription()`

Liest Subscription-Daten aus der DB (mit Fallback für ältere Schemas).

**Endpoint**: `GET /api/subscription`

**Response**:
```json
{
  "status": "active",
  "plan": "basic",
  "cancel_at_period_end": false,
  "current_period_end": "2026-03-08 17:42:14",
  "active_since": "2026-02-08 17:42:14",
  "cancelled_at": null
}
```

#### `cancelSubscription()`

Kündigt das Abonnement zum Ende der Periode.

**Endpoint**: `POST /api/subscription/cancel`

**Request Body**:
```json
{
  "reason": "Optional cancellation reason"
}
```

**Ablauf**:
1. Ruft Stripe API auf: `cancel_at_period_end = true`
2. Speichert Kündigungsgrund in Stripe Metadata
3. **Schreibt sofort in DB**:
   - `subscription_cancel_at_period_end = 1`
   - `subscription_current_period_end` von Stripe Response
4. Gibt Response mit Enddatum zurück

**Response**:
```json
{
  "success": true,
  "message": "Subscription will be cancelled at the end of the billing period.",
  "cancel_at_period_end": true,
  "current_period_end": "2026-03-08 17:42:14"
}
```

#### `reactivateSubscription()`

Entfernt die Kündigungs-Vormerkung.

**Endpoint**: `POST /api/subscription/reactivate`

**Ablauf**:
1. Prüft ob Subscription gekündigt ist
2. Ruft Stripe API auf: `cancel_at_period_end = false`
3. **Schreibt sofort in DB**:
   - `subscription_cancel_at_period_end = 0`
   - `subscription_cancelled_at = NULL`
   - `subscription_current_period_end` von Stripe Response
4. Abo verlängert sich automatisch am Periodenende

**Response**:
```json
{
  "success": true,
  "message": "Subscription reactivated successfully.",
  "cancel_at_period_end": false,
  "current_period_end": "2026-03-08 17:42:14"
}
```

### WebhookController.php

#### `syncTenantSubscriptionFromStripe()`

Synchronisiert **alle** Subscription-Felder von Stripe in die DB.

**Wird aufgerufen von**:
- `handleSubscriptionCreated()` - `customer.subscription.created`
- `handleSubscriptionUpdated()` - `customer.subscription.updated`
- `handleSubscriptionDeleted()` - `customer.subscription.deleted`

**Synchronisierte Felder**:
- `subscription_status`
- `stripe_subscription_id`
- `subscription_cancel_at_period_end`
- `subscription_current_period_end`
- `subscription_cancelled_at`

**Logic**:
```php
$cancelAtPeriodEnd = !empty($subscription['cancel_at_period_end']) ? 1 : 0;

$currentPeriodEnd = null;
if (!empty($subscription['current_period_end'])) {
    $currentPeriodEnd = date('Y-m-d H:i:s', (int)$subscription['current_period_end']);
}

$cancelledAt = null;
if ($status === 'cancelled' && empty($subscription['canceled_at'])) {
    $cancelledAt = date('Y-m-d H:i:s', time());
} elseif (!empty($subscription['canceled_at'])) {
    $cancelledAt = date('Y-m-d H:i:s', (int)$subscription['canceled_at']);
}
```

---

## Frontend Implementation

### AccountSettingsPage.tsx

#### TanStack Query Integration

```tsx
// Fetch subscription data
const { data: subscription, isLoading: subscriptionLoading } = useQuery({
  queryKey: ['admin', tenantSlug, 'subscription'],
  queryFn: () => client.getSubscription(),
  refetchOnMount: 'always', // Always fresh data
  staleTime: 0
});
```

#### Cancel Mutation

```tsx
const cancelMutation = useMutation({
  mutationFn: (reason: string) => client.cancelSubscription({ reason }),
  onSuccess: (response: any) => {
    queryClient.invalidateQueries({ queryKey: ['admin', tenantSlug, 'subscription'] });
    const endDate = formatDate(response?.current_period_end);
    toast.success(
      endDate 
        ? `Kündigung vorgemerkt bis ${endDate}.`
        : 'Kündigung vorgemerkt. Das Abo läuft noch bis zum Ende der aktuellen Abrechnungsperiode.',
    );
    setShowCancelDialog(false);
  }
});
```

#### Reactivate Mutation

```tsx
const reactivateMutation = useMutation({
  mutationFn: () => client.reactivateSubscription(),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['admin', tenantSlug, 'subscription'] });
    toast.success('Reaktivierung angefragt.');
  }
});
```

#### UI States

**Aktives Abo**:
- Status Badge: Grün "Aktiv"
- Button: "Abo kündigen" (rot)
- Zeigt nächste Abrechnung

**Gekündigtes Abo**:
- Status Badge: Gelb "Kündigt am [Datum]" oder "Zur Kündigung vorgemerkt"
- Button: "Abo reaktivieren" (blau)
- Zeigt Enddatum der aktuellen Periode

---

## Stripe Configuration

### Environment Variables

`.env` Datei:
```env
STRIPE_SECRET_KEY=sk_test_51SbzL6R686oeIoxb38...
STRIPE_PUBLIC_KEY=pk_test_51SbzL6R686oeIoxb5B...
STRIPE_WEBHOOK_SECRET=whsec_7bebdc816482b14024df...
```

**Wichtig**: Backend und Stripe CLI müssen denselben Account verwenden!

### Webhook Events

Folgende Events werden vom Backend verarbeitet:

| Event | Handler | Aktion |
|-------|---------|--------|
| `customer.subscription.created` | `handleSubscriptionCreated()` | Sync subscription data |
| `customer.subscription.updated` | `handleSubscriptionUpdated()` | Sync subscription data |
| `customer.subscription.deleted` | `handleSubscriptionDeleted()` | Mark as cancelled in DB |

### Webhook Setup

```bash
# Development
stripe listen --forward-to http://localhost:8000/webhooks/stripe

# Production
# Webhook URL in Stripe Dashboard konfigurieren
# URL: https://api.smart-qr.org/webhooks/stripe
```

---

## Testing Guide

### 1. Setup

```bash
# Terminal 1: PHP Backend
cd backend
php -S localhost:8000 -t public

# Terminal 2: Frontend
cd frontend
npm run dev

# Terminal 3: Stripe Webhooks
stripe listen --forward-to http://localhost:8000/webhooks/stripe
```

### 2. Test Cancellation

1. Login als Restaurant-Besitzer
2. Navigiere zu Account Settings
3. Klick "Abo kündigen"
4. Optional: Kündigungsgrund eingeben
5. Bestätigen

**Erwartetes Verhalten**:
- ✅ Toast: "Kündigung vorgemerkt bis [Datum]"
- ✅ Status Badge ändert sich zu gelb "Kündigt am [Datum]"
- ✅ Button ändert sich zu "Abo reaktivieren"
- ✅ Stripe Webhook `customer.subscription.updated` wird empfangen
- ✅ DB: `subscription_cancel_at_period_end = 1`

### 3. Test Reactivation

1. Bei gekündigtem Abo: Klick "Abo reaktivieren"
2. Bestätigen

**Erwartetes Verhalten**:
- ✅ Toast: "Reaktivierung angefragt"
- ✅ Status Badge ändert sich zu grün "Aktiv"
- ✅ Button ändert sich zu "Abo kündigen"
- ✅ Stripe Webhook `customer.subscription.updated` wird empfangen
- ✅ DB: `subscription_cancel_at_period_end = 0`, `subscription_cancelled_at = NULL`

### 4. Database Verification

```sql
-- Prüfe Subscription-Felder
SELECT 
    id, 
    email,
    subscription_status,
    subscription_cancel_at_period_end,
    subscription_current_period_end,
    subscription_started_at,
    subscription_cancelled_at
FROM tenants
WHERE email = 'test@example.com';
```

### 5. Stripe Verification

```bash
# Subscription in Stripe prüfen
stripe subscriptions retrieve sub_xxx

# Wichtige Felder:
# - cancel_at_period_end: true/false
# - current_period_end: Unix timestamp
# - status: active
```

---

## Known Issues & Future Improvements

### Current Issues

1. **Period End Date Display**
   - **Problem**: Frontend zeigt manchmal Fallback-Text "Zur Kündigung vorgemerkt" statt Datum
   - **Ursache**: React Query cached alte Response mit `current_period_end: null`
   - **Workaround**: Hard Reload (Cmd+Shift+R) oder `refetchOnMount: 'always'` gesetzt
   - **Status**: ⚠️ Beeinträchtigt User nicht kritisch

### Future Improvements

1. **Retention Workflow**
   - Dialog mit Discount-Angebot bei Kündigungsversuch
   - "Warum kündigst du?" Analyse im Admin Dashboard

2. **Email Notifications**
   - Bestätigungs-Email nach Kündigung
   - Reminder 7 Tage vor Abo-Ende
   - "Wir vermissen dich" Email nach Ablauf

3. **Downgrade Option**
   - Statt Kündigung: Downgrade zu günstigerem Plan
   - Prorated Billing

4. **Subscription History**
   - Verlauf aller Subscription-Änderungen
   - Audit Log für Compliance

---

## Troubleshooting

### "No active subscription found"

**Ursache**: `stripe_subscription_id` fehlt in DB oder stimmt nicht mit Stripe überein.

**Lösung**:
```sql
-- Subscription ID von Stripe holen
-- stripe subscriptions list --customer cus_xxx

-- In DB updaten
UPDATE tenants 
SET stripe_subscription_id = 'sub_xxx'
WHERE stripe_customer_id = 'cus_xxx';
```

### Webhook Events kommen nicht an

**Prüfen**:
```bash
# Stripe CLI läuft?
ps aux | grep stripe

# Webhook Secret korrekt?
stripe listen --print-secret

# Backend läuft auf Port 8000?
curl http://localhost:8000/
```

### Period End ist NULL

**Manuell synchronisieren**:
```sql
-- Subscription von Stripe abrufen und current_period_end setzen
UPDATE tenants
SET subscription_current_period_end = '2026-03-08 17:42:14'
WHERE stripe_customer_id = 'cus_xxx';
```

---

## API Reference

### GET /api/subscription

**Auth**: Required (JWT)

**Response**: 200 OK
```json
{
  "status": "active",
  "plan": "basic",
  "cancel_at_period_end": false,
  "current_period_end": "2026-03-08 17:42:14",
  "active_since": "2026-02-08 17:42:14",
  "cancelled_at": null
}
```

### POST /api/subscription/cancel

**Auth**: Required (JWT)

**Request**:
```json
{
  "reason": "Too expensive"
}
```

**Response**: 200 OK
```json
{
  "success": true,
  "message": "Subscription will be cancelled at the end of the billing period.",
  "cancel_at_period_end": true,
  "current_period_end": "2026-03-08 17:42:14"
}
```

**Errors**:
- `404 NO_ACTIVE_SUBSCRIPTION`: Keine aktive Subscription gefunden
- `404 NO_STRIPE_CUSTOMER`: Kein Stripe Customer verknüpft
- `500 STRIPE_ERROR`: Stripe API Fehler

### POST /api/subscription/reactivate

**Auth**: Required (JWT)

**Response**: 200 OK
```json
{
  "success": true,
  "message": "Subscription reactivated successfully.",
  "cancel_at_period_end": false,
  "current_period_end": "2026-03-08 17:42:14"
}
```

**Errors**:
- `400 NOT_CANCELLED`: Subscription ist nicht gekündigt
- `404 NO_SUBSCRIPTION`: Keine Subscription gefunden
- `500 STRIPE_ERROR`: Stripe API Fehler

---

## Deployment Checklist

- [ ] Migrations ausgeführt auf Prod DB
- [ ] Stripe Webhook URL in Dashboard konfiguriert
- [ ] Stripe Secret Keys in Prod `.env` gesetzt
- [ ] Backend deployed und läuft
- [ ] Frontend deployed
- [ ] Webhook Secret validiert
- [ ] Test-Kündigung durchgeführt
- [ ] Test-Reaktivierung durchgeführt
- [ ] Monitoring für Subscription Events aktiv

---

## Related Documentation

- [ONBOARDING_API.md](./ONBOARDING_API.md) - Initiales Subscription Setup
- [STRIPE_SETUP.md](./STRIPE_SETUP.md) - Stripe Integration Guide
- [DSGVO_ENCRYPTION.md](./DSGVO_ENCRYPTION.md) - Data Privacy

---

**Version**: 1.0  
**Last Updated**: 8. Februar 2026  
**Maintainer**: Development Team
