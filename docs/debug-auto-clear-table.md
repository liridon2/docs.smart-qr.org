# Debug: Auto-Clear Table nach Stripe-Zahlung

## Problem
Tisch wird nach erfolgreicher Stripe-Zahlung nicht automatisch freigegeben.

## Erwartetes Verhalten
Nach erfolgreicher Online-Zahlung:
1. Alle Orders am Tisch werden als `paid` markiert
2. System prüft: Gibt es noch unpaid Orders?
3. Wenn NEIN → Orders werden gelöscht, Tisch ist frei

## Debug-Workflow

### 1. Backend neustarten (wichtig!)
```bash
# Terminal php: Ctrl+C, dann:
cd /home/dev/smartqr-v2/backend
php -S localhost:8000 -t public
```

### 2. Logs überwachen
```bash
# In neuem Terminal:
cd /home/dev/smartqr-v2
./debug-auto-clear.sh
```

### 3. DB-Status VOR Zahlung prüfen
```bash
mysql -u smartqr_user -pwibrug-rExhix-9vuwxi smartqr_dev < debug-tables.sql
```

Notiere:
- `table_id` deiner Test-Tischs
- Anzahl Orders
- Status der Orders (sollten 'preparing' oder 'completed' sein)

### 4. Zahlung durchführen
- Im Browser: Stripe-Zahlung abschließen
- **Sofort** zu den Logs schauen!

### 5. Erwartete Log-Sequenz

#### Erfolgreiche Auto-Clear:
```
confirmPaymentIntent: Marked 2 orders paid for PI=pi_xxx (tenant_id=1)
autoClearTableIfFullyPaid: START - tenant_id=1, table_id_meta=1, PI=pi_xxx
autoClearTableIfFullyPaid: Resolved table_id=1
autoClearTableIfFullyPaid: Found 0 unpaid orders for table_id=1
autoClearTableIfFullyPaid: Total orders to delete: [{"id":14,"status":"paid"},{"id":15,"status":"paid"}]
autoClearTableIfFullyPaid: Deleted 8 order items
autoClearTableIfFullyPaid: Deleted 2 orders
autoClearTableIfFullyPaid: SUCCESS - cleared table_id=1 for tenant_id=1
```

#### Problem-Fall 1: table_id nicht gefunden
```
autoClearTableIfFullyPaid: START - tenant_id=1, table_id_meta=null, PI=pi_xxx
autoClearTableIfFullyPaid: ABORT - tableId is null
```
**Ursache:** `table_id` fehlt in PaymentIntent Metadata

#### Problem-Fall 2: Noch unpaid Orders
```
autoClearTableIfFullyPaid: Found 1 unpaid orders for table_id=1
autoClearTableIfFullyPaid: ABORT - Still has 1 unpaid orders
```
**Ursache:** Nicht alle Orders wurden als paid markiert

#### Problem-Fall 3: Funktion wird gar nicht aufgerufen
```
confirmPaymentIntent: Marked 2 orders paid for PI=pi_xxx (tenant_id=1)
[NICHTS MEHR]
```
**Ursache:** Exception oder früher Return vor autoClearTableIfFullyPaid()

### 6. DB-Status NACH Zahlung prüfen
```bash
mysql -u smartqr_user -pwibrug-rExhix-9vuwxi smartqr_dev < debug-tables.sql
```

**Erwartetes Ergebnis:**
- Keine Orders mehr für den Tisch
- `order_count = 0`

**Wenn Orders noch da sind:**
- Prüfe `status` → Sollten alle 'paid' sein
- Prüfe Logs → Warum wurde nicht gelöscht?

## Mögliche Ursachen

### 1. Metadata fehlt table_id
**Prüfen:**
```sql
SELECT stripe_payment_intent_id, table_id 
FROM orders 
WHERE stripe_payment_intent_id IS NOT NULL 
ORDER BY id DESC LIMIT 5;
```

**Fix:** Stelle sicher, dass in `createPaymentIntent()` die Metadata korrekt gesetzt wird:
```php
'metadata' => [
    'table_id' => (int)$table['id'],  // ← Muss gesetzt sein!
    'table_uuid' => $tableUuid,
    'order_ids' => implode(',', $orderIds),
    'tenant_id' => $tenantId,
]
```

### 2. Nicht alle Orders als paid markiert
**Prüfen:**
```sql
SELECT id, status, stripe_payment_intent_id 
FROM orders 
WHERE table_id = 1 AND tenant_id = 1;
```

Wenn eine Order NOT 'paid':
- Warum wurde sie übersprungen?
- Hat sie eine andere `stripe_payment_intent_id`?
- Hat sie einen anderen Status-Filter?

### 3. Transaktion/Lock-Problem
Vielleicht gibt es ein DB-Lock oder die Transaction wird nicht committed.

**Test:** Manuell SQL ausführen:
```sql
SELECT COUNT(*) 
FROM orders
WHERE tenant_id = 1
  AND table_id = 1
  AND status NOT IN ('paid','cancelled');
```

Sollte 0 zurückgeben wenn alle paid sind.

### 4. Race Condition
Wenn sowohl `confirmPaymentIntent` UND Webhook aufgerufen werden:
- Erster Call markiert Orders als paid → deleted sie
- Zweiter Call findet keine Orders mehr → macht nichts

**Prüfen in Logs:** Wird autoClearTableIfFullyPaid zweimal aufgerufen?

## Quick Fix Test

Falls du manuell testen willst, ob der Delete funktioniert:

```sql
-- Simuliere die Auto-Clear-Logik
SET @table_id = 1;
SET @tenant_id = 1;

-- Prüfe unpaid orders
SELECT COUNT(*) as unpaid
FROM orders
WHERE tenant_id = @tenant_id
  AND table_id = @table_id
  AND status NOT IN ('paid','cancelled');

-- Wenn 0, dann lösche:
DELETE oi FROM order_items oi
INNER JOIN orders o ON oi.order_id = o.id
WHERE o.table_id = @table_id AND o.tenant_id = @tenant_id;

DELETE FROM orders
WHERE table_id = @table_id AND tenant_id = @tenant_id;
```

## Testing Checklist

- [ ] Backend neugestartet?
- [ ] Logs werden geschrieben? (check `/home/dev/smartqr-v2/logs/dev-YYYY-MM-DD.log`)
- [ ] Zwei Orders am gleichen Tisch erstellt?
- [ ] PaymentIntent erstellt → `table_id` in Metadata?
- [ ] Zahlung erfolgreich?
- [ ] Log zeigt "autoClearTableIfFullyPaid: START"?
- [ ] Log zeigt "Found 0 unpaid orders"?
- [ ] Log zeigt "Deleted X orders"?
- [ ] DB-Check: Orders gelöscht?
- [ ] Frontend: Tisch zeigt als frei?

## Nächste Schritte

1. **Backend neustarten**
2. **Debug-Script starten:** `./debug-auto-clear.sh`
3. **Zahlung durchführen**
4. **Logs analysieren**
5. **Ergebnis hier posten** - dann kann ich gezielt helfen!

Erwartete Info von dir:
- Komplette Log-Ausgabe von autoClearTableIfFullyPaid
- DB-Output von debug-tables.sql VOR und NACH Zahlung
- Funktioniert Auto-Clear oder nicht?
