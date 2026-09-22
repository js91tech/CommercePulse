# CommercePulse

Automated market research, supply–demand scoring, and warehouse-aware supplier matching for **digital** and **physical** dropshipping businesses.

The app runs on a laptop with **no API keys**. Optional OpenAI, Anthropic, and SerpAPI keys enrich summaries and shopping listings when present.

## What it does

1. **Hybrid catalog** — Digital products (instant file + license keys) and physical dropship SKUs (weight, shipping, inventory).
2. **Research engine** — Keyword in, viability score (0–100), recommended prices, and consumer personas out.
3. **Demand vs supply** — Search volume, social momentum, ad activity, competitor density, saturation rating, and a **Buy / Watch / Pass** verdict.
4. **Supplier directory** — Filter US / UK / EU local warehouses (2–5 day ship) vs China / global nodes, with MOQ and return policy.
5. **Product builder** — One form that exports Shopify or WooCommerce CSV.

Live marketplace scraping of Amazon / TikTok / Etsy is intentionally **not** implemented. Those surfaces block unauthorized scraping. CommercePulse uses a deterministic scoring model, public Google News RSS, and optional SerpAPI instead.

## Stack

- Next.js 15 (App Router, Server Actions)
- Tailwind CSS + shadcn/ui
- Prisma ORM + PostgreSQL (Neon on Vercel, or Docker locally)
- OpenAI GPT-4o / Anthropic Claude for optional narrative summaries
- Cheerio for public RSS; SerpAPI when `SERPAPI_KEY` is set

## Local setup

Postgres is required. The fastest local option is Docker:

```bash
docker compose up db -d
cp .env.example .env
```

In `.env`, use the Docker URLs (or a Neon non-pooling URL for both values):

```
DATABASE_URL="postgresql://commercepulse:commercepulse@localhost:5432/commercepulse"
DIRECT_URL="postgresql://commercepulse:commercepulse@localhost:5432/commercepulse"
```

Then:

```bash
npm install
npm run db:setup
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

`npm run db:setup` generates the Prisma client, pushes the schema, and seeds products, suppliers, and sample research reports.

### Optional keys (`.env`)

```
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
SERPAPI_KEY=
```

If they are empty, mock fallbacks still produce full reports.

## Project layout

```
app/
  page.tsx                 Dashboard + research bar
  research/                Report list + detail
  demand/                  Demand vs supply analyzer
  suppliers/               Filterable warehouse directory
  catalog/                 Digital / physical catalog
  builder/                 CSV export builder
  api/research|suppliers|demand|products|export
lib/
  scoring.ts               Viability model
  research-engine.ts       Orchestrates mock + live + LLM
  supplier-engine.ts       Local vs overseas matching
  csv-export.ts            Shopify / WooCommerce
prisma/schema.prisma       Product, Supplier, MarketTrend, ResearchReport
```

## Hosting

Suggested path:

1. **GitHub** for source.
2. **Vercel** for the web app (hobby plan is enough for a demo).
3. **Neon Postgres** via Vercel Storage (or neon.tech).

### Vercel + Neon (required for production)

Prisma is already on PostgreSQL with `DATABASE_URL` + `DIRECT_URL`. `npm run build` runs `prisma generate && prisma db push && next build --turbopack`, so the schema is applied during each Vercel build.

1. Import this repository in Vercel (root directory = repo root).
2. In the Vercel project: **Storage → Create Database → Neon**.
3. Copy the Neon **non-pooling** (direct) connection string.
4. Set both env vars to that same non-pooling URL:
   - `DATABASE_URL`
   - `DIRECT_URL`
5. Optional env vars: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `SERPAPI_KEY`.
6. Deploy. After the first successful build, run `npx prisma db seed` against Neon (or hit `/api/research` to generate reports).

You can later point `DATABASE_URL` at Neon’s pooled URL and keep `DIRECT_URL` on the non-pooling URL. Until then, using the non-pooling URL for both is the simplest setup.

### Other hosts

- **Railway / Render / Fly.io** — Next.js app + a Postgres addon. Use `npm run start` after `npm run build`.
- **Docker** — `docker compose up db` for Postgres, then point `DATABASE_URL` and `DIRECT_URL` at `postgresql://commercepulse:commercepulse@localhost:5432/commercepulse`.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/research` | Run + persist a research report |
| `GET` | `/api/research` | List reports |
| `GET` | `/api/demand?query=` | Demand / supply snapshot |
| `GET` | `/api/suppliers?region=LOCAL&q=` | Supplier search |
| `POST` | `/api/products` | Create catalog SKU |
| `POST` | `/api/products/:id/license` | Issue a digital license key |
| `POST` | `/api/export` | Shopify or WooCommerce CSV |
| `GET` | `/api/health` | Database + key status |

## Scripts

```bash
npm run dev          # Next.js on :3000
npm run db:setup     # generate + push + seed
npm run lint
npm run build        # generate + db push + Next.js (used on Vercel)
```
