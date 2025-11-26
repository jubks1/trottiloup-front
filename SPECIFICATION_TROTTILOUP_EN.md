# Functional & Technical Specification — Le Trottiloup

> IMPORTANT: The entire website user interface (all labels, buttons, messages, content presented to end users) MUST be in French only. This specification is written in English for development clarity, but the product is monolingual French.

---

## 1. Global Overview

### 1.1 Context

"Le Trottiloup" is a scooter (trottinette) racing event for scout units. Each unit submits one registration for a chosen race, including multiple teams. Races have participant constraints and a pricing model. Each unit has exactly one responsible leader.

### 1.2 Primary Objectives

- Provide a smooth, validated race registration experience.
- Centralize data for a future administrative dashboard (statistics, payment status, participant breakdown).
- Guarantee data integrity (participant constraints, anti-abuse, coherent totals) with clear frontend error handling.
- Present essential information (dates, rules, safety) in a modern, readable, aesthetically pleasing UI.

### 1.3 Target Audiences & UX

- **Louveteaux (8–12 years)**: Vibrant yet restrained colors, lightly playful elements, high readability (rounded fonts, accessible contrast).
- **Chefs (young adult leaders)**: Clarity, structured typography, consistent sections, obvious navigation.
- **Administrators (future)**: API & schema ready for extensions (exports, dashboard, audits).

### 1.4 Design Principles

- Limited palette (e.g., deep blue, orange accent, neutral grays). Accessible contrast (AA/AAA for main text).
- Reusable atomic components (buttons, cards, alerts, forms) via Tailwind CSS.
- Fully responsive layout (MUST support mobile, tablet, desktop equally). Mobile-first approach: all pages, components, and interactions must render cleanly and remain fully usable on small screens (≥320px) as often accessed via phones. Registration form specifically optimized for mobile entry (reduced scrolling, large touch targets).
- Immersive hero background image (Unsplash) with subtle dark gradient overlay for readability.

### 1.5 Security & Compliance

- Dual validation (client + server). Server-side remains source of truth.
- Anti-spam/abuse protection (rate limiting critical endpoints: registration submission, team addition).
- Basic IP heuristic detection (e.g., > X registrations within 5 minutes flagged).
- Minimal GDPR compliance: privacy & cookies pages (monolingual French).
- Admin area protection: password-only access via a specific URL, session cookie, secure headers.

### 1.6 High-Level Application Architecture

- Single Next.js repository (frontend + API routes).
- PostgreSQL database (UTF8), hosted (Supabase / Railway / Neon / similar).
- ORM: Prisma (schema modeling, migrations, type safety).
- Authentication: admin-only password login (no public accounts). Admin UI accessible via a specific, unlinked URL; session-based auth.
- Observability: structured logs (console + optional hosted log service later).

### 1.7 Key User Journey

1. Landing → sees event title, visuals, teaser videos.
2. Navigates to Registration → selects race → enters unit, leader, teams.
3. Submits → receives confirmation (page or toast) + potential future email.
4. Can consult Rules / Contact / Legal pages.

### 1.8 Future KPIs (Anticipated)

- Total registrations per race.
- Participant vs team distribution.
- Form conversion rate (start → successful submission).
- Constraint error rate (min/max participant violations).

---

## 2. Backend

### 2.1 Technical Stack

- **Framework**: Next.js API Routes.
- **Language**: TypeScript.
- **ORM**: Prisma.
- **Database**: PostgreSQL (UTF8).
- **Validation**: Zod (request payload schemas).
- **Security**: Helmet (headers), rate limiting (`@upstash/ratelimit` or Redis custom). Planned, can be staged.
- **Admin Auth**: Minimal password-based login. Store a bcrypt-hashed admin password in env (e.g., `ADMIN_PASSWORD_HASH`). Session cookie (`httpOnly`, `secure` in prod, `SameSite=strict`) with short TTL and inactivity timeout.

### 2.2 Data Models (Conceptual / Prisma)

#### Unit

- id (Int, identity)
- unitName (String, required)
- region (String, optional)
- createdAt (DateTime, default now)
  Relations: 1 Unit → 1 Leader, 1 Unit → N Registrations.

#### Leader

