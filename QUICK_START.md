# Quick Start - Local Implementation (30 minutes to production)

## Step 1: Clone & Setup (5 min)

```bash
git clone https://github.com/Abisodun/inventory-advisor.git
cd inventory-advisor

# Install Next.js
npx create-next-app@latest . --typescript --tailwind --eslint --skip-git --app 

# When prompted: choose defaults (y for all)

# Install additional deps
npm install @supabase/supabase-js csv-parse
```

## Step 2: Supabase Setup (5 min)

1. Go to https://supabase.com and create a new project
2. Wait for database initialization
3. Go to **Settings > API** and copy your keys
4. Create `.env.local` with:
   ```env
   NEXT_PUBLIC_SUPABASE_URL=your_url_here
   NEXT_PUBLIC_SUPABASE_ANON_KEY=your_key_here
   SUPABASE_SERVICE_ROLE_KEY=your_service_key_here
   ```

5. In Supabase **SQL Editor**, run the entire SQL from SETUP.md

## Step 3: Create App Files (10 min)

Create these files with content from IMPLEMENTATION_SNIPPETS.md:

**lib/supabase.ts**
```typescript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
const supabaseServiceKey = process.env.SUPABASE_SERVICE_ROLE_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey)
export const supabaseAdmin = createClient(supabaseUrl, supabaseServiceKey)
```

**lib/recommendations.ts** - See IMPLEMENTATION_SNIPPETS.md

**app/api/recommendations/route.ts** - See IMPLEMENTATION_SNIPPETS.md

**app/api/upload-csv/route.ts** - See IMPLEMENTATION_SNIPPETS.md

**app/dashboard/page.tsx** - See IMPLEMENTATION_SNIPPETS.md

## Step 4: Test Locally (3 min)

```bash
npm run dev
# Visit http://localhost:3000/dashboard
```

You should see an empty recommendations dashboard.

## Step 5: Deploy to Vercel (7 min)

```bash
npm install -g vercel
vercel
```

Follow prompts, then:

```bash
# Add env vars in Vercel dashboard
vercel env add NEXT_PUBLIC_SUPABASE_URL
vercel env add NEXT_PUBLIC_SUPABASE_ANON_KEY
vercel env add SUPABASE_SERVICE_ROLE_KEY

# Deploy
vercel --prod
```

Your app will be live at `your-app.vercel.app/dashboard`!

---

## Next: Add Test Data

1. Upload a CSV to your dashboard with test data
2. Recommendations will auto-generate
3. See CSV format in IMPLEMENTATION_SNIPPETS.md

## Troubleshooting

**"Module not found" errors?**
```bash
rm -rf node_modules package-lock.json
npm install
```

**Supabase auth errors?**
- Check .env.local has correct keys
- Verify keys are from **Settings > API**, not auth settings

**Vercel deployment fails?**
- Ensure all env vars are set in Vercel dashboard
- Check build logs: `vercel logs`
