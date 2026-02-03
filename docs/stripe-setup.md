# Stripe Connect Integration - Setup Anleitung

## ✅ Implementierte Features

### Backend
- ✅ Stripe PHP SDK installiert und konfiguriert
- ✅ Database Migration für Stripe Connect Accounts
- ✅ StripeController mit Account Management Endpoints
- ✅ Payment Intent Creation mit Destination Charges
- ✅ Webhook Handler für payment_intent.succeeded und account.updated
- ✅ Router-Integration für alle Stripe Endpoints

### Frontend
- ✅ Stripe.js und React Stripe Elements installiert
- ✅ StripePaymentForm Component für Guest Checkout
- ✅ CheckoutModal Integration mit Stripe Payment
- ✅ Admin Stripe Settings Page für Restaurant Onboarding

## 🔧 Setup-Schritte

### 1. Stripe Account erstellen
1. Gehe zu [https://stripe.com](https://stripe.com)
2. Erstelle einen Stripe Account
3. Aktiviere **Stripe Connect** in deinem Dashboard

### 2. API Keys konfigurieren

#### Backend (`/backend/config/.env.php`)
Die Stripe-Konfiguration ist bereits im Array vorhanden:
```php
'stripe' => [
    'secret_key' => env('STRIPE_SECRET_KEY', ''),
    'public_key' => env('STRIPE_PUBLIC_KEY', ''),
    'webhook_secret' => env('STRIPE_WEBHOOK_SECRET', ''),
],
```

Du kannst entweder:
1. **Environment Variables** setzen (empfohlen für Production):
   ```bash
   export STRIPE_SECRET_KEY=sk_test_...
   export STRIPE_PUBLIC_KEY=pk_test_...
   export STRIPE_WEBHOOK_SECRET=whsec_...
   ```

2. **Oder Defaults direkt in `.env.php` eintragen** (für Development):
   ```php
   'stripe' => [
       'secret_key' => env('STRIPE_SECRET_KEY', 'sk_test_YOUR_KEY_HERE'),
       'public_key' => env('STRIPE_PUBLIC_KEY', 'pk_test_YOUR_KEY_HERE'),
       'webhook_secret' => env('STRIPE_WEBHOOK_SECRET', 'whsec_YOUR_SECRET_HERE'),
   ],
   ```

#### Frontend (`/frontend/.env`)
```bash
# In frontend/.env
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
```

### 3. Database Migration ausführen
```bash
cd backend
sudo mysql -u root smartqr_dev < migrations/20251208__stripe_connect_accounts.sql
```

### 4. Webhook Endpoint registrieren

#### Lokale Entwicklung mit Stripe CLI:
```bash
# Stripe CLI installieren
brew install stripe/stripe-cli/stripe  # macOS
# oder von https://stripe.com/docs/stripe-cli

# Login (wichtig: auf den richtigen Account einloggen!)
stripe login

# ⚠️ WICHTIG: Prüfe, ob du auf dem richtigen Stripe Account eingeloggt bist
# Der Account muss mit deinem STRIPE_SECRET_KEY übereinstimmen
stripe config --list

# Falls falscher Account: Neu einloggen
# stripe logout
# stripe login

# Webhook forwarding starten
stripe listen --forward-to localhost:8000/webhooks/stripe

# Secret Key kopieren (whsec_...) und in .env eintragen
```

**Häufiger Fehler:** Wenn Webhooks nicht ankommen, prüfe ob die Stripe CLI auf dem gleichen Account eingeloggt ist wie dein `STRIPE_SECRET_KEY`!

#### Production:
1. Gehe zu Stripe Dashboard → Developers → Webhooks
2. Füge Endpoint hinzu: `https://your-domain.com/webhooks/stripe`
3. Wähle folgende Events:
   - `payment_intent.succeeded`
   - `payment_intent.payment_failed`
   - `account.updated`
4. Kopiere den Webhook Secret

### 5. Router-Konfiguration prüfen

In `backend/public/index.php` sollten folgende Routes existieren:

```php
// Stripe Connect - Admin Endpoints
if ($method === 'POST' && $uri === '/api/' . $tenant . '/admin/stripe/create-account') {
    $stripeController = new StripeController();
    $stripeController->createConnectedAccount();
    exit;
}

if ($method === 'POST' && $uri === '/api/' . $tenant . '/admin/stripe/onboarding-link') {
    $stripeController = new StripeController();
    $stripeController->generateOnboardingLink();
    exit;
}

if ($method === 'GET' && $uri === '/api/' . $tenant . '/admin/stripe/account-status') {
    $stripeController = new StripeController();
    $stripeController->getAccountStatus();
    exit;
}

// Guest Payment Endpoint
if ($method === 'POST' && $uri === '/api/' . $tenant . '/stripe/create-payment-intent') {
    $stripeController = new StripeController();
    $stripeController->createPaymentIntent();
    exit;
}
```

## 🧪 Testing Flow

### 1. Restaurant Onboarding
1. Als Admin einloggen
2. Zu Stripe Settings navigieren (`/admin/stripe-settings`)
3. "Jetzt Stripe-Konto erstellen" klicken
4. E-Mail eingeben
5. Stripe Onboarding durchlaufen
6. Bankverbindung hinterlegen

### 2. Guest Payment
1. Als Gast QR-Code scannen
2. Bestellung aufgeben
3. Checkout öffnen
4. "Online bezahlen" wählen
5. Kreditkartendaten eingeben (Test-Karten siehe unten)
6. Zahlung abschließen

### 3. Test Kreditkarten (Stripe Test Mode)
```
Erfolgreiche Zahlung:
4242 4242 4242 4242
CVC: beliebig (3 Ziffern)
Datum: beliebiges zukünftiges Datum

3D Secure Test:
4000 0027 6000 3184

Abgelehnte Karte:
4000 0000 0000 0002

Mehr: https://stripe.com/docs/testing
```

## 📊 Webhook Events Überwachung

### Stripe CLI (Development):
```bash
stripe listen --forward-to localhost:8000/webhooks/stripe
```

### Logs anschauen:
```bash
# Backend Logs
tail -f backend/logs/error.log

# Webhook Events in Stripe Dashboard:
# Dashboard → Developers → Webhooks → [Your Endpoint] → Events
```

## 💰 Gebührenstruktur

- **Application Fee**: 10% des Bestellbetrags
- **Stripe Processing Fee**: ~1,4% + 0,25€ pro Transaktion
- **Auszahlung**: Automatisch an Restaurant nach Abzug der Gebühren

## 🔒 Sicherheit

- ✅ Webhook Signature Verification implementiert
- ✅ HTTPS erforderlich für Production
- ✅ API Keys über Environment Variables
- ✅ PCI Compliance durch Stripe Elements

## 📝 API Endpoints

### Admin Endpoints (Authentifizierung erforderlich):
- `POST /api/{tenant}/admin/stripe/create-account` - Create Stripe Connect Account
- `POST /api/{tenant}/admin/stripe/onboarding-link` - Generate Onboarding URL
- `GET /api/{tenant}/admin/stripe/account-status` - Get Account Status

### Guest Endpoints:
- `POST /api/{tenant}/stripe/create-payment-intent` - Create Payment Intent

### Webhooks:
- `POST /webhooks/stripe` - Handle Stripe Events

## 🚀 Deployment Checklist

- [ ] Stripe Keys in Production .env eingetragen
- [ ] Webhook Endpoint in Stripe Dashboard registriert
- [ ] Database Migration ausgeführt
- [ ] HTTPS aktiviert
- [ ] Router-Konfiguration geprüft
- [ ] Test-Zahlung durchgeführt
- [ ] Webhook Events kommen an
- [ ] Auszahlungen funktionieren

## 🐛 Troubleshooting

### Problem: Webhooks kommen nicht an
- Prüfe Webhook Secret in .env
- Prüfe URL in Stripe Dashboard
- Teste mit Stripe CLI: `stripe trigger payment_intent.succeeded`

### Problem: Payment Intent Creation schlägt fehl
- Prüfe ob Restaurant Stripe Account hat
- Prüfe ob `charges_enabled = true`
- Prüfe Logs in `backend/logs/error.log`

### Problem: Onboarding Link funktioniert nicht
- Prüfe `APP_URL` in backend .env
- Prüfe refresh_url und return_url in StripeController

## 📚 Dokumentation

- [Stripe Connect Docs](https://stripe.com/docs/connect)
- [Destination Charges](https://stripe.com/docs/connect/destination-charges)
- [Stripe Elements](https://stripe.com/docs/stripe-js)
- [Webhook Events](https://stripe.com/docs/api/events/types)

## ✉️ Support

Bei Fragen oder Problemen:
1. Stripe Dashboard Logs prüfen
2. Backend Error Logs anschauen
3. Stripe Support kontaktieren