- id (Int)
- unitId (Int, unique, FK → Unit.id)
- firstName (String)
- lastName (String)
- email (String)
- phone (String)
- createdAt (DateTime)
  Relation: 1 Leader ↔ 1 Unit.

#### Race

- id (Int)
- name (String, unique)
- participationPrice (Decimal NUMERIC(10,2))
- minParticipants (Int > 0)
- maxParticipants (Int >= minParticipants)
- raceDate (Date)
- description (Text nullable)
- createdAt (DateTime)
  Relation: 1 Race → N Registrations.

#### Registration

- id (Int)
- unitId (Int, FK → Unit.id)
- raceId (Int, FK → Race.id)
- totalParticipants (Int > 0)
- totalPrice (Decimal NUMERIC(10,2))
- paymentStatus (Enum: PENDING, PAID; default PENDING)
- createdAt (DateTime)
  Relations: N Registrations → 1 Unit; N Registrations → 1 Race; 1 Registration → N Teams.

#### Team

- id (Int)
- registrationId (Int, FK → Registration.id)
- teamName (String)
- participantCount (Int > 0)
- createdAt (DateTime)
  Relation: N Teams → 1 Registration.

### 2.3 Enum

`paymentStatus`: PENDING | PAID

### 2.4 Constraints & Business Rules

- `sum(Teams.participantCount) == Registration.totalParticipants` (enforced server-side during creation).
- Each Team `participantCount` must respect race min/max boundaries (server validation).
- Abuse prevention: if `teams.length > MAX_TEAMS_PER_REQUEST` (e.g., 10) → reject or require manual verification (future captcha).
- Rate limit registration attempts per IP (e.g., >3 successful or >5 failed <2 min → temporary block).

### 2.5 Recommended Indexes

- Leader(unitId) UNIQUE
- Race(name) UNIQUE
- Registration(unitId), Registration(raceId), Registration(paymentStatus)
- Team(registrationId)

### 2.6 Registration Flow (API)

Endpoint: POST `/api/registration`
Payload example:

```jsonc
{
  "raceId": 1,
  "unit": { "unitName": "Scouts de la Forêt", "region": "Lux" },
  "leader": {
    "firstName": "Jean",
    "lastName": "Dupont",
    "email": "jean@example.com",
    "phone": "+32470000000"
  },
  "teams": [
    { "teamName": "Louveteaux Rapides", "participantCount": 5 },
    { "teamName": "Les Verts", "participantCount": 4 }
  ]
}
```

Server steps:

1. Validate payload (Zod) + business rules.
2. Fetch Race + verify team-level and aggregate constraints.
3. Create or upsert Unit + Leader (ensure uniqueness). If existing with conflicting leader, handle as conflict.
4. Compute `totalParticipants` from team sum.
5. Compute `totalPrice = participationPrice * totalParticipants` (fixed pricing model).
6. Transaction: create Registration + Teams.
7. Return success `{ id, paymentStatus }`.

Confirmation payload (extended response consumed by confirmation page):

- On success, return a detailed summary so the frontend can render the confirmation without any additional fetch:

```jsonc
{
  "id": 123,
  "paymentStatus": "PENDING",
  "createdAt": "2025-11-26T10:00:00.000Z",
  "race": {
    "id": 1,
    "name": "Catégorie Louveteaux",
    "raceDate": "2026-04-20",
    "participationPrice": 10.0,
    "minParticipants": 3,
    "maxParticipants": 12
  },
  "unit": { "unitName": "Scouts de la Forêt", "region": "Lux" },
  "leader": {
    "firstName": "Jean",
    "lastName": "Dupont",
    "email": "jean@example.com",
    "phone": "+32470000000"
  },
  "teams": [
    {
      "teamName": "Louveteaux Rapides",
      "participantCount": 5
    },
    { "teamName": "Les Verts", "participantCount": 4 }
  ],
  "pricing": {
    "participationPrice": 10.0,
    "totalParticipants": 9,
    "numberOfTeams": 2,
    "totalPrice": 90.0
  }
}
```

Note: the confirmation page uses this response directly and does not perform a second fetch. If the page is reloaded without state, redirect back to `/inscription`.

### 2.7 Admin Authentication & Endpoints

