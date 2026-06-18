# Scout

**Find your next shot. Share where you stood.**

Scout is a photographer-focused spot-discovery platform — find, view, review, and save great photo locations. Browse a searchable feed of spots backed by real photos from other photographers, see them on a map, and check the details that actually matter before you go: best light, gear, access, crowds, and seasons.

This repository is the **whole project**: a native **SwiftUI iOS app** (`Scout-iOS/`) talking to a **FastAPI backend on Google Cloud Run** (`Scout-backend/`) over a versioned REST API, with Firebase providing auth, data, and photo storage.

> Each side has its own deep-dive repositories — [`Scout IOS`](https://github.com/kkennethsieu/Scout-iOS) (iOS) and [`Scout Backend`](https://github.com/kkennethsieu/Scout-Backend) (backend). This document is the system-level overview that ties them together.

---

## Screenshots

<p align="center">
  <img src="marketing/out/01_explore.png"     width="230" alt="Explore — Find your next shot">
  <img src="marketing/out/02_spot_detail.png" width="230" alt="Spot Detail — Know before you go">
  <img src="marketing/out/03_map.png"          width="230" alt="Map — Every spot on the map">
</p>
<p align="center">
  <img src="marketing/out/04_saved.png"   width="230" alt="Saved — Build your shot list">
  <img src="marketing/out/05_create.png"  width="230" alt="Create — Share where you stood">
  <img src="marketing/out/06_review.png"  width="230" alt="Review — Rate it. Review it.">
  <img src="marketing/out/07_profile.png" width="230" alt="Profile — Your spots, your story">
</p>

---

## What it does

- **Explore** — a searchable, filterable, sortable feed of spots, each card a swipeable photo carousel. Paginated and geo-scoped to your location or a place you pick.
- **Map** — spots as MapKit pins, with a "search this area" re-query, recenter-to-user, and tap-to-preview.
- **Spot detail** — hero photos, quick facts, best shooting times, gear & composition tips, access logistics (permits / drone / tripod / fee), and reviews.
- **Create & review** — add a spot from a photo's EXIF location or your current location, drop a pin, rate it, and contribute photos, notes, gear, and tips. New spots and their first review are submitted atomically.
- **Saved** — organize spots into lists (with a protected system "Favorites" list); save a spot to multiple lists at once.
- **Profile** — your reviews, review count, editable profile, and account deletion.
- **Auth** — Sign in with Apple, Google, or email (Firebase).

---

## System architecture

```
┌─────────────────────────┐         HTTPS + Bearer (Firebase ID token)        ┌──────────────────────────┐
│      Scout iOS app       │  ───────────────────────────────────────────────▶ │   Scout backend (FastAPI) │
│      (SwiftUI, MVVM)      │              REST /api  · multipart uploads        │     on Google Cloud Run   │
│                          │ ◀─────────────────────────────────────────────── │                          │
│  • Firebase Auth SDK     │              JSON (snake_case, ISO-8601)            │  • Verifies ID tokens     │
│  • URLSession client     │                                                    │  • Rate limit + App Check │
└─────────────────────────┘                                                    └────────────┬─────────────┘
            │                                                                                │
            │ authenticates directly                                                         │ Admin SDK
            ▼                                                                                ▼
   ┌──────────────────┐                                                    ┌─────────────────────────────────┐
   │  Firebase Auth   │  ◀── same project ──▶                              │  Firestore (spots, reviews,     │
   │ (Apple/Google/   │                                                    │  users, per-user lists)         │
   │   email)         │                                                    │  Cloud Storage (review/avatar   │
   └──────────────────┘                                                    │  photos, public read)           │
                                                                           │  Google Geocoding API           │
                                                                           └─────────────────────────────────┘
```

**The contract between the two halves:**

- The app authenticates **directly** with Firebase Auth and never sees a password on the backend. Every API call carries the Firebase **ID token** as `Authorization: Bearer <token>`; the backend verifies it with the Admin SDK and derives the user from the token's `uid`.
- The **backend owns all user records and data.** The app only authenticates, forwards the token, and renders responses.
- Errors come back as a consistent `{ detail, code }` body so the client can branch on a stable `code` (e.g. `SPOT_ALREADY_EXISTS` carries the existing spot so the app can deep-link to it).

---

## The two halves

### 📱 iOS app — `Scout/`

SwiftUI for iPhone (deployment target **iOS 26.1**), MVVM with `@Observable @MainActor` view models.

- **App** — `ScoutApp` (entry, configures Firebase), `RootView` (auth gate: splash → main / login), `MainTabView` (a **custom tab bar** that keeps every tab's scroll position and nav stack alive, not a stock `TabView`).
- **Screens** — feature areas (Auth, Explore, Map, Spot Detail, Create, Saved, Search, Profile), each with a local `Components/` folder.
- **Service layer** — each data domain (`SpotService`, `UserService`, `SavedListService`, `LegalService`) is a protocol with a **Live** implementation (URLSession + Bearer token via shared `BackendClient`) and a **Mock** implementation for previews/tests. `AppServices` is the default dependency wired into the app.
- **Design system** — `s`-prefixed tokens (`Color.sAccent`, `Font.sHeadingM`, `Spacing`/`Radius` enums). Forest-green accent on warm off-white; no raw hex or magic-number padding.
- **Tests** — Swift Testing (`import Testing`, `@Test`, `#expect`) in `ScoutTests/`, injecting mock services to verify view-model logic (pagination, filtering/sorting, submission payloads, EXIF parsing) without the network.

Dependencies via Swift Package Manager: **Firebase Auth** and **GoogleSignIn**. Also uses **MapKit**, **CoreLocation**, and **PhotosUI / ImageIO** (reads EXIF GPS from picked photos with no photo-library permission).

### 🖥️ Backend — `Scout-backend/`

FastAPI (Python 3.12) served by Uvicorn, containerized on **Google Cloud Run** (project `scout-497021`, region `us-west2`).

- **`app/main.py`** — app, lifespan (Firebase init), CORS, exception handlers (domain errors → `{detail, code}`), router wiring.
- **`app/core/`** — config, Firebase init, security (Firebase token + App Check verification), per-user rate limiting, error types.
- **`app/api/v1/`** — versioned routers: `spots`, `reviews`, `users`, `lists`, `legal` + shared deps.
- **`app/services/`** — Firestore/Storage access, reverse-geocoding new spots (Google Geocoding API), aggregate computation (averaging ratings/access/crowd/fee across reviews), an in-process spot cache for nearby/search scans.
- **`app/schemas/`** — Pydantic request/response validation, including the constrained review vocabularies.
- **Data** — **Firestore** (spots, reviews, users, per-user `lists` subcollections) and **Cloud Storage** (review and avatar photos, EXIF-stripped, served via public URLs).
- **Tests** — pytest unit + integration suites run against the **Firebase Emulator Suite**.

---

## API surface (at a glance)

All endpoints except `/`, `/health`, and `/legal` require a Firebase Bearer token.

| Area | Endpoints |
| :--- | :--- |
| **Health / legal** | `GET /health`, `GET /legal` |
| **Users** | `GET /users/me`, `PATCH /users/me` (multipart profile + avatar), `DELETE /users/me`, `GET /users/me/reviews` |
| **Spots** | `GET /spots` (nearby), `GET /spots/search` (by name), `GET /spots/{id}`, `GET /spots/{id}/reviews` |
| **Reviews** | `POST /spots/{id}/reviews`, `POST /spots/with-review` (new spot + first review atomically), `GET /reviews/{id}`, `DELETE /reviews/{id}` |
| **Lists** | `GET /users/me/lists` (`{lists, memberships}` snapshot), `POST`/`PATCH`/`DELETE /users/me/lists[/{id}]`, `GET /users/me/lists/{id}/spots`, `PATCH /users/me/spots/{spot_id}/lists` (the single membership write path) |

See [`Scout Backend`](https://github.com/kkennethsieu/Scout-Backend) for the full contract, error codes, and review-submission vocabularies.

---

## Security model

- **Auth** — direct Firebase sign-in on the client; the backend only ever sees and verifies the ID token. Missing/invalid → `401`.
- **Authorization** — user-scoped resources are enforced by path (`/users/me/...`); review deletion checks authorship (`403 FORBIDDEN`).
- **Rate limiting** — per-user (keyed on Firebase uid) limits on write endpoints, returning `429 RATE_LIMITED` with `Retry-After`.
- **App Check** — requests may carry a Firebase App Check token; currently log-only (`APP_CHECK_ENFORCED=false`) until the iOS app ships the SDK.
- **Firestore/Storage rules** are deny-all for direct client access — everything goes through the authenticated API.
- **Secrets** (e.g. the Geocoding API key) live in **Secret Manager**, never in env vars or commits. `GoogleService-Info.plist` is gitignored on the iOS side.

---

## Running the whole thing locally

You generally run the backend (with Firebase emulators) and the iOS app on the Simulator together.

- Add your own `GoogleService-Info.plist` (gitignored) and set your signing team.
- Point `BackendClient.baseURL` at your local backend (`http://127.0.0.1:8000`) for Simulator development, or leave it on the hosted Cloud Run URL.
- Build/run on an **iPhone 17** Simulator (the iPhone 16 Simulator is often not installed).

> The iOS Simulator shares the Mac's network, so `localhost:8000` is reachable. A physical device can't reach localhost — expose the backend with `ngrok http 8000` (or `make dev-device`) and point `baseURL` at that URL.

---

## Deployment

- **Backend** — GitHub Actions runs the test suite on every PR; **merging to `main` auto-deploys** to Cloud Run via Workload Identity Federation (`gcloud run deploy scout-backend --source .`). Legal pages (`public/`) deploy to Firebase Hosting.
- **iOS** — built and signed in Xcode for TestFlight / the App Store.

---

## Repository layout

```
Scout-Overview/
├── README.md          ← you are here (system overview)
├── Scout/             iOS app (SwiftUI) — see Scout/README.md
└── Scout-backend/     FastAPI backend on Cloud Run — see Scout-backend/README.md
```
