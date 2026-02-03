# DSGVO/GDPR Compliance: Verschlüsselung sensibler Daten

## ✅ Implementiert am 2. Februar 2026

### Problem
Die `payload_json` Spalte in der `onboarding_requests` Tabelle enthielt personenbezogene Daten im Klartext:
- Vollständiger Name des Inhabers
- Vollständige Adresse (inkl. PLZ, Stadt)  
- E-Mail-Adresse
- Restaurant-Name

Dies verstößt gegen **Art. 32 DSGVO** (Sicherheit der Verarbeitung).

### Lösung
**Verschlüsselung sensibler Daten at-rest** mit AES-256-CBC + HMAC-Authentifizierung.

### Implementierung

#### 1. Neue Verschlüsselungsservice-Klasse
- **Datei**: `backend/src/Services/EncryptionService.php`
- **Algorithmus**: AES-256-CBC mit HMAC-SHA256 Authentifizierung
- **Key Derivation**: SHA-256 Hash des `APP_SECRET`
- **IV**: Random 16 bytes pro Verschlüsselung

#### 2. Angepasste Controller
- **OnboardingController**: Verschlüsselt `payload_json` beim Speichern
- **WebhookController**: Entschlüsselt `payload_json` beim Auslesen

#### 3. Migrations-Script
- **Datei**: `backend/migrations/encrypt_onboarding_payloads.php`
- Verschlüsselt alle bestehenden Einträge
- Prüft ob Daten bereits verschlüsselt sind (idempotent)

### Verwendung

#### Verschlüsselung (automatisch bei Onboarding)
```php
$encryption = new EncryptionService();
$encryptedPayload = $encryption->encrypt(json_encode($data));
```

#### Entschlüsselung (automatisch bei Webhook)
```php
$encryption = new EncryptionService();
$decryptedPayload = $encryption->decrypt($encryptedData);
$data = json_decode($decryptedPayload, true);
```

#### Manuelle Verschlüsselung existierender Daten
```bash
php backend/migrations/encrypt_onboarding_payloads.php
```

### Sicherheit

#### Was ist geschützt? ✅
- ✅ Passwörter: Bcrypt-Hash (irreversibel)
- ✅ Kitchen PINs: Argon2ID-Hash (irreversibel)
- ✅ 2FA Secrets: AES-256-CBC verschlüsselt
- ✅ 2FA Recovery Codes: Bcrypt-Hash (irreversibel)
- ✅ **Onboarding Payloads: AES-256-CBC verschlüsselt** ⭐ NEU

#### Nicht verschlüsselt (nicht sensibel)
- Restaurant-Name in `tenants` Tabelle (öffentlich sichtbar)
- E-Mail in `tenants` (benötigt für Login)
- Stripe IDs (public/safe)

### DSGVO Compliance Status

| Anforderung | Status | Implementierung |
|-------------|--------|-----------------|
| Art. 32 - Sicherheit der Verarbeitung | ✅ | AES-256-CBC Verschlüsselung |
| Art. 25 - Privacy by Design | ✅ | Automatische Verschlüsselung |
| Art. 5 - Datenminimierung | ✅ | Nur notwendige Daten gespeichert |
| Art. 17 - Recht auf Löschung | ✅ | Tenant-Löschung implementiert |

### Technische Details

#### Verschlüsselungsformat
```
Base64(IV || HMAC || Ciphertext)
├─ IV: 16 bytes (random)
├─ HMAC: 32 bytes (SHA-256)
└─ Ciphertext: variable (AES-256-CBC)
```

#### Key Management
- Master Key: `APP_SECRET` aus `.env.php`
- Derived Key: `SHA-256(APP_SECRET)` → 32 bytes
- **WICHTIG**: `APP_SECRET` NIEMALS committen!

### Wartung

#### Bei DB-Backup
- Verschlüsselte Daten bleiben im Backup verschlüsselt
- Backup ohne `APP_SECRET` ist nutzlos für Angreifer ✅

#### Bei Key-Rotation
1. Neuen `APP_SECRET` generieren
2. Alle verschlüsselten Daten mit altem Key entschlüsseln
3. Mit neuem Key neu verschlüsseln
4. ⚠️ **Nicht implementiert** - Bei Bedarf nachrüsten

### Testing

```bash
# Test Verschlüsselung
php backend/migrations/encrypt_onboarding_payloads.php

# Verify in DB (sollte Base64 sein, nicht JSON)
mysql -u smartqr_user -p smartqr_dev -e "SELECT LEFT(payload_json, 50) FROM onboarding_requests LIMIT 1;"
```

### Changelog

- **2026-02-02**: Initiale Implementierung
  - EncryptionService erstellt
  - OnboardingController & WebhookController angepasst
  - Migrations-Script für bestehende Daten
  - 1 Eintrag erfolgreich verschlüsselt