Admin endpoints are protected by session-based auth established via a password login. All responses are in JSON (CSV endpoints return `text/csv`). UI text remains French.

Auth:

- POST `/api/admin/login`
  - Body: `{ "password": "..." }`
  - Verifies against env-stored bcrypt hash; on success, issues session cookie.
  - Rate limited (e.g., 5 attempts / 5 min per IP). Returns `401` on failure.
- POST `/api/admin/logout`
  - Clears session cookie.

Teams:

- GET `/api/admin/teams`
  - Filters: `raceId` (number), `unitName` (string, partial), `createdFrom`/`createdTo` (ISO), pagination `page`/`pageSize`, `sort`.
  - Returns: `teamName`, `participantCount`, `unitName`, `raceName`, `createdAt`, `id`.
- GET `/api/admin/teams.csv`
  - Same filters; returns CSV with headers: `Nom de l'équipe,Nombre de participants,Unité,Course,Créé le`.

Leaders:

- GET `/api/admin/leaders`
  - Returns: `unitName`, `firstName`, `lastName`, `email`, `phone`, `createdAt`, `id`.
- GET `/api/admin/leaders.csv`
  - CSV headers: `Unité,Prénom,Nom,Email,Téléphone,Créé le`.

Units:

- GET `/api/admin/units`
  - Returns: `unitName`, `region`, `createdAt`, `id`.
- GET `/api/admin/units.csv`
  - CSV headers: `Unité,Région,Créé le`.

Registrations:

- GET `/api/admin/registrations`
  - Returns: `unitName`, `raceName`, `totalParticipants`, `totalPrice`, `paymentStatus`, `createdAt`, `id`.
- GET `/api/admin/registrations.csv`
  - CSV headers: `Unité,Course,Participants,Total,Statut de paiement,Créé le`.
- PATCH `/api/admin/registrations/{id}/mark-paid`
  - Sets `paymentStatus = PAID`. Returns updated record summary.

Notes:

- All admin routes return `401 UNAUTHORIZED` if no session, `403 FORBIDDEN` if session invalid/expired.
- Add indexes or query filters where needed to keep list endpoints responsive.

### 2.8 Error Handling & Codes

- 400: Invalid payload / violated business rule.
- 409: Conflict (existing leader mismatch, closed race, duplicate scenario).
- 422: Valid format but logical constraint (min/max) failed.
- 429: Too many requests (abuse prevention).
- 500: Internal error (logged, user-friendly generic message).

Error JSON format:

```json
{
  "error": {
    "code": "PARTICIPANT_CONSTRAINT",
    "message": "Le nombre de participants dépasse la limite de la course."
  }
}
```

Admin-specific error codes:

- `UNAUTHORIZED` → "Authentification requise."
- `FORBIDDEN` → "Accès refusé. Session invalide ou expirée."
- `RATE_LIMIT_LOGIN` → "Trop de tentatives de connexion. Réessayez plus tard."

Messages remain in French for UI consumption.

### 2.9 Validation & Anti-Abuse Heuristics

- Ensure `teams.length <= MAX_TEAMS_PER_REQUEST`.
- Ensure `totalParticipants <= race.maxParticipants`.
- Flag anomalous ratios (e.g., team count high with extremely large participantCount) for logging.
- Apply IP rate limiting.

### 2.10 Observability & Logging

Structured JSON logs: `{ level: "info", event: "registration.created", registrationId: 123 }`.
Errors: level `error` + stack trace.
Admin:

- Log `admin.login.success`/`admin.login.failure` with IP and minimal metadata (no raw password), `admin.export` events (entity, filter size), `admin.registration.markPaid` with registration id.

### 2.11 Migrations

Managed via Prisma CLI commands:

- `npx prisma migrate dev --name add_race_table` (development)
- `npx prisma migrate deploy` (production)

Naming convention: `YYYYMMDDHHMM_add_race_table` etc. (automatically generated by Prisma).

---

## 3. Frontend

### 3.1 Stack & Tools

- **Framework**: Next.js (App Router preferred).
- **Language**: TypeScript.
- **Styling**: Tailwind CSS (custom palette + light motion).
- **Forms**: React Hook Form + `@hookform/resolvers/zod`.
- **UI Accessibility**: Headless UI or Radix if needed.
- **Error Handling**: `<AlertErreur />` component showing code + friendly French message.

