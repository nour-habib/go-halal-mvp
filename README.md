# GoHalal — Halal Food Delivery Platform

> A full-stack food delivery MVP built for halal eaters in Vancouver, BC. Connects customers with verified halal restaurants for discovery, ordering, and delivery.

---

## Overview

GoHalal is a production-oriented food delivery platform built from the ground up. The project covers the full product surface: a consumer-facing web app, a REST API backend, database design, authentication, payments, and a coming-soon landing page with a waitlist.

The goal was to build something production-ready — not a tutorial project — with real decisions around security, performance, and user experience.

---

## Tech Stack

### Frontend
| | |
|---|---|
| Framework | Next.js 15 (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS v4 (`@theme` directive) |
| Fonts | DM Sans (display), Plus Jakarta Sans (UI), Be Vietnam Pro (body) |
| Icons | Material Symbols Outlined (Google) |
| Maps | Google Places API via `@vis.gl/react-google-maps` |
| Payments | Stripe.js + React Stripe Elements |
| State | React Context (Auth + Cart) |

### Backend
| | |
|---|---|
| Framework | Spring Boot 3.5 |
| Language | Java 24 |
| Security | Spring Security + JWT (JJWT 0.12) |
| Database | PostgreSQL 15 |
| ORM | Spring Data JPA / Hibernate |
| Payments | Stripe Java SDK |
| Build | Gradle (Kotlin DSL) |

---

## Architecture

```
┌─────────────────────────────────────────────┐
│                 Client (Next.js)            │
│                                             │
│  AuthContext ──► /api/auth/*                │
│  CartContext ──► /api/cart/*                │
│  Pages       ──► /api/restaurants/*         │
│                  /api/orders/*              │
│                  /api/checkout/*            │
└─────────────────────┬───────────────────────┘
                      │ HTTP/REST + JWT
┌─────────────────────▼───────────────────────┐
│           Spring Boot REST API              │
│                                             │
│  JwtAuthFilter → SecurityConfig             │
│  AuthController / SearchController          │
│  CartController / OrderController           │
│  CheckoutController                         │
└─────────────────────┬───────────────────────┘
                      │ JPA
┌─────────────────────▼───────────────────────┐
│              PostgreSQL 15                  │
│                                             │
│  users · restaurants · categories · tags   │
│  menu_items · carts · cart_items            │
│  orders · order_items                       │
└─────────────────────────────────────────────┘
```

The frontend and backend are fully decoupled — the Next.js app is a pure API client. CORS is locked to the app origin and the backend is stateless (JWT, no server-side sessions).

---

## Features

### Consumer App (`/mvp`)

**Discovery**
- Restaurant listing with live backend data
- Cuisine filtering via sidebar (Central Asian, Chinese, Middle Eastern — Levant/Yemeni, Pakistani)
- Full-text search across restaurant names, categories, and tags
- Nearby restaurant lookup using the Haversine distance formula
- Google Places autocomplete for delivery address

**Ordering**
- Session-based cart with single-restaurant enforcement
- Menu browsing with item customisation (special instructions)
- Checkout flow with delivery address, contact info, promo codes
- Stripe payment integration (PaymentIntent API)
- Order history and status tracking

**Authentication**
- JWT-based register and login
- Passwords hashed with BCrypt
- Token persisted in `localStorage`, restored on page load via `/api/auth/me`
- Protected routes redirect unauthenticated users to `/login`
- Auth state (user name, token) surfaced in the nav

**Account**
- Profile management (name, email, phone, country)
- Password change with current password verification
- Saved payment methods (UI layer — Stripe integration ready)

**Other Pages**
- Deals page with copyable promo codes
- Sadaqah Sofra — community giving feature (charity donation at checkout)
- Careers page with expandable role listings → contact form pre-fill
- Partner with Us page → contact form pre-fill
- Contact form (Formspree-backed, async submit with success state)

**Mobile**
- Responsive layout with mobile-first breakpoints
- Bottom navigation bar on mobile (`md:hidden`)
- Touch-friendly 44px tap targets
- Collapsible sidebar on desktop

### Landing Page (`/`)
- Static coming-soon page served at the root domain
- Email waitlist form via Formspree (async, no page reload)
- Completely separate from the Next.js app

---

## Security

- **XSS prevention** — all user text inputs sanitised on change: strips `<>`, `javascript:` URIs, inline event handlers (`on*=`), and non-printable control characters before touching state
- **Email sanitisation** — strips shell-injection characters (`'`, `` ` ``, `;`) and enforces RFC 5321 length limit
- **Phone sanitisation** — allowlist of `[0-9 +\-().]` only
- **Password fields** — intentionally left unsanitised (stripping characters from passwords would silently corrupt them)
- **BCrypt** — passwords are never stored in plaintext; BCrypt with default work factor (10)
- **JWT** — stateless auth, tokens signed with HMAC-SHA256, configurable expiry
- **Spring Security** — auth endpoints public; all other sensitive routes require a valid token
- **CORS** — locked to the app origin; `Authorization` header explicitly allowed
- **Circular dependency** — `PasswordEncoder` extracted to a dedicated `PasswordConfig` to avoid Spring bean initialisation cycle

---

## Key Engineering Decisions

**Why JWT over sessions?**
The backend is designed to scale horizontally — stateless JWT means no shared session store needed between instances.

**Why session-based cart (not user-bound)?**
Lets unauthenticated users build a cart before signing in, reducing friction. The cart is associated with a session ID header (`X-Session-Id`) and can be migrated to a user account post-login.

**Why a separate `PasswordConfig` class?**
Spring Security creates a circular dependency if `PasswordEncoder` lives in `SecurityConfig`: `SecurityConfig → JwtAuthFilter → AuthService → PasswordEncoder → SecurityConfig`. Isolating the bean breaks the cycle cleanly.

**Tailwind v4 `@theme`**
Tailwind v4 strips external `@import url()` from CSS during its build pipeline. Google Fonts and Material Symbols must be loaded via `<link>` in the HTML `<head>` — not from CSS — or they silently fail to load.

**`output: export` removed**
Static export is incompatible with dynamic routes (`[id]`) and live API calls. The app runs as a standard Next.js server.

---

## Project Structure

```
client_app/
├── index.html              # Static coming-soon page (root domain)
├── backend/                # Spring Boot API
│   └── src/main/java/com/gohalal/backend/
│       ├── config/         # Security, CORS, password encoder, data init
│       ├── controller/     # Auth, search, cart, orders, checkout
│       ├── dto/            # Request/response DTOs
│       ├── model/          # JPA entities
│       ├── repository/     # Spring Data repositories
│       ├── security/       # JwtUtil, JwtAuthFilter
│       └── service/        # AuthService, CartService, OrderService, etc.
└── ui/                     # Next.js frontend
    └── app/
        ├── components/     # RestaurantCard, CartDrawer, SideMenu, AddressSearch, ProtectedRoute
        ├── context/        # AuthContext, CartContext
        ├── lib/            # sanitize.ts
        ├── account/        # Account settings (protected)
        ├── careers/
        ├── checkout/
        ├── contact/
        ├── deals/
        ├── login/
        ├── orders/         # (protected)
        ├── partner/
        ├── payment/        # (protected)
        ├── register/
        ├── restaurants/    # Main app shell + [id] restaurant detail
        └── sadaqah-sofra/
```


---

## API Reference (selected endpoints)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/auth/register` | — | Register new user, returns JWT |
| `POST` | `/api/auth/login` | — | Login, returns JWT |
| `GET` | `/api/auth/me` | ✓ | Get current user profile |
| `PUT` | `/api/auth/me` | ✓ | Update profile |
| `POST` | `/api/auth/change-password` | ✓ | Change password |
| `GET` | `/api/restaurants/recommended` | — | Top-rated restaurants |
| `GET` | `/api/restaurants/nearby` | — | Restaurants within radius (lat/lng) |
| `GET` | `/api/restaurants/{id}/menu` | — | Menu items for a restaurant |
| `GET` | `/api/cart` | — | Get cart (session-based) |
| `POST` | `/api/cart/items` | — | Add item to cart |
| `POST` | `/api/orders` | — | Place order |
| `GET` | `/api/orders` | — | Order history |
| `POST` | `/api/checkout/estimate` | — | Calculate total with fees + promo |
| `POST` | `/api/checkout/payment-intent` | — | Create Stripe PaymentIntent |

---

---

## About

Built by [Nour Habibi](https://github.com/nour-habib) · Vancouver, BC  
GoHalal is a real product in development. The codebase is private; this README describes the architecture and engineering decisions.
