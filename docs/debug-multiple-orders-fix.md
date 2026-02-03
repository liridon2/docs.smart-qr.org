# Debug: Mehrere Orders werden nicht als "paid" markiert

## Problem
Nach erfolgreicher Stripe-Zahlung wird nur eine von mehreren Orders eines Tisches als `paid` markiert.

## Analyse

### Verdächtige Stellen

1. **WebhookController - Fehlender tenant_id Filter**
   - **Problem**: WHERE-Clause hatte keinen `tenant_id` Filter
   - **Risiko**: In Multi-Tenant-Setup könnten Orders anderer Tenants mit gleichen IDs fälschlicherweise geupdatet werden
   - **Fix**: `tenant_id` aus PaymentIntent Metadata extrahiert und in WHERE-Clause hinzugefügt

2. **Parameter-Reihenfolge**
   - Beide Controller verwenden `array_merge()` für SQL-Parameter
   - StripeController: `[$normalizedMethod, $paymentIntentId, $chargeId, $tenantId, $paymentIntentId, ...orderIds]`
   - WebhookController: `[$normalizedMethod, $chargeId, $transferId, $paymentIntentId, $tenantId?, ...orderIds]`

3. **Mögliche Race Condition**
   - `confirmPaymentIntent` und Webhook könnten beide aufgerufen werden
   - Erster Call markiert Orders → zweiter Call findet keine unpaid Orders mehr (wegen `status <> 'paid'`)
   - Das ist aber OK, beide sollten beim ersten Mal ALLE Orders markieren

## Implementierte Fixes

### 1. Umfassendes Debug-Logging

**StripeController::markOrdersPaidForIntent()**:
- Loggt tenant_id, PI-ID, orderIds vor Update
- Prüft und loggt Order-Status VORHER (id, status, stripe_payment_intent_id)
- Loggt finale SQL-Parameter
- Loggt Anzahl aktualisierter Rows
- Prüft und loggt Order-Status NACHHER (id, status, stripe_payment_intent_id, paid_at)

**StripeController::createPaymentIntent()**:
- Loggt welche Orders mit PI-ID verknüpft werden
- Loggt Anzahl aktualisierter Rows
- Verifiziert und loggt Orders nach Update

**WebhookController::handlePaymentIntentSucceeded()**:
- Extrahiert tenant_id aus PaymentIntent Metadata
- Loggt PI-ID, tenant_id, orderIds bei Webhook-Empfang
- Ruft markOrdersPaidForIntent mit tenant_id auf

**WebhookController::markOrdersPaidForIntent()**:
- Akzeptiert optionalen $tenantId Parameter
- Loggt alle Parameter und Orders vor/nach Update
- Fügt tenant_id zur WHERE-Clause hinzu wenn verfügbar

### 2. Security-Fix: tenant_id Filter

```php
// WebhookController - NEU:
if ($tenantId !== null) {
    $conditions[] = "tenant_id = ?";
    $params[] = $tenantId;
}
```

Verhindert versehentliches Update von Orders anderer Tenants.

## Debugging-Workflow

### Logs überwachen:
```bash
tail -f /home/dev/smartqr-v2/logs/dev-$(date +%Y-%m-%d).log | grep -E "(createPaymentIntent|confirmPaymentIntent|Webhook|markOrdersPaid)"
```

### Nach Test erwartete Log-Sequenz:

1. **PaymentIntent erstellen:**
   ```
   createPaymentIntent: Setting PI=pi_xxx on orders: [14,15]
   createPaymentIntent: Updated 2 orders with PI
   createPaymentIntent: Verified orders after update: [{"id":14,"stripe_payment_intent_id":"pi_xxx"},{"id":15,"stripe_payment_intent_id":"pi_xxx"}]
   ```