### 3.2 Pages & Content (All rendered text strictly in French)

#### Landing (`/`)

- Hero title: "Le Trottiloup" (display font).
- Full-screen background image (Unsplash trottinette / nature) with gradient overlay.
- Teaser video section: two HTML5 videos (autoplay, muted, loop, lazy loaded). Placeholder fallback.
- Key Dates section: cards per race (Name, Date, Registration period) + button "S'inscrire".
- Safety section: list (Helmet required, knee pads recommended, supervision). Icons (Heroicons).
- Footer: links to Privacy, Cookies, Terms, Contact.

#### Registration (`/inscription`)

Multi-section form:

1. Race selection (radio/select) → dynamic description + min/max badges.
2. Unit: Name (required), Region (optional).
3. Leader: First name, Last name, Phone, Email (all required; French validation messages).
4. Teams: "Add Team" button → each block: Name, participantCount + min/max info. Removable.
5. Live summary: total participants + computed price.
6. Submit button (loading state, disabled on errors). Success toast/page in French.

#### Registration Confirmation (`/inscription/confirmation`)

- Purpose: display a detailed, French-only recap immediately after a successful registration.
  Access: redirected from successful POST `/api/registration` and consumes the full response payload directly (no second fetch). If accessed without payload (e.g., page refresh), show a soft error and link back to `/inscription`.
- Content blocks (all labels in French):
  - Race: `Nom de la course` (Race name), `Date de la course` (Race date), `Limites` (min/max participants limits) via `BadgeLimite`.
  - Teams: readable table with columns `Nom de l'équipe` (Team name), `Participants`.
  - Price summary:
    - `Prix par participant` (Price per participant) displayed.
    - `Participants totaux` (Total participants) and `Nombre d'équipes` (Number of teams).
    - `Total` (computed as `participationPrice * totalParticipants`) displayed prominently, with thousands separator and `EUR`.
  - Payment instructions:
    - Display a prominent info box with payment details (all labels in French):
      - Message: "Veuillez effectuer le paiement de {{totalPrice}} EUR sur le compte bancaire suivant avant le {{paymentDeadline}}:" (Please make the payment of {{totalPrice}} EUR to the following bank account before {{paymentDeadline}}:)
      - Bank account number (IBAN) from configuration.
      - Account holder name from configuration.
      - Payment reference: Registration ID or unit name for identification.
    - Styling: light blue background, border, prominent typography for amount and deadline.
  - Actions:
    - Link `Retour à l'accueil` (Back to home).
- UI states:
  - Consumes data returned by the registration creation endpoint (present in the response).
  - Displays `AlertErreur` if payload is missing, with French message and option to return to the form.
  - Loads data via `/api/registrations/{id}` (admin-protected for extras, but public confirmation only reads non-sensitive summary via a dedicated route, see Backend).
  - Displays `AlertErreur` if retrieval fails, with French message and retry option.
  - Responsive: stacked cards on mobile, teams table with horizontal scroll if necessary.

#### Rules (`/reglement`)

Suggested sections: Event purpose, Participation conditions, Safety & equipment, Race flow, Scout spirit, Responsibilities & insurance, Sanctions. Structured with `h2`, bullet lists, info boxes.

#### Contact (`/contact`)

- Official email (mailto).
- Facebook / Instagram links.
- Previous year after-movie (YouTube embed).
- Optional simple contact form (future).

#### Privacy Policy (`/confidentialite`)

French GDPR minimal: collected data (registration), purpose (event organization), retention period, rights (rectification, deletion), contact email, hosting provider mention.

#### Terms of Service (`/cgu`)

Purpose, scope, user responsibilities, liability limitation, intellectual property (logo), site modification clause.

#### Cookies Policy (`/cookies`)

State no non-essential cookies initially. Future tracking requires consent banner.

#### Admin (`/admin`) & Admin Login (`/admin/connexion`)

