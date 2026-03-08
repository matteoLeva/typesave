# TypeSave — Architettura Tecnica Dettagliata

## Overview

TypeSave è un'app macOS che cattura i tasti digitati e li salva localmente con crittografia AES-256. L'infrastruttura commerciale è completamente serverless e non richiede database propri: usa GitHub API come storage per le licenze, Stripe per i pagamenti, e Vercel per hosting e API.

---

## Infrastruttura

```
┌──────────────────────────────────────────────────────────┐
│                    UTENTE FINALE                          │
└────────────┬──────────────────────────┬──────────────────┘
             │                          │
             ▼                          ▼
┌────────────────────┐      ┌───────────────────────┐
│  sssuperfast.com   │      │   App macOS TypeSave   │
│    /typesave       │      │   (Swift + SwiftUI)    │
│  (Vercel CDN)      │      └──────────┬────────────┘
└────────┬───────────┘                 │
         │                             │ /api/validate
         │ Buy CTA                     ▼
         ▼                  ┌───────────────────────┐
┌────────────────────┐      │  api.sssuperfast.com  │
│   Stripe Checkout  │      │  (Vercel Serverless)  │
│  buy.stripe.com/…  │      └──────────┬────────────┘
└────────┬───────────┘                 │
         │                             │ GitHub API
         │ webhook                     ▼
         ▼                  ┌───────────────────────┐
┌────────────────────┐      │  matteoLeva/           │
│ /api/webhook       │─────▶│  typesave-licenses     │
│ (genera licenza)   │      │  (repo privato JSON)   │
└────────────────────┘      └───────────────────────┘
         │
         │ success redirect
         ▼
┌────────────────────┐
│ /typesave?         │
│ purchased=1&       │
│ session_id=xxx     │
│ (mostra licenza)   │
└────────────────────┘
```

---

## Repositories

| Repo | Hosting | Scopo |
|------|---------|-------|
| `matteoLeva/typesave` | — | App Swift + CLAUDE.md + docs |
| `matteoLeva/typesave-web` | Vercel (`sssuperfast.com/typesave`) | Landing page HTML/CSS/JS |
| `matteoLeva/typesave-api` | Vercel (`api.sssuperfast.com`) | Backend licenze TypeScript |
| `matteoLeva/typesave-licenses` | GitHub API (privato) | Database JSON licenze |
| `matteoLeva/matteoLeva.github.io` | GitHub Pages (`sssuperfast.com`) | Root dominio (redirect) |

---

## DNS (OVHcloud)

| Tipo | Nome | Valore | TTL |
|------|------|--------|-----|
| A | `@` | `76.76.21.21` | 3600 |
| CNAME | `www` | `cname.vercel-dns.com` | 3600 |
| A | `api` | `76.76.21.21` | 3600 |

**Nota:** `sssuperfast.com` → Vercel (typesave-web), `api.sssuperfast.com` → Vercel (typesave-api).

---

## Flusso Acquisto Dettagliato

```
1. Utente clicca "Acquista" su sssuperfast.com/typesave
2. Redirect → buy.stripe.com/4gM14nbac0rS3n77lOaAw0d
3. Stripe Checkout (€14.99 + IVA automatica via Stripe Tax)
4. Pagamento completato → Stripe invia webhook POST a api.sssuperfast.com/api/webhook
5. webhook.ts:
   a. Verifica firma Stripe (STRIPE_WEBHOOK_SECRET)
   b. Genera UUID v4 come license key
   c. Salva JSON in matteoLeva/typesave-licenses via GitHub API
   d. Testa il customer Stripe con la license key (metadata)
   e. Redirect → sssuperfast.com/typesave?purchased=1&session_id=SESS_ID
6. Landing page rileva ?purchased=1, mostra modal
7. Modal chiama GET /api/license?session_id=SESS_ID con retry (8 tentativi × 2s)
8. Riceve license key → mostrata all'utente con pulsante "Copia"
9. Utente apre app macOS → "Attiva Licenza" → inserisce UUID
10. App chiama POST /api/validate con { key, machineId }
11. validate.ts:
    a. Cerca licenza in GitHub repo
    b. Verifica che machineId non sia già registrato (MAX_DEVICES=1)
    c. Aggiorna JSON con machineId
    d. Risponde { valid: true }
12. App attivata ✓
```

