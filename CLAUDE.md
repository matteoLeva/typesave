# TypeSave — Project Context for Claude

## Obiettivo
App macOS che cattura automaticamente tutto ciò che l'utente digita e lo salva localmente con crittografia AES-256. Permette di recuperare testi persi in pochi secondi.

**Modello di business:** One-time purchase a €14.99 tramite Stripe.

---

## Architettura

```
sssuperfast.com/typesave     → Landing page (Vercel, repo: typesave-web)
api.sssuperfast.com          → Backend licenze (Vercel, repo: typesave-api)
github.com/matteoLeva/typesave-licenses  → Database licenze (privato)
```

**Repos GitHub:**
- `matteoLeva/typesave` — app macOS (SwiftUI)
- `matteoLeva/typesave-web` — landing page (HTML/CSS/JS puro)
- `matteoLeva/typesave-api` — backend Node.js/TypeScript
- `matteoLeva/typesave-licenses` — database JSON delle licenze (privato)
- `matteoLeva/matteoLeva.github.io` — root del dominio sssuperfast.com

Per i dettagli tecnici completi → vedi [`architecture.md`](./architecture.md)

---

## Tech Stack

| Componente | Tecnologia |
|---|---|
| App | SwiftUI + AppKit + CryptoKit + Combine |
| Landing | HTML/CSS/JS puro (no framework) |
| Backend API | Node.js + TypeScript su Vercel |
| Database licenze | GitHub API (repo privato JSON) |
| Pagamenti | Stripe (Payment Links + Webhooks) |
| Hosting | Vercel (landing + API) |
| Dominio | sssuperfast.com su OVHcloud |
| i18n | Auto-detect `navigator.language` (IT/EN) |

---

## Flusso Vendita

```
Utente → buy.stripe.com/4gM14nbac0rS3n77lOaAw0d
       → Stripe checkout (€14.99 + IVA automatica)
       → Webhook → api.sssuperfast.com/api/webhook
       → Genera UUID license key
       → Salva in typesave-licenses repo (GitHub API)
       → Redirect → sssuperfast.com/typesave?purchased=1&session_id=xxx
       → Modal mostra license key
       → Utente apre app → inserisce key → /api/validate → attivata
```

---

## API Endpoints

| Endpoint | Metodo | Descrizione |
|---|---|---|
| `/api/webhook` | POST | Stripe webhook — genera license key |
| `/api/validate` | POST | Valida key + registra device |
| `/api/license` | GET | Recupera key da session_id Stripe |
| `/api/lookup` | POST | Admin — cerca licenze per email/key |

**Limite dispositivi:** 1 Mac per licenza.

---

## Variabili d'Ambiente (Vercel — typesave-api)

```
STRIPE_SECRET_KEY          sk_live_...
STRIPE_WEBHOOK_SECRET      whsec_...
GITHUB_TOKEN               gho_... (token con accesso a typesave-licenses)
ADMIN_SECRET               stringa random per /api/lookup
```

---

## Design System (Landing)

- **Font:** Cormorant (headlines) + DM Sans (body) + DM Mono (mono)
- **Colori:** bg `#06080C`, text `#EDE9E1`, accent `#FFB800` (amber gold)
- **Stile:** Dark editorial, grain texture, scroll reveal animations
- **i18n:** IT se `navigator.language.startsWith('it')`, EN default

---

## Regole

- Non committare mai chiavi API o segreti — usare sempre env vars
- La landing è in HTML/CSS/JS puro — no framework pesanti
- Il database licenze è su GitHub API — no servizi esterni aggiuntivi
- Stripe Tax gestisce l'IVA EU automaticamente (già configurato)
- Max 1 dispositivo per licenza (modificabile in `validate.ts` → `MAX_DEVICES`)

---

## Milestone

- [x] App macOS v1.0.0 — completata
- [x] Landing page IT/EN — live su Vercel
- [x] Stripe prodotto + Payment Link — configurato
- [x] Backend licenze — deployato su Vercel
- [x] Dominio sssuperfast.com — configurato
- [ ] Email automatica con license key (richiede Resend o simile)
- [ ] Integrazione licenza nell'app Swift (LicenseManager.swift)
- [ ] Analytics basilari (Plausible o simile, privacy-first)