- Admin Login page (not linked in public UI, accessible only via a specific URL): password field, submit, French validation, error messages and rate-limit messaging.
- Admin Dashboard page (`/admin`):
  - Global header: "Administration — Le Trottiloup".
  - Tabs with tables and filters:
    - Teams (Équipes): filter by race, unit, creation date; columns: Team name, Participant count, Unit, Creation date; export CSV.
    - Leaders (Responsables): columns: Unit, First name, Last name, Email, Phone, Creation date; export CSV.
    - Units (Unités): columns: Unit, Region; export CSV.
    - Registrations (Inscriptions): columns: Unit, Race, Total participants, Total price, Payment status, Creation date; actions: "Mark as paid" button; export CSV.
  - All strings displayed strictly in French. Tables support pagination, sorting, and filtering.
  - If session missing/expired, redirect to `/admin/connexion`.

### 3.3 Key Components

- `Layout` (header + footer). Header: stylized text logo, links to Inscription, Règlement, Contact.
- `Hero` (landing page banner).
- `VideoTeaser` (accessible video wrapper, fallback image).
- `FormSection`, `FieldGroup` (consistent spacing & grouping).
- `TeamFormItem` (reusable team block).
- `AlertErreur` (soft red background, red border, exclamation icon).
- `BadgeLimite` (Race min/max display).
- `LoadingButton` (spinner + `aria-busy`).
- Admin components:
  - `AdminLayout` (secured shell).
  - `DataTable` (sortable, paginated, filter-aware table).
  - `ExportCsvButton`.
  - `FiltersBar` (race/unit/date filters).
  - `AdminLoginForm`.
  - Confirmation components:
    - `ConfirmationSummary.tsx` (header + key facts).
    - `ReceiptDetails.tsx` (unit, leader, race, teams, pricing breakdown).

### 3.4 Error Handling (UI)

- Field-level errors below inputs (short French messages, red color).
- Global form error panel at top for aggregate issues.
- Mapping backend error codes → French messages:
  - `PARTICIPANT_CONSTRAINT` → "Nombre de participants invalide pour cette course."
  - `RACE_NOT_FOUND` → "Course sélectionnée introuvable."
  - `TOO_MANY_TEAMS` → "Trop d'équipes envoyées d'un coup. Réessayez en divisant."
  - `RATE_LIMIT` → "Trop de tentatives récentes. Patientez un instant."

### 3.5 Accessibility

- WCAG contrast adherence.
- Proper `<label for>` associations.
- Error messages in `aria-live="assertive"` regions.
- Touch-target sizing ≥44px.
- Full keyboard navigation with visible focus styles.

### 3.6 Tailwind Styling (Example)

Palette example (`tailwind.config.js`):

```js
module.exports = {
  theme: {
    extend: {
      colors: {
        primaire: "#1E3A8A", // deep blue
        accent: "#F97316", // bright orange
        neutre: {
          50: "#F9FAFB",
          100: "#F3F4F6",
          300: "#D1D5DB",
          700: "#374151",
          900: "#111827",
        },
      },
    },
  },
};
```

Usage: `class="bg-primaire text-neutre-50"`, `hover:bg-accent`.

### 3.7 Forms & Validation

- Shared isomorphic Zod schemas for consistency.
- React Hook Form + Zod resolver.
- Debounced computation of total participants.
- Prevent duplicate rapid submissions (disable submit + request id token).

### 3.8 Frontend Anti-Abuse

- Limit team blocks added: if >10 show educational message instead of adding more.
- Display team counter.
- Immediate red highlight + tooltip if participantCount > race max.

### 3.9 Feedback & Notifications

- Success: green toast ("Inscription envoyée !" + participant summary).
- Failure: red toast with code (for support) + French message.
- Global loading overlay on mutation (light semi-transparent layer).

### 3.10 Rules (Initial Suggested French Content)

```
RÈGLEMENT — LE TROTTILOUP

1. Objet
Le Trottiloup est une course de trottinettes destinée aux unités scouts. L'objectif est de promouvoir l'esprit d'équipe, la sécurité et le fair-play.

2. Conditions de participation
Chaque équipe est composée d'un nombre de participants respectant les limites de la course choisie. Les inscriptions se font via le formulaire officiel.

3. Sécurité
Casque obligatoire. Genouillères et gants fortement recommandés. Tout comportement dangereux entraîne l'exclusion.

4. Déroulement
Les courses se tiennent à la date indiquée pour chaque catégorie. Les horaires détaillés seront communiqués par e-mail au responsable.

5. Esprit scout
Respect mutuel, entraide et honnêteté priment. Aucune triche ne sera tolérée.

6. Responsabilités
Les organisateurs déclinent toute responsabilité en cas de non-respect des règles de sécurité. L’unité doit assurer la supervision de ses membres.

7. Données
Les informations fournies sont utilisées uniquement pour l'organisation. Aucune revente.
```

