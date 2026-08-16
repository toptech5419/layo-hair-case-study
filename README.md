# LAYO HAIR: Case Study

> Booking and payment platform built from a client brief for a premium African hair styling
> salon in Lincoln, UK. Real customers, real appointments, real money through Stripe.
> Built end-to-end from scratch, with no template and no starter kit.

**Live:** https://layohair.shop

> **Why this repository has no source code.** This is production software for a paying client, so
> the application repository is private. This is a public case study covering architecture, schema,
> and the decisions behind them.

---

## The brief

The salon had no digital presence. Bookings arrived through Instagram DMs and WhatsApp, which meant
double-bookings, no-shows with no deposit taken, and no record of anything. The client wanted one
system that took bookings, took money, and let them run the business from a phone.

Delivered as a single platform: a customer-facing site with a booking flow, Stripe deposits, and an
admin dashboard. I owned it from requirements through to post-launch maintenance.

---

## Architecture

```
Customer                              Admin
   │                                     │
   ├─ browse styles (9 categories)       ├─ bookings: confirm / reschedule / cancel
   ├─ pick date ──► GET /api/slots       ├─ availability: hours, blocked dates
   │                    │                ├─ styles & pricing CRUD
   │                    ▼                ├─ review moderation
   │        availability x blocked dates ├─ analytics (Recharts)
   │        x existing bookings          └─ settings (singleton row)
   │        x buffer x min/max notice
   │                    │
   │                    ▼
   ├─ book (guest, no account) ──► POST /api/bookings
   │                                     │  re-validate slot server-side
   │                                     ▼
   ├─ pay deposit ──► POST /api/checkout ──► Stripe Checkout
   │                                            │
   │                                     webhook ▼
   │                              POST /api/webhook/stripe
   │                              (signature-verified, source of truth)
   │                                            │
   │                                            ▼
   ├─ confirmation email (Resend) ◄── booking CONFIRMED
   │
   └─ track by bookingRef ──► /track   (no login required)

              cron ──► /api/cron/send-reminders   (24h before, configurable)
```

### Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 (App Router) |
| UI | React 19, Tailwind CSS v4, Radix UI / shadcn/ui |
| Language | TypeScript 5 |
| ORM | Prisma 7 |
| Database | PostgreSQL (Supabase) |
| Auth | NextAuth v5, OAuth and credentials, admin only |
| Payments | Stripe Checkout with webhooks |
| Images | Cloudinary |
| Email | Resend |
| Client data | TanStack React Query |
| Forms | React Hook Form with Zod |
| Charts | Recharts |
| Hosting | Vercel |

---

## Data model

Eleven models. The interesting ones:

| Model | Notes |
|---|---|
| `Booking` | `customerId` is **nullable**, so guest bookings are first-class. `bookingRef` is a unique public cuid used for login-free tracking. Indexed on `date`, `status`, `customerId`, `bookingRef`. |
| `Style` | Service catalogue across 9 categories, with duration and price. Duration is what drives slot maths. |
| `Availability` | Recurring working hours. |
| `BlockedDate` | Explicit exclusions such as holidays and personal days, kept separate from working hours so a block never mutates the schedule. |
| `Payment` | One-to-many against `Booking`. `stripeSessionId` and `stripePaymentId` are both `@unique`. `Decimal(10,2)`, never float. |
| `Settings` | **Singleton row**, `id @default("default")`. Holds buffer minutes, deposit percentage, min/max advance notice, reminder lead time, home-service toggle. |
| `Review` | Moderated, so nothing appears publicly until approved. |

Enums: `Role`, `Category`, `BookingStatus`, `PaymentType`, `PaymentStatus`.

---

## Technical decisions

*Chose X over Y because Z.*

- **Chose a singleton `Settings` row over hard-coded business rules or environment variables.**
  Buffer time between appointments, deposit percentage, minimum notice, maximum booking horizon,
  and reminder lead time are all things the salon owner wanted to change without calling me. A
  config table means those are UI toggles rather than a deployment. It costs one extra join on the
  booking path and it removed an entire category of support request.

- **Chose nullable `customerId` with a public `bookingRef` over requiring account creation.**
  Forcing signup before a first booking is the single largest drop-off point in a salon funnel.
  Guest bookings store name, email and phone directly on the row, and a unique cuid `bookingRef`
  gives the customer a tracking URL with no password. The trade-off is a weaker identity model,
  since a guest cannot see history across bookings, and for this client's volume that was the
  correct side of the trade.

- **Chose the Stripe webhook as the source of truth over the client-side success redirect.**
  A customer who closes the tab after paying, or whose browser drops the redirect, must still end
  up with a confirmed booking. The redirect updates the UI optimistically, but the
  signature-verified webhook is what actually transitions the booking and writes the `Payment` row.
  A separate `/api/verify-payment` route reconciles the case where the webhook is slower than the
  redirect.

- **Chose to re-derive availability server-side on submit rather than trust the selected slot.**
  The slot list the customer sees is a snapshot. Between rendering and submitting, someone else can
  book. `POST /api/bookings` recomputes the slot against working hours, blocked dates, existing
  bookings, buffer, and notice windows before it writes. The client-side check is UX, the
  server-side check is correctness.

- **Chose `Decimal(10,2)` over float for every monetary column.**
  Deposits are a configurable percentage of a service price. Percentage arithmetic on binary floats
  produces amounts that do not reconcile against Stripe. Decimal at the database boundary means the
  number charged and the number stored are the same number.

---

## Engineering details worth naming

- **Conflict prevention is compositional.** A slot is bookable only if it survives working hours,
  blocked dates, existing bookings *plus their buffer*, minimum notice, and maximum advance horizon.
  Each rule is independent and independently configurable.
- **Deposit reconciliation.** Deposit charged at booking, balance tracked to completion, both rows
  linked to the same `Booking` through `PaymentType`.
- **Transactional email on state change.** Confirmation, cancellation and reminder are driven by
  booking status transitions, with `confirmationSent` and `reminderSent` flags on the row so a retry
  or a re-run of the reminder cron cannot double-send.
- **Home service as a first-class variant.** `serviceType` plus address fields on the booking, gated
  behind a settings toggle so the client can switch the offering off seasonally.
- **Review moderation.** Public reviews pass through an approval queue rather than posting live.

---

## Screenshots

<!-- TODO: add 3-4 screenshots. Suggested: style catalogue, the booking flow with the slot picker,
     the admin dashboard with the revenue chart, the tracking page. Put files in /screenshots.
     A recruiter scrolls before they read. -->

---

## Author

**Temitope Alabi**, MSc Computer Science (AI), University of Lincoln
[GitHub](https://github.com/toptech5419) · [LinkedIn](https://www.linkedin.com/in/toptech5419/) · alabitemitope51@gmail.com
