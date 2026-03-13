# Fulbbo

**Full-stack social football platform** — Book fields, organize matches, build your team and connect with other players, all in one place.

> **Live demo:** [fulbbo.vercel.app](https://fulbbo.vercel.app)
>
> *The source code is private for intellectual property reasons. This document describes the project's architecture and technical decisions.*

---

## What Fulbbo does

Fulbbo solves two problems simultaneously: the friction of booking a football field and the difficulty of organizing a match with friends. It connects players with sports complexes in a single platform, with integrated payments and social management.

**For players:**

- Sports complex search on an interactive map with geolocation (Google Places API)
- Online field booking with configurable deposit payment via MercadoPago
- Match creation and management with A/B teams, positions and goalkeepers
- Real-time chat inside each match
- Friends system, favorite team member list and match invitations
- Custom jersey: number, colors, patterns (gradient, stripes, checkered, etc.)
- Push notifications and configurable reminders before the match

**For complex owners:**

- Field management panel: surface, price, availability by day and time slot
- Pricing zones with time-based discounts (e.g. "Happy Hour")
- Date-based closure and scheduled maintenance management
- MercadoPago account linking to receive deposits directly
- Deposit percentage configuration per field or per complex
- Dashboard with pending and approved reservations

---

## Tech Stack

| Layer             | Technology                                       |
| ----------------- | ------------------------------------------------ |
| **Monorepo**      | Turborepo + pnpm workspaces                      |
| **Web**           | Next.js 14 (App Router, Server Components, SSR)  |
| **Mobile**        | Expo (React Native)                              |
| **UI**            | Tailwind CSS + shadcn/ui + Lucide Icons          |
| **Auth**          | Clerk (users + organizations for complexes)      |
| **API**           | tRPC v11 (end-to-end type safety)                |
| **Database**      | PostgreSQL (Supabase) + Drizzle ORM              |
| **Payments**      | MercadoPago SDK (OAuth per complex)              |
| **Maps**          | Google Maps JS API + Places API                  |
| **Push**          | Web Push API (VAPID)                             |
| **Validation**    | Zod (client + server, shared via tRPC)           |
| **Deploy**        | Vercel + GitHub Actions CI/CD                    |

---

## Architecture decisions

**Monorepo with Turborepo + T3 Turbo**
The project started as a standard Next.js app. Migrating to a monorepo allowed sharing the Drizzle schema, tRPC types and UI components between the web (Next.js) and mobile (Expo) apps without duplicating code. Turborepo handles build caching and parallel task execution.

**tRPC over REST**
End-to-end type safety without code generation. Changes to the database schema propagate automatically all the way to client components. This completely eliminated the category of bugs where "the type returned by the API doesn't match what the frontend expects."

**Clerk over NextAuth**
Clerk handles organizations natively, which simplified the "one complex = one Clerk organization" model. Users with the `admin` role in the organization get access to the owner panel. The entire onboarding flow (email, Google OAuth, session management) comes resolved out-of-the-box.

**Drizzle ORM over Prisma**
The Drizzle schema is TypeScript code, which means IDE autocomplete and versioning in Git alongside the rest of the codebase. Queries are typed SQL with no magic or runtime overhead. Predictable and auditable migrations.

**MercadoPago over Stripe**
The target market is Argentina and LATAM. MercadoPago has higher penetration, supports local payment methods (cash payments at convenience stores, bank transfers, installments) and the checkout UX is familiar to local users. MercadoPago OAuth was implemented so each complex links its own account and receives payments directly (no intermediation).

**Google Places cache with geographic grid**
Google Places API charges per request. A PostgreSQL cache system was built with a geographic grid (~5km × 5km) that shares results between users in the same area, reducing API costs by up to 90% in high-traffic zones. TTL of 24-48h to maintain data freshness.

**Native Web Push API over third-party services**
For push notifications (match reminders, friend requests, match invitations) the Web Push API with VAPID was implemented directly, without subscribing to an external service. A full system of templates, delivery tracking and retries was modeled in the database.

---

## Project structure

```
fulbbo/
├── apps/
│   ├── nextjs/                         # Next.js 14 App Router
│   │   └── src/app/panel/
│   │       ├── buscar-complejos/       # Map search
│   │       ├── complejos-deportivos/   # Owner panel
│   │       ├── partidos/               # Match management
│   │       ├── amigos/                 # Social system
│   │       ├── notificaciones/         # Notification center
│   │       └── perfil/                 # Profile and jersey
│   └── expo/                           # Mobile app (React Native)
├── packages/
│   ├── api/                            # tRPC routers
│   │   └── src/router/
│   │       ├── match.ts                # Matches, teams, players, chat
│   │       ├── reservation.ts          # Reservations and payments
│   │       ├── mercado-pago.ts         # MP OAuth, preferences, webhooks
│   │       ├── friendship.ts           # Friends, requests, team
│   │       ├── match-invitation.ts     # Match invitations
│   │       ├── notification.ts         # Push notifications
│   │       ├── jersey-back.ts          # Jersey customization
│   │       ├── placesCache.ts          # Google Places cache
│   │       └── sportComplex/           # Complex management
│   ├── db/                             # Drizzle schema + Supabase client
│   └── ui/                             # Shared shadcn/ui components
└── tooling/
    ├── eslint/                         # Shared ESLint config
    ├── prettier/                       # Shared Prettier config
    ├── tailwind/                       # Shared Tailwind config
    └── typescript/                     # Base tsconfig
```

---

## Database schema (simplified)

```
User              1 ──── N  FriendRequest        (friend requests)
User              1 ──── N  Friendship            (established friendships)
User              1 ──── N  TeamMember            (favorite team, max 10)
User              1 ──── 1  UserJerseyPreferences (custom jersey)
User              1 ──── N  Reservation
User              1 ──── N  Notification

SportsComplex     1 ──── N  Field
SportsComplex     1 ──── N  ComplexAvailability   (hours per day)
SportsComplex     1 ──── N  ComplexClosure        (date-based closures)
SportsComplex     1 ──── N  PricingZone           (discount zones)
SportsComplex     1 ──── 1  MercadoPagoAccount    (linked MP account)

Field             1 ──── N  FieldAvailability     (hours per field)
Field             1 ──── N  FieldMaintenance      (scheduled maintenance)
Field             1 ──── N  Reservation
Field             1 ──── 1  FieldDepositSetting   (deposit % per field)

Reservation           ──── embedded Payment fields (MP preference, status, amounts)

Match             1 ──── 2  Team                  (team A and team B)
Match             1 ──── N  MatchPlayer           (players with position)
Match             1 ──── N  MatchMessage          (match chat)
Match             1 ──── N  MatchInvitation
Match             1 ──── N  MatchReminder
Match             0..1 ── 1  Reservation           (booked field)
```

**User roles:** `player` | `staff`
**Clerk Org roles (complexes):** `admin` | `member`
**Reservation statuses:** `pending` | `approved` | `rejected`
**Payment statuses (MercadoPago):** `pending` | `approved` | `rejected` | `cancelled`

---

## Main tRPC Routers

| Router               | Responsibility                                               |
| -------------------- | ------------------------------------------------------------ |
| `match`              | Match CRUD, teams, players, chat, reminders                 |
| `reservation`        | Create reservation, approve/reject, check availability      |
| `mercado-pago`       | OAuth flow, create payment preference, payment webhook      |
| `friendship`         | Send/accept/reject requests, manage team members            |
| `match-invitation`   | Invite players to a match, respond to invitations           |
| `notification`       | Push subscriptions, send/mark notifications                 |
| `sportComplex`       | Complex CRUD, fields, availability, pricing zones           |
| `jersey-back`        | Jersey templates, user preferences                          |
| `placesCache`        | Map field search with geographic cache                      |

---

## Relevant environment variables

> The project is not publicly installable. Listed here as a reference for the integrations used.

```env
# Database
DATABASE_URL=

# Clerk (auth)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=

# MercadoPago
MP_CLIENT_ID=
MP_CLIENT_SECRET=
NEXT_PUBLIC_MP_PUBLIC_KEY=

# Google Maps
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY=
GOOGLE_MAPS_SERVER_API_KEY=

# Web Push (VAPID)
NEXT_PUBLIC_VAPID_PUBLIC_KEY=
VAPID_PRIVATE_KEY=
```

---

<div align="center">

Built with ☕ and 🧉 in Argentina

</div>