---

## Storage delle Licenze (GitHub API)

Ogni licenza è un file JSON in `matteoLeva/typesave-licenses`:

```
typesave-licenses/
└── licenses/
    └── {license-key}.json
```

Struttura JSON:
```json
{
  "key": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "email": "user@example.com",
  "stripeSessionId": "cs_live_...",
  "stripeCustomerId": "cus_...",
  "createdAt": "2026-03-08T04:00:00.000Z",
  "devices": ["machine-uuid-here"],
  "active": true
}
```

---

## API Endpoints

### POST `/api/webhook`
- **Auth:** Stripe signature (`stripe-signature` header)
- **Trigger:** Stripe `checkout.session.completed`
- **Azione:** Genera UUID, salva licenza su GitHub, testa customer Stripe

### POST `/api/validate`
```json
// Request
{ "key": "uuid", "machineId": "machine-identifier" }

// Response OK
{ "valid": true, "message": "License activated" }

// Response Error
{ "valid": false, "error": "Device limit reached" }
```

### GET `/api/license?session_id=xxx`
- **Uso:** Success page landing — recupera license key da session_id Stripe
- **Polling:** Max 8 tentativi × 2 secondi (webhook può arrivare con ritardo)

### POST `/api/lookup`
- **Auth:** Header `x-admin-secret: ADMIN_SECRET`
- **Body:** `{ "email": "..." }` o `{ "key": "..." }`
- **Uso:** Support admin — cerca licenze

---

## Variabili d'Ambiente (Vercel — typesave-api)

```
STRIPE_SECRET_KEY       → sk_live_...
STRIPE_WEBHOOK_SECRET   → whsec_...
GITHUB_TOKEN            → gho_... (scope: repo, contents R/W su typesave-licenses)
ADMIN_SECRET            → stringa random per /api/lookup
```

---

## App macOS (Swift)

```
TypeSave.app/
├── TypeSaveApp.swift          — Entry point, AppDelegate
├── KeyLogger/
│   ├── KeyboardMonitor.swift  — CGEvent tap per cattura tasti
│   └── TextBuffer.swift       — Buffer circolare con timestamp
├── Storage/
│   └── EncryptedStore.swift   — CryptoKit AES-GCM, chiave in Keychain
├── UI/
│   ├── MenuBarView.swift      — Barra menu con history
│   └── PreferencesView.swift  — Impostazioni + attivazione licenza
└── License/
    └── LicenseManager.swift   — Validazione con api.sssuperfast.com [TODO]
```

**Permessi macOS richiesti:**
- Accessibility (CGEvent tap per key logging)
- Network (validazione licenza)

---

## Design System (Landing)

| Elemento | Valore |
|----------|--------|
| Font Headlines | Cormorant (serif, Google Fonts) |
| Font Body | DM Sans (sans-serif) |
| Font Mono | DM Mono |
| Background | `#06080C` (quasi nero) |
| Text | `#EDE9E1` (avorio caldo) |
| Accent | `#FFB800` (amber gold) |
| Texture | CSS grain overlay (`filter: url(#grain)`) |
| Animazioni | Scroll reveal con `IntersectionObserver` |
| i18n | IT se `navigator.language.startsWith('it')`, EN default |

---

## TODO — Prossimi Step

- [ ] `LicenseManager.swift` — integrazione validazione in-app
- [ ] Email automatica con license key (Resend.com quando pronto)
- [ ] Analytics privacy-first (Plausible)
- [ ] macOS Sequoia compatibility check
- [ ] Notarizzazione app per distribuzione fuori App Store