### 3.11 Performance

- Optimize images (Next/Image), preload hero.
- Lazy-load teaser videos.
- Code splitting per page.

### 3.12 Internationalization

- Not required (French-only). Centralize all strings in `i18n/fr.ts` for future extension.

### 3.13 Basic SEO

- Meta title: "Le Trottiloup — Course de Trottinettes Scouts".
- Meta description: concise event description.
- Open Graph image: compressed hero image.

### 3.14 Potential Frontend Evolution

- Admin summary dashboard (private area).
- Confirmation email integration (mail provider).
- Light gamification (team badges).

### 3.15 Suggested Directory Structure

```
/src
  /app
    /page.tsx (Landing)
    /inscription/page.tsx
    /inscription/confirmation/page.tsx
    /reglement/page.tsx
    /contact/page.tsx
    /confidentialite/page.tsx
    /cgu/page.tsx
    /cookies/page.tsx
  /components
    Hero.tsx
    RegistrationForm.tsx
    TeamFormItem.tsx
    AlertErreur.tsx
    BadgeLimite.tsx
    Layout.tsx
    AdminLayout.tsx
    DataTable.tsx
    ExportCsvButton.tsx
    FiltersBar.tsx
    AdminLoginForm.tsx
    ConfirmationSummary.tsx
    ReceiptDetails.tsx
  /lib
    api.ts (fetch abstractions)
    validation.ts (Zod schemas)
    pricing.ts (total price logic)
  /styles
    globals.css
  /i18n
    fr.ts
  /utils
    antiAbus.ts
/prisma
  schema.prisma
  /app
    /admin/page.tsx
    /admin/connexion/page.tsx
```

### 3.16 Example Zod Schema (Simplified)

```ts
import { z } from "zod";

export const teamSchema = z.object({
  teamName: z.string().min(2, "Nom trop court"),
  participantCount: z.number().int().positive("Doit être > 0"),
});

export const registrationSchema = z.object({
  raceId: z.number().int(),
  unit: z.object({
    unitName: z.string().min(2, "Nom requis"),
    region: z.string().optional(),
  }),
  leader: z.object({
    firstName: z.string().min(2),
    lastName: z.string().min(2),
    email: z.string().email("Email invalide"),
    phone: z.string().min(6),
  }),
  teams: z.array(teamSchema).min(1, "Au moins une équipe"),
});
```

(Validation messages remain in French for direct UI binding.)

### 3.17 User Feedback & Tone

- Tone: positive, direct, simple, reassuring.
- Success example: "Inscription enregistrée ! Nous reviendrons vers vous pour les détails pratiques.".
- Error example: "Veuillez vérifier le nombre de participants pour l'équipe \"Louveteaux Rapides\".".

### 3.18 Pre-Launch Quality Checklist

- Lighthouse Mobile performance >85.
- No console errors.
- Full keyboard accessibility.
- Legal pages reachable via footer.

---

## Annexes

### A. Relational Diagram (Text)

Unit(1) — Leader(1)
Unit(1) — Registration(N)
Race(1) — Registration(N)
Registration(1) — Team(N)

### B. Pricing Model (Fixed)

- **Fixed pricing formula**: `totalPrice = participationPrice * totalParticipants`
- The `participationPrice` is defined per race and represents the cost per individual participant.
- Example: If `participationPrice = 10 EUR` and `totalParticipants = 9`, then `totalPrice = 90 EUR`.

### C. Unit Duplicate Strategy

If `unitName` exists with a different leader → require confirmation (French message: "Cette unité possède déjà un responsable enregistré. Contactez l'organisation si changement.").

### D. Audit Logging (Future)

AuditLog table: id, entity, entityId, action, timestamp, actor (optional), diff JSON.

---

Initial English specification completed. UI remains strictly French.
