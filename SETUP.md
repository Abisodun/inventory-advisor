# Inventory Advisor - Complete Implementation Guide

## Phase 1: Quick Local Setup (15 minutes)

### 1. Initialize Next.js project

```bash
git clone https://github.com/Abisodun/inventory-advisor.git
cd inventory-advisor
npx create-next-app@latest . --typescript --tailwind --eslint --skip-git --app
```

When prompted:
- ESLint: Yes
- Tailwind: Yes  
- src/ directory: No
- App Router: Yes (already done)
- Turbopack: No

### 2. Install dependencies

```bash
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs csv-parse
npm install -D @types/node @types/react @types/react-dom typescript
```

## Phase 2: Supabase Setup (10 minutes)

### 1. Create Supabase project

1. Go to [supabase.com](https://supabase.com)
2. Sign up or log in
3. Create a new project (Organization: personal, Name: inventory-advisor)
4. Wait for initialization
5. Go to **Settings > API** and copy:
   - Project URL → `NEXT_PUBLIC_SUPABASE_URL`
   - Anon Key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - Service Role Key → `SUPABASE_SERVICE_ROLE_KEY`

### 2. Create `.env.local`

```env
NEXT_PUBLIC_SUPABASE_URL=your_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
```

### 3. Run schema migration

Go to Supabase dashboard → SQL Editor → New Query, paste this entire block and execute:

```sql
-- Enable UUID
create extension if not exists "uuid-ossp";

-- Retailers
create table public.retailers (
  id uuid primary key default gen_random_uuid(),
  shop_domain text unique not null,
  shopify_token text not null,
  name text,
  timezone text default 'America/Winnipeg',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Locations
create table public.locations (
  id uuid primary key default gen_random_uuid(),
  retailer_id uuid references public.retailers(id) on delete cascade not null,
  external_id text not null,
  name text not null,
  kind text check (kind in ('store','warehouse','online')) default 'store',
  is_active boolean default true,
  created_at timestamptz default now()
);

-- SKUs
create table public.skus (
  id uuid primary key default gen_random_uuid(),
  retailer_id uuid references public.retailers(id) on delete cascade not null,
  external_id text not null,
  name text not null,
  category text,
  cost numeric(12,2),
  price numeric(12,2),
  is_active boolean default true,
  created_at timestamptz default now()
);

-- Vendors
create table public.vendors (
  id uuid primary key default gen_random_uuid(),
  retailer_id uuid references public.retailers(id) on delete cascade not null,
  name text not null,
  lead_time_days_default integer default 7,
  moq_default integer default 0,
  created_at timestamptz default now()
);

-- SKU-Vendor
create table public.sku_vendor (
  id uuid primary key default gen_random_uuid(),
  sku_id uuid references public.skus(id) on delete cascade not null,
  vendor_id uuid references public.vendors(id) on delete cascade not null,
  lead_time_days integer,
  moq integer,
  pack_size integer,
  unique (sku_id, vendor_id)
);

-- SKU-Location config
create table public.sku_locations (
  id uuid primary key default gen_random_uuid(),
  sku_id uuid references public.skus(id) on delete cascade not null,
  location_id uuid references public.locations(id) on delete cascade not null,
  safety_stock_days_target integer default 7,
  unique (sku_id, location_id)
);

-- Snapshots
create table public.sku_location_snapshots (
  id bigserial primary key,
  sku_id uuid references public.skus(id) on delete cascade not null,
  location_id uuid references public.locations(id) on delete cascade not null,
  date date not null,
  units_sold numeric default 0,
  units_returned numeric default 0,
  on_hand numeric default 0,
  on_order numeric default 0,
  created_at timestamptz default now(),
  unique (sku_id, location_id, date)
);

-- Settings
create table public.retailer_settings (
  retailer_id uuid primary key references public.retailers(id) on delete cascade,
  default_buffer_days integer default 7,
  min_velocity_threshold numeric default 0.05,
  overstock_multiplier numeric default 3,
  updated_at timestamptz default now()
);

-- Recommendation type
create type public.recommendation_type as enum ('order_more','order_less','move');

-- Recommendations
create table public.recommendations (
  id uuid primary key default gen_random_uuid(),
  retailer_id uuid references public.retailers(id) on delete cascade not null,
  sku_id uuid references public.skus(id),
  source_location_id uuid references public.locations(id),
  dest_location_id uuid references public.locations(id),
  type public.recommendation_type not null,
  recommended_qty integer not null,
  rationale text,
  metric_days_of_stock numeric,
  metric_target_days numeric,
  metric_avg_daily_units numeric,
  status text check (status in ('open','accepted','snoozed','dismissed')) default 'open',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Indexes
create index idx_locations_retailer on public.locations(retailer_id);
create index idx_skus_retailer on public.skus(retailer_id);
create index idx_snapshots_date on public.sku_location_snapshots(date);
create index idx_snapshots_sku_location on public.sku_location_snapshots(sku_id, location_id);
create index idx_recommendations_retailer on public.recommendations(retailer_id);
create index idx_recommendations_status on public.recommendations(status);
create index idx_recommendations_created on public.recommendations(created_at desc);
```

## Phase 3: Create App Files

Create these files with the code provided in the next sections:

- `lib/supabase.ts` - Supabase client
- `lib/recommendations.ts` - Core recommendation logic
- `app/api/recommendations/route.ts` - API endpoint
- `app/api/upload-csv/route.ts` - CSV importer
- `app/dashboard/page.tsx` - Main UI
- `app/layout.tsx` - Root layout
- `app/globals.css` - Tailwind config

## Phase 4: Test Locally

```bash
npm run dev
# Visit http://localhost:3000/dashboard
```

## Phase 5: Deploy to Vercel

```bash
npm install -g vercel
vercel
```

When prompted to link project, click to create new project. Vercel will auto-detect Next.js.

Then add environment variables in Vercel dashboard:
- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`

Deploy: `vercel --prod`

---

# Code Files

See individual files in `/docs` folder or continue reading below for complete implementation.
