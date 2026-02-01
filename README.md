# Inventory Advisor

A lightweight demand & reorder assistant that plugs into POS/e-commerce data and surfaces actionable recommendations instead of dashboards.

## Overview

**Inventory Advisor** is designed for small retailers (1–5 locations, 100–5,000 SKUs) who need:
- **Order More**: Which items are at risk of stockout in the next N days?
- **Order Less**: Which items are overstocked and should be held from reordering?
- **Move Stock**: Which items should be transferred between locations to optimize fill rates?

Instead of a full ERP, this tool provides a **daily decision queue** of prioritized actions per SKU/location.

## Tech Stack

- **Frontend**: Next.js 14 + React + Tailwind CSS
- **Backend**: Supabase (Postgres) + Edge Functions (serverless)
- **Data Ingestion**: CSV importer (CSV) or Shopify API integration (coming soon)
- **Deployment**: Vercel

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/Abisodun/inventory-advisor.git
cd inventory-advisor
npm install
```

### 2. Set up Supabase

- Create a new project at [supabase.com](https://supabase.com)
- Run the SQL schema (see `docs/schema.sql`)
- Copy your `SUPABASE_URL` and `SUPABASE_ANON_KEY` from the project settings

### 3. Environment variables

Create `.env.local`:

```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
```

### 4. Run locally

```bash
npm run dev
# Open http://localhost:3000
```

### 5. Deploy to Vercel

```bash
npm install -g vercel
vercel
```

Add the same environment variables in Vercel's project settings.

## Data Model

### Core Tables

- `retailers`: Merchant accounts
- `locations`: Store/warehouse/online 
- `skus`: Products
- `sku_location_snapshots`: Daily sales & inventory data
- `recommendations`: Actionable insights (order_more, order_less, move)

## MVP Features

1. **CSV Importer**: Upload historical sales & inventory data
2. **Recommendations Dashboard**: Prioritized list of actions
3. **Status Tracking**: Accept, snooze, or dismiss recommendations
4. **Export Actions**: Generate PO drafts or transfer lists

## Recommendation Logic

### Order More
- Condition: `days_of_stock < lead_time + buffer` AND `avg_daily_units > min_velocity`
- Suggested qty: `(target_days - current_days) * avg_daily_units`, rounded to pack size

### Order Less
- Condition: `days_of_stock > 3 * target_days` AND `velocity < threshold`
- Action: Hold from reordering; consider promo/markdown

### Move Stock
- Condition: One location understocked, another overstocked (same SKU)
- Action: Transfer units from surplus to deficit location

## Project Structure

```
inventory-advisor/
├── app/
│   ├── api/
│   │   ├── recommendations/       # Fetch & update recommendations
│   │   └── upload-csv/            # CSV importer
│   ├── dashboard/
│   │   └── page.tsx               # Main UI
│   └── layout.tsx
├── lib/
│   ├── supabase.ts                # Supabase client
│   └── recommendations.ts         # Core logic
├── supabase/
│   ├── functions/
│   │   └── recompute-recommendations/  # Nightly job
│   └── migrations/
│       └── 001_initial_schema.sql
├── docs/
│   └── schema.sql                 # Database schema
└── package.json
```

## Next Steps

- [ ] CSV importer with test data
- [ ] Shopify API integration
- [ ] Email alerts for stockouts
- [ ] A/B classification for focus
- [ ] Seasonality modeling
- [ ] Mobile app

## License

MIT
