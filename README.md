# Haven — AI Real Estate Platform (MERN Stack)

A full MongoDB + Express + React + Node.js implementation of the Haven AI real
estate demo: an AI-guided property finder with match scoring, a cinematic
3D-style room tour, a real embedded map with turn-by-turn directions, budget
and financing calculators, and an AI-assisted seller draft tool.

This is the **real, running application** version — a proper client/server
project instead of a single HTML file. The frontend calls a real Express API
backed by MongoDB instead of holding data in memory.

```
haven-mern/
├── server/          Express API + MongoDB models
│   ├── config/       DB connection
│   ├── models/       Mongoose schemas (Property, User, SavedProperty)
│   ├── controllers/  Route handlers
│   ├── routes/       Express routers
│   ├── middleware/   Auth + error handling
│   ├── utils/        Currency + AI match-scoring logic (shared by controllers)
│   ├── seed/         Example property data + seed script
│   └── server.js     Entry point
└── client/          React (Vite) frontend
    └── src/
        ├── api/       axios wrappers for each API resource
        ├── components/ All UI components
        ├── context/    Global state (currency, saved, compare)
        └── utils/      Currency formatting, SVG art generator, map/directions helpers
```

## 1. Prerequisites

- **Node.js 18+** and npm
- **MongoDB** — either:
  - a local install ([MongoDB Community Server](https://www.mongodb.com/try/download/community)), or
  - a free [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) cluster (recommended if you don't want to install MongoDB locally)

## 2. Setup

```bash
# from the haven-mern/ folder
npm run install:all
```

This installs dependencies for both `server/` and `client/`.

### Configure the backend

```bash
cd server
cp .env.example .env
```

Edit `server/.env`:
- `MONGO_URI` — your local MongoDB URL, or your Atlas connection string
- `JWT_SECRET` — any long random string (used to sign login tokens)
- Leave `PORT` and `CLIENT_URL` as-is unless you change ports

### Configure the frontend

```bash
cd client
cp .env.example .env
```

The default (`VITE_API_URL=http://localhost:5000/api`) works out of the box
if you keep the server's default port.

### Seed the database

```bash
npm run seed
```

(from the `haven-mern/` root — or `npm run seed --prefix server`). This loads
10 example properties into MongoDB. **Run this once** before starting the app,
and again any time you want to reset the data.

## 3. Run it

```bash
# from the haven-mern/ root — runs both server and client together
npm install       # installs `concurrently`, used to run both at once
npm run dev
```

Or run them separately in two terminals:

```bash
npm run dev:server   # http://localhost:5000
npm run dev:client   # http://localhost:5173
```

Open **http://localhost:5173**.

## 4. What's real vs. illustrative — please read this

Same honesty note as the single-file version, because it still applies here:

- **Real**: every property's coordinates, the market pricing (sourced from
  Bayut, Rightmove, Zameen, RentHop, Square Yards, etc. — see the `source`
  field on each property and the link shown in its modal), the embedded
  Google Map, and the directions links (Apple Maps on iOS, Google Maps
  elsewhere).
- **Illustrative**: the specific units themselves. There's no live MLS or
  listings feed wired up, so the exact address/photos/videos for each card
  are stand-ins, not a real for-sale listing. This is clearly labeled in the
  UI ("Illustrative example").
- **Rule-based, not a real LLM (yet)**: `server/utils/matchScoring.js`'s
  `extractRequirements()` is a small regex-based parser standing in for a
  real AI call. See "Wiring in a real LLM" below for how to upgrade it.

## 5. Extending this into a production app

This scaffold is built so each of these is a contained, well-marked swap:

**Wire in a real LLM (e.g. Claude).** Replace the body of
`extractRequirements()` in `server/utils/matchScoring.js` (or better, do this
directly inside `server/controllers/aiController.js`'s `finder` handler) with
a call to the Anthropic Messages API, prompting it to return the same
`{ beds, budgetUSD, city, purpose, lifestyle[] }` JSON shape. Nothing
downstream (scoring, the UI) needs to change. Do the same for
`sellerDraft()`.

**Connect a real listings feed.** Replace `server/seed/seedData.js` with a
script that pulls from a real MLS/IDX API or a partner data feed, mapped to
the `Property` schema in `server/models/Property.js`. Run it on a schedule
(cron / a queue worker) instead of as a one-off seed.

**Add real authentication.** `server/models/User.js`,
`server/controllers/authController.js`, and `server/middleware/auth.js`
already implement JWT register/login/`me`. `SavedProperty` already supports a
logged-in `userId` in addition to the guest-id flow the frontend uses today —
wire up login/register screens in the client and switch `client/src/api/saved.js`
to send the JWT instead of (or alongside) `guestId`.

**Real photos/video per unit.** `Property.media` (`photoUrl`, `videoUrl`,
`videoPoster`) already supports per-property real media — once you have a
real listings feed, point these at real per-listing photography instead of
the shared stock photos/videos used in this demo.

**Live FX rates.** `server/utils/currency.js` and
`client/src/utils/format.js` use a hardcoded rate table. Swap for a real FX
API (e.g. exchangerate.host) and cache the result.

## 6. API reference

| Method | Route | Description |
|---|---|---|
| GET | `/api/properties` | List properties. Query params: `purpose`, `beds`, `city`, `country`, `q` |
| GET | `/api/properties/:id` | Get one property |
| POST | `/api/properties` | Create a property |
| PUT | `/api/properties/:id` | Update a property |
| DELETE | `/api/properties/:id` | Delete a property |
| POST | `/api/ai/finder` | `{ message, purpose }` → structured requirements |
| POST | `/api/ai/match` | `{ requirements }` → ranked, scored, explained matches |
| POST | `/api/ai/seller-draft` | `{ type, beds, features }` → listing headline + description |
| GET | `/api/saved?guestId=` | List saved properties |
| POST | `/api/saved` | `{ guestId, property }` → save one |
| DELETE | `/api/saved/:propertyId?guestId=` | Unsave one |
| POST | `/api/auth/register` | `{ name, email, password }` |
| POST | `/api/auth/login` | `{ email, password }` |
| GET | `/api/auth/me` | Current user (requires `Authorization: Bearer <token>`) |

## 7. Deploying to gethaven.com

> **Before you register:** "GetHaven" is already in active use as a brand
> name by an unrelated, funded product (a crypto/privacy shopping app at
> `gethaven.app`, with its own social presence). The `.com` itself may or may
> not be free — check availability yourself at a registrar, since that isn't
> something that can be verified from here — but it's worth being aware of
> the naming overlap before you invest in branding, business cards, trademark
> filings, etc. around it.

Once you've registered the domain, this app is two separate deployables (a
static React build + a Node API), so a typical setup looks like:

| Piece | Suggested host | Domain |
|---|---|---|
| `client/` (React build) | [Vercel](https://vercel.com) or [Netlify](https://netlify.com) | `gethaven.com` (root domain) |
| `server/` (Express API) | [Render](https://render.com), [Railway](https://railway.app), or [Fly.io](https://fly.io) | `api.gethaven.com` (subdomain) |
| Database | [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) (free tier is fine to start) | — no domain needed, just a connection string |

**Steps:**

1. **Database:** create a free MongoDB Atlas cluster, get its connection
   string, and use it as `MONGO_URI` in the next step.
2. **Backend:** deploy `server/` to Render/Railway/Fly. Set its environment
   variables in that platform's dashboard: `MONGO_URI` (from step 1),
   `JWT_SECRET` (a long random string), `CLIENT_URL=https://gethaven.com`.
   Once deployed, point `api.gethaven.com` at it — Render/Railway both give
   you a CNAME target to add in your DNS provider, and issue a free HTTPS
   cert automatically once that DNS resolves.
3. **Frontend:** deploy `client/` to Vercel/Netlify. Set
   `VITE_API_URL=https://api.gethaven.com/api` as an environment variable in
   that platform's dashboard (not just in a local `.env` — Vercel/Netlify
   read their own env var settings at build time). Point `gethaven.com` (and
   optionally `www.gethaven.com`) at it the same way — both platforms walk
   you through the DNS records in their UI and handle HTTPS automatically.
4. **Seed production data:** run `npm run seed` once, pointed at the
   production `MONGO_URI` (e.g. by temporarily setting it in `server/.env`
   locally, or via your host's one-off task/shell feature).
5. **Verify CORS:** with `CLIENT_URL` set correctly in step 2, the API will
   only accept requests from `https://gethaven.com`. If you're also serving
   `www.gethaven.com`, set
   `CLIENT_URL=https://gethaven.com,https://www.gethaven.com` — the server
   already supports a comma-separated list, no code changes needed.

## 8. What was verified before delivery

- Every backend file passes `node --check` (syntax-valid)
- `npm install` succeeds cleanly in both `server/` and `client/`
- The server boots correctly and attempts a real MongoDB connection (it will
  fail until you provide a real `MONGO_URI` — that's expected)
- The full React app builds with `vite build` with **zero errors** across all
  113 modules

What was **not** possible to verify in this environment: an actual live
MongoDB connection and a full click-through in a real browser (no MongoDB
server or browser was available in the sandbox this was built in). Please
run `npm run dev` locally and sanity-check the flow end to end before relying
on it.


For Demo Check 
https://getshaven.vercel.app/
