# Fulbbo

**Plataforma fullstack de fútbol social** — Reservá canchas, organizá partidos, armá tu equipo y conectá con otros jugadores, todo en un mismo lugar.

> **Demo en vivo:** [fulbbo.vercel.app](https://fulbbo.vercel.app)
>
> *El código fuente es privado por razones de propiedad intelectual. Este documento describe la arquitectura y decisiones técnicas del proyecto.*

---

## Qué hace Fulbbo

Fulbbo resuelve dos problemas en simultáneo: la fricción de reservar una cancha de fútbol y la dificultad de organizar un partido con amigos. Une a jugadores con complejos deportivos en una sola plataforma, con pagos integrados y gestión social.

**Para jugadores:**

- Búsqueda de complejos deportivos en mapa interactivo con geolocalización (Google Places API)
- Reserva online de canchas con pago de seña configurable vía MercadoPago
- Creación y gestión de partidos con equipos A/B, posiciones y arqueros
- Chat en tiempo real dentro de cada partido
- Sistema de amigos, lista de miembros de equipo favoritos e invitaciones a partidos
- Dorsal personalizable: número, colores, patrones (degradado, rayas, cuadros, etc.)
- Notificaciones push y recordatorios configurables antes del partido

**Para propietarios de complejos:**

- Panel de gestión de canchas: superficie, precio, disponibilidad por día y horario
- Zonas de precios con descuentos por franja horaria (ej: "Hora feliz")
- Gestión de cierres por fecha y mantenimientos programados
- Vinculación de cuenta MercadoPago para recibir señas directamente
- Configuración de porcentaje de seña por cancha o por complejo
- Dashboard de reservas pendientes y aprobadas

---

## Tech Stack

| Capa              | Tecnología                                      |
| ----------------- | ----------------------------------------------- |
| **Monorepo**      | Turborepo + pnpm workspaces                     |
| **Web**           | Next.js 14 (App Router, Server Components, SSR) |
| **Mobile**        | Expo (React Native)                             |
| **UI**            | Tailwind CSS + shadcn/ui + Lucide Icons         |
| **Auth**          | Clerk (usuarios + organizaciones para complejos)|
| **API**           | tRPC v11 (end-to-end type safety)               |
| **Database**      | PostgreSQL (Supabase) + Drizzle ORM             |
| **Payments**      | MercadoPago SDK (OAuth por complejo)            |
| **Maps**          | Google Maps JS API + Places API                 |
| **Push**          | Web Push API (VAPID)                            |
| **Validation**    | Zod (client + server, compartido vía tRPC)      |
| **Deploy**        | Vercel + GitHub Actions CI/CD                   |

---

## Decisiones de arquitectura

**Monorepo con Turborepo + T3 Turbo**
El proyecto nació como una app Next.js estándar. La migración a monorepo permitió compartir el schema de Drizzle, los tipos de tRPC y los componentes de UI entre la web (Next.js) y el móvil (Expo) sin duplicar código. Turborepo se encarga del build caching y la ejecución paralela de tareas.

**tRPC sobre REST**
Type safety end-to-end sin generar código. Los cambios en el schema de la base de datos se propagan automáticamente hasta los componentes del cliente. Eliminó completamente la categoría de bugs de "el tipo de datos que devuelve la API no coincide con lo que espera el frontend".

**Clerk sobre NextAuth**
Clerk maneja organizaciones de forma nativa, lo que simplificó la implementación del modelo de "un complejo = una organización Clerk". Los miembros con rol `admin` de la organización tienen acceso al panel de propietario. Además, el flujo de onboarding (email, Google OAuth, gestión de sesiones) viene resuelto out-of-the-box.

**Drizzle ORM sobre Prisma**
El schema de Drizzle es código TypeScript, lo que da autocompletado en el IDE y se puede versionar en Git junto al resto del código. Las queries son SQL puro con type safety encima, sin magia ni overhead de runtime. Migraciones predecibles y auditables.

**MercadoPago sobre Stripe**
El mercado target es Argentina y LATAM. MercadoPago tiene mayor penetración, soporta medios de pago locales (Rapipago, Pago Fácil, transferencia bancaria, cuotas) y la UX de checkout es familiar para los usuarios locales. Se implementó OAuth de MP para que cada complejo vincule su propia cuenta y reciba pagos directamente (sin intermediación).

**Caché de Google Places con grid geográfico**
Google Places API cobra por request. Se implementó un sistema de caché en PostgreSQL con grid geográfico (~5km × 5km) que comparte resultados entre usuarios de la misma zona y reduce los costos de API hasta un 90% en zonas con tráfico recurrente. TTL de 24-48h para mantener frescura de datos.

**Web Push API nativa sobre servicios de terceros**
Para las notificaciones push (recordatorios de partido, solicitudes de amistad, invitaciones) se implementó Web Push API con VAPID directamente, sin contratar un servicio externo. Se modeló un sistema completo de templates, delivery tracking y reintentos en la base de datos.

---

## Estructura del proyecto

```
fulbbo/
├── apps/
│   ├── nextjs/                         # Next.js 14 App Router
│   │   └── src/app/panel/
│   │       ├── buscar-complejos/       # Búsqueda con mapa
│   │       ├── complejos-deportivos/   # Panel propietario
│   │       ├── partidos/               # Gestión de partidos
│   │       ├── amigos/                 # Sistema social
│   │       ├── notificaciones/         # Centro de notificaciones
│   │       └── perfil/                 # Perfil y dorsal
│   └── expo/                           # App móvil (React Native)
├── packages/
│   ├── api/                            # tRPC routers
│   │   └── src/router/
│   │       ├── match.ts                # Partidos, equipos, jugadores, chat
│   │       ├── reservation.ts          # Reservas y pagos
│   │       ├── mercado-pago.ts         # OAuth MP, preferencias, webhooks
│   │       ├── friendship.ts           # Amigos, solicitudes, equipo
│   │       ├── match-invitation.ts     # Invitaciones a partidos
│   │       ├── notification.ts         # Push notifications
│   │       ├── jersey-back.ts          # Dorsales
│   │       ├── placesCache.ts          # Caché Google Places
│   │       └── sportComplex/           # Gestión de complejos
│   ├── db/                             # Drizzle schema + cliente Supabase
│   └── ui/                             # shadcn/ui components compartidos
└── tooling/
    ├── eslint/                         # Config ESLint compartida
    ├── prettier/                       # Config Prettier compartida
    ├── tailwind/                       # Config Tailwind compartida
    └── typescript/                     # tsconfig base
```

---

## Schema de base de datos (simplificado)

```
User              1 ──── N  FriendRequest        (solicitudes de amistad)
User              1 ──── N  Friendship            (amistades establecidas)
User              1 ──── N  TeamMember            (equipo favorito, máx. 10)
User              1 ──── 1  UserJerseyPreferences (dorsal personalizado)
User              1 ──── N  Reservation
User              1 ──── N  Notification

SportsComplex     1 ──── N  Field
SportsComplex     1 ──── N  ComplexAvailability   (horarios por día)
SportsComplex     1 ──── N  ComplexClosure        (cierres por fecha)
SportsComplex     1 ──── N  PricingZone           (zonas de descuento)
SportsComplex     1 ──── 1  MercadoPagoAccount    (cuenta MP vinculada)

Field             1 ──── N  FieldAvailability     (horarios por cancha)
Field             1 ──── N  FieldMaintenance      (mantenimientos)
Field             1 ──── N  Reservation
Field             1 ──── 1  FieldDepositSetting   (% seña por cancha)

Reservation           ──── embedded Payment fields (MP preference, status, amounts)

Match             1 ──── 2  Team                  (equipo A y equipo B)
Match             1 ──── N  MatchPlayer           (jugadores con posición)
Match             1 ──── N  MatchMessage          (chat del partido)
Match             1 ──── N  MatchInvitation
Match             1 ──── N  MatchReminder
Match             0..1 ── 1  Reservation           (cancha reservada)
```

**Roles de usuario:** `player` | `staff`
**Roles Clerk Org (complejos):** `admin` | `member`
**Estados de reserva:** `pending` | `approved` | `rejected`
**Estados de pago (MercadoPago):** `pending` | `approved` | `rejected` | `cancelled`

---

## tRPC Routers principales

| Router               | Responsabilidad                                              |
| -------------------- | ------------------------------------------------------------ |
| `match`              | CRUD de partidos, equipos, jugadores, chat, recordatorios   |
| `reservation`        | Crear reserva, aprobar/rechazar, consultar disponibilidad   |
| `mercado-pago`       | OAuth flow, crear preferencia de pago, webhook de pago      |
| `friendship`         | Enviar/aceptar/rechazar solicitudes, gestionar equipo       |
| `match-invitation`   | Invitar jugadores a un partido, responder invitaciones      |
| `notification`       | Push subscriptions, enviar/marcar notificaciones            |
| `sportComplex`       | CRUD de complejo, canchas, disponibilidad, pricing zones    |
| `jersey-back`        | Templates de dorsal, preferencias de usuario                |
| `placesCache`        | Búsqueda de canchas en mapa con caché geográfico            |

---

## Variables de entorno relevantes

> El proyecto no es instalable públicamente. Se lista como referencia de las integraciones.

```env
# Base de datos
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

Hecho con ☕ y 🧉 en Argentina

</div>