2. **Payment Confirmation (entweder via confirm oder webhook):**
   ```
   markOrdersPaidForIntent: tenant_id=1, PI=pi_xxx, orderIds=[14,15]
   markOrdersPaidForIntent: Orders BEFORE update: [{"id":14,"status":"completed","stripe_payment_intent_id":"pi_xxx"},{"id":15,"status":"completed","stripe_payment_intent_id":"pi_xxx"}]
   markOrdersPaidForIntent: SQL params: ["online","pi_xxx","ch_xxx",1,"pi_xxx",14,15]
   markOrdersPaidForIntent: Updated 2 rows
   markOrdersPaidForIntent: Orders AFTER update: [{"id":14,"status":"paid","stripe_payment_intent_id":"pi_xxx","paid_at":"2025-12-11 ..."},{"id":15,"status":"paid","stripe_payment_intent_id":"pi_xxx","paid_at":"2025-12-11 ..."}]
   ```

### Was zu suchen:

1. **Sind beide Orders mit PI-ID verknüpft?**
   - `createPaymentIntent: Verified orders` sollte beide Orders mit gleicher PI-ID zeigen

2. **Haben beide Orders den richtigen Status vor Update?**
   - `Orders BEFORE update` sollte beide als 'completed' oder 'preparing' zeigen (nicht 'paid')

3. **Werden wirklich 2 Rows aktualisiert?**
   - `Updated 2 rows` sollte erscheinen
   - Wenn nur `Updated 1 rows`, dann matcht eine Order die WHERE-Bedingungen nicht

4. **Sind beide Orders nach Update 'paid'?**
   - `Orders AFTER update` sollte beide mit status='paid' und paid_at zeigen

## Mögliche Ursachen wenn Problem weiterhin besteht

1. **Eine Order hat falschen Status:**
   - Z.B. schon 'paid' oder 'cancelled' → wird übersprungen wegen `status <> 'paid'`

2. **Eine Order hat andere stripe_payment_intent_id:**
   - Prüfen: `SELECT id, stripe_payment_intent_id FROM orders WHERE id IN (14, 15);`

3. **Eine Order hat andere tenant_id:**
   - Prüfen: `SELECT id, tenant_id FROM orders WHERE id IN (14, 15);`

4. **Race Condition:**
   - Webhook wird VOR confirmPaymentIntent aufgerufen
   - Webhook markiert Orders
   - confirmPaymentIntent findet keine unpaid Orders mehr
   - Sollte aber trotzdem funktionieren wenn Logs zeigen "Updated 2 rows" beim ersten Call

## Testing

### 1. DB-Status prüfen VORHER:
```sql
SELECT id, tenant_id, status, stripe_payment_intent_id, paid_at 
FROM orders 
WHERE table_id = (SELECT id FROM restaurant_tables WHERE qr_uuid = 'xxx')
  AND status NOT IN ('paid', 'cancelled');
```

### 2. PaymentIntent erstellen und bezahlen

### 3. DB-Status prüfen NACHHER:
```sql
SELECT id, tenant_id, status, stripe_payment_intent_id, paid_at 
FROM orders 
WHERE id IN (14, 15);
```

Beide sollten `status='paid'` und `paid_at` gesetzt haben.

### 4. Logs analysieren:
```bash
grep "markOrdersPaidForIntent" /home/dev/smartqr-v2/logs/dev-$(date +%Y-%m-%d).log
```

## Commit Message Template

```
Fix: Multiple orders not marked as paid after Stripe payment

Issues fixed:
1. Added comprehensive debug logging to track order updates
2. Added tenant_id filter to WebhookController for multi-tenant security
3. Added verification logging in createPaymentIntent to ensure PI-ID is set on all orders

Changes:
- backend/src/Controllers/StripeController.php:
  - markOrdersPaidForIntent(): Added before/after state logging
  - createPaymentIntent(): Added verification after PI-ID update
  
- backend/src/Controllers/WebhookController.php:
  - handlePaymentIntentSucceeded(): Extract tenant_id from metadata
  - markOrdersPaidForIntent(): Add optional tenant_id parameter and WHERE filter
  - Added comprehensive logging for debugging

This will help identify why only one order is marked as paid instead of all orders.
```
