# 🧪 Testing Guide - Stripe Integration

## 📋 Voraussetzungen

✅ API Keys in `/backend/config/.env.php` hinterlegt
✅ Frontend `.env` mit VITE_STRIPE_PUBLISHABLE_KEY konfiguriert
✅ Stripe CLI installiert

## 🚀 Schritt-für-Schritt Testing

### 1. Backend Server starten

```bash
cd /home/dev/smartqr-v2/backend
php -S localhost:8000 -t public/
```

**Erwartet:** Server läuft auf `http://localhost:8000`

---

### 2. Frontend Dev Server starten

**Neues Terminal öffnen:**

```bash
cd /home/dev/smartqr-v2/frontend
npm run dev
```

**Erwartet:** Frontend läuft auf `http://localhost:5173`

---

### 3. Stripe Webhook Forwarding starten

**Neues Terminal öffnen:**

```bash
cd /home/dev/smartqr-v2
stripe login
```

Folge den Anweisungen im Browser um dich bei Stripe einzuloggen.

Dann starte das Webhook Forwarding:

```bash
stripe listen --forward-to localhost:8000/webhooks/stripe
```

**WICHTIG:** Die CLI gibt dir einen **Webhook Signing Secret** aus:
```
> Ready! Your webhook signing secret is whsec_abc123...
```

Kopiere diesen Secret und füge ihn in `/backend/config/.env.php` ein:

```php
'stripe' => [
    'secret_key' => env('STRIPE_SECRET_KEY', 'sk_test_...'),
    'public_key' => env('STRIPE_PUBLIC_KEY', 'pk_test_...'),
    'webhook_secret' => env('STRIPE_WEBHOOK_SECRET', 'whsec_abc123...'), // <-- HIER
],
```

**Oder setze die Environment Variable:**
```bash
export STRIPE_WEBHOOK_SECRET=whsec_abc123...
```

---

### 4. Restaurant Onboarding testen

1. **Admin Login:**
   - Öffne `http://localhost:5173`
   - Logge dich als Admin ein

2. **Stripe Settings öffnen:**
   - Navigiere zu Settings/Einstellungen
   - Öffne "Stripe Connect Settings" oder ähnlich

3. **Connect Account erstellen:**
   - Klicke auf "Create Stripe Account"
   - Gib deine E-Mail ein
   - Klicke auf "Start Onboarding"

4. **Stripe Onboarding durchlaufen:**
   - Du wirst zu Stripe weitergeleitet
   - Fülle die Formulare aus (für Test kannst du Fake-Daten nutzen)
   - Bestätige das Onboarding

5. **Verify Status:**
   - Zurück in der Admin App
   - Status sollte zeigen: ✅ Charges Enabled, ✅ Payouts Enabled

---

### 5. Guest Payment Flow testen

1. **Order erstellen:**
   - Öffne Customer App (als Gast)
   - Scanne QR Code oder öffne `/c/{restaurant-slug}`
   - Füge Produkte zum Warenkorb hinzu
   - Gehe zu Checkout

2. **Payment durchführen:**
   - Wähle "Pay with Card" / "Mit Karte zahlen"
   - Verwende eine Test-Karte:
     - **Erfolg:** `4242 4242 4242 4242`
     - **Declined:** `4000 0000 0000 0002`
     - **3D Secure:** `4000 0025 0000 3155`
   - Ablaufdatum: Beliebiges zukünftiges Datum (z.B. `12/30`)
   - CVC: Beliebige 3 Ziffern (z.B. `123`)
   - Klicke "Pay"

3. **Verify Webhook:**
   - Im Terminal mit `stripe listen` solltest du sehen:
     ```
     [200] POST /webhooks/stripe [evt_abc123] payment_intent.succeeded
     ```

4. **Verify Database:**
   ```bash
   sudo mysql -u root smartqr_dev -e "
   SELECT id, status, stripe_payment_intent_id, stripe_charge_id, application_fee_amount 
   FROM orders 
   ORDER BY created_at DESC 
   LIMIT 5;"
   ```
   
   **Erwartet:** Status = 'paid', Stripe IDs ausgefüllt

5. **Verify in Stripe Dashboard:**
   - Öffne https://dashboard.stripe.com/test/payments
   - Suche nach deinem Payment Intent
   - Prüfe dass 10% Application Fee abgezogen wurde
   - Transfer zum Connected Account sollte sichtbar sein

---

## 🧪 Test Cases

### Test Case 1: Erfolgreicher Payment
- **Input:** Karte `4242 4242 4242 4242`
- **Expected:** Order Status = 'paid', Webhook empfangen, Stripe Dashboard zeigt Payment

### Test Case 2: Abgelehnter Payment
- **Input:** Karte `4000 0000 0000 0002`
- **Expected:** Error Message angezeigt, Order Status bleibt 'pending'

### Test Case 3: Restaurant ohne Stripe Account
- **Setup:** Tenant ohne stripe_account_id
- **Expected:** Payment Intent Creation schlägt fehl mit Error "Restaurant not connected to Stripe"

### Test Case 4: Restaurant Onboarding incomplete
- **Setup:** Tenant mit stripe_account_id aber charges_enabled = false
- **Expected:** Payment Intent Creation schlägt fehl mit Error "Restaurant onboarding not complete"

---

## 🐛 Debugging

### Webhook nicht empfangen?
```bash
# Prüfe Stripe CLI Output
# Zeigt es [200] oder [500]?

# Prüfe PHP Error Log
tail -f /var/log/php8.2-fpm.log  # oder wo auch immer deine PHP Logs sind

# Teste Webhook manuell
stripe trigger payment_intent.succeeded
```

### Payment schlägt fehl?
```bash
# Prüfe Browser Console (F12)
# Sollte keine CORS oder Network Errors zeigen

# Prüfe Stripe API Keys sind korrekt
stripe config --list

# Prüfe dass Frontend den richtigen Key hat
cat frontend/.env | grep VITE_STRIPE
```

### Database Update klappt nicht?
```bash
# Prüfe MySQL Error Log
sudo tail -f /var/log/mysql/error.log

# Teste Database Connection
php backend/test_db.php
```

---

## ✅ Success Checklist

- [ ] Backend Server läuft auf Port 8000
- [ ] Frontend läuft auf Port 5173
- [ ] Stripe CLI forwarding aktiv
- [ ] Webhook Secret in .env.php konfiguriert
- [ ] Restaurant Stripe Account erstellt
- [ ] Onboarding completed (charges_enabled = true)
- [ ] Test Payment erfolgreich
- [ ] Webhook empfangen (Status 200)
- [ ] Order Status = 'paid' in Database
- [ ] Payment sichtbar in Stripe Dashboard
- [ ] Application Fee (10%) korrekt abgezogen
- [ ] Transfer zum Restaurant sichtbar

---

## 🌐 Production Deployment

Für Production musst du den Webhook **manuell** in der Stripe Console registrieren:

1. Gehe zu https://dashboard.stripe.com/webhooks
2. Klicke "Add endpoint"
3. URL: `https://your-domain.com/webhooks/stripe`
4. Events auswählen:
   - `payment_intent.succeeded`
   - `payment_intent.payment_failed`
   - `account.updated`
5. Webhook Secret kopieren und in Production .env.php eintragen
6. **WICHTIG:** Verwende `sk_live_...` Keys statt `sk_test_...`

