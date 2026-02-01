# Deploy Inventory Advisor - 30 Minute Build-to-Production

## 🚀 TL;DR - Run This Now

```bash
# 1. Clone and enter directory
git clone https://github.com/Abisodun/inventory-advisor.git
cd inventory-advisor

# 2. Follow QUICK_START.md step by step (5+5+10+3+7 = 30 min)
# This will get you from zero to deployed
```

## What You'll Have After 30 Minutes

✅ Live product: `yourdomain.vercel.app/dashboard`
✅ CSV data importer working
✅ Recommendation engine running
✅ Connected to Supabase database
✅ Ready for first merchant to upload data

## Architecture

```
Your Merchant
    ↓
   [CSV Upload to Dashboard]
    ↓
  Vercel (Next.js API)
    ↓
 Supabase (Postgres + RLS)
    ↓
Recommendations Engine
    ↓
  [Display in Dashboard]
```

## Files to Know

- **README.md** - Project overview
- **QUICK_START.md** - Step-by-step (START HERE)
- **SETUP.md** - Detailed setup with SQL schema
- **docs/CODEBASE.md** - All code snippets
- **package.json** - Dependencies

## Next Phase (After Deployment)

### Phase 2: Shopify Integration (optional)
- Replace CSV importer with Shopify API
- Auto-sync daily orders & inventory
- Use Edge Function for scheduled sync

### Phase 3: Advanced Features
- Email alerts for stockouts
- ABC classification for focus
- Seasonality modeling
- Multi-warehouse transfer optimization

## Support / Issues

If something breaks during setup:

1. Check `.env.local` has 3 keys from Supabase
2. Verify SQL schema executed completely
3. Run `npm install` again if module errors
4. Check Vercel logs: `vercel logs`

## Cost Breakdown (2025)

- **Supabase**: Free tier covers 1-2 locations, 5K-10K SKUs
- **Vercel**: Free tier (up to $20/month if needed)
- **Total for small retailer**: $0-20/month

Scales to $50-100/month at 5+ locations, 50K SKUs.

---

**Ready?** Go to QUICK_START.md and follow the 5 phases.
