# 🚀 Quick Start - Stripe Testing

## Zugriff auf Stripe Onboarding

Die Stripe Settings sind jetzt in der Admin-Navigation verfügbar:

### URL
```
http://localhost:5173/a/{your-restaurant-slug}/stripe
```

### Navigation
1. Öffne Admin Panel: `http://localhost:5173/a/{slug}`
2. Klicke in der Sidebar auf **"💳 Zahlungen"**

---

## 📋 Komplette Test-Anleitung

### Schritt 1: Server starten

**Terminal 1 - Backend:**
```bash
cd /home/dev/smartqr-v2/backend
php -S localhost:8000 -t public/
```

**Terminal 2 - Frontend:**
```bash
cd /home/dev/smartqr-v2/frontend
npm run dev
```

**Terminal 3 - Stripe Webhook:**
```bash
cd /home/dev/smartqr-v2
stripe listen --forward-to localhost:8000/webhooks/stripe
```

> **WICHTIG:** Kopiere den `whsec_...` Secret aus Terminal 3 und trage ihn in `/backend/config/.env.php` ein!

---

### Schritt 2: Restaurant Onboarding

1. **Admin Login:** 
   - Öffne `http://localhost:5173/a/{slug}`
   - Logge dich als Admin ein

2. **Stripe Settings öffnen:**
   - Klicke auf **"💳 Zahlungen"** in der Sidebar
   - Oder öffne direkt: `http://localhost:5173/a/{slug}/stripe`

3. **Stripe Account erstellen:**
   - Klicke auf "Stripe Account erstellen"
   - Gib deine E-Mail ein
   - Klicke auf "Onboarding starten"

4. **Stripe Onboarding durchlaufen:**
   - Du wirst zu Stripe weitergeleitet
   - Fülle die Formulare aus (Test-Daten sind OK!)
   - Vervollständige das Onboarding

5. **Status prüfen:**
   - Zurück in der Admin App
   - Status sollte ✅ **Charges Enabled** zeigen

---

### Schritt 3: Guest Payment testen

1. **Customer App öffnen:**
   - `http://localhost:5173/c/{slug}`
   - Produkte zum Warenkorb hinzufügen
   - Checkout öffnen

2. **Mit Stripe zahlen:**
   - Wähle "Pay with Card"
   - Test-Karte: `4242 4242 4242 4242`
   - Ablauf: `12/30`
   - CVC: `123`
   - Klicke "Zahlen"

3. **Verify:**
   - Im Terminal 3 solltest du sehen:
     ```
     [200] POST /webhooks/stripe [evt_...] payment_intent.succeeded
     ```
   - Order Status in DB sollte `paid` sein

---

## 🧪 Test-Karten

| Karte                | Typ               | Ergebnis         |
|----------------------|-------------------|------------------|
| 4242 4242 4242 4242  | Visa              | ✅ Erfolgreich   |
| 5555 5555 5555 4444  | Mastercard        | ✅ Erfolgreich   |
| 4000 0000 0000 0002  | Visa              | ❌ Declined      |
| 4000 0025 0000 3155  | Visa              | 🔐 3D Secure     |

**Alle Karten:**
- Ablauf: Beliebig in der Zukunft (z.B. `12/30`)
- CVC: Beliebige 3 Ziffern (z.B. `123`)

---

## 🐛 Troubleshooting

### Seite zeigt "404" oder "Ungültige URL"
- **Fix:** Server starten mit `npm run dev` im Frontend
- Prüfe: Browser Console (F12) auf Fehler

### "Stripe not configured" Error
- **Fix:** Prüfe `/backend/config/.env.php`:
  ```php
  'stripe' => [
      'secret_key' => env('STRIPE_SECRET_KEY', 'sk_test_...'),
      'public_key' => env('STRIPE_PUBLIC_KEY', 'pk_test_...'),
      'webhook_secret' => env('STRIPE_WEBHOOK_SECRET', 'whsec_...'),
  ],
  ```

### Webhook nicht empfangen (Status 500)
- **Fix:** `stripe listen` Terminal prüfen
- Webhook Secret in `.env.php` eintragen
- Backend Server neu starten

### API Route nicht gefunden
- **Fix:** Prüfe dass Backend auf Port 8000 läuft
- Frontend proxy in `vite.config.ts` sollte auf `http://localhost:8000` zeigen

---

## ✅ Success Checklist

- [ ] Backend läuft auf `localhost:8000`
- [ ] Frontend läuft auf `localhost:5173`
- [ ] Stripe CLI forwarding aktiv
- [ ] Webhook Secret in `.env.php` konfiguriert
- [ ] Admin kann "💳 Zahlungen" Seite öffnen
- [ ] Stripe Account erfolgreich erstellt
- [ ] Onboarding completed (✅ Charges Enabled)
- [ ] Test-Payment erfolgreich
- [ ] Webhook [200] empfangen
- [ ] Order Status = `paid` in Database

