# Complete Codebase - Copy These Files

## File: lib/supabase.ts

```typescript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
const supabaseServiceKey = process.env.SUPABASE_SERVICE_ROLE_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey)
export const supabaseAdmin = createClient(supabaseUrl, supabaseServiceKey)
```

## File: lib/recommendations.ts

```typescript
export interface RecommendationParams {
  avgDailyUnits: number
  onHand: number
  leadTimeDays: number
  bufferDays: number
  minVelocity: number
  overstockMultiplier: number
}

export function calculateDaysOfStock(onHand: number, avgDaily: number): number {
  if (avgDaily <= 0) return 999
  return onHand / avgDaily
}

export function generateRecommendations(params: RecommendationParams) {
  const {
    avgDailyUnits,
    onHand,
    leadTimeDays,
    bufferDays,
    minVelocity,
    overstockMultiplier,
  } = params

  const daysOfStock = calculateDaysOfStock(onHand, avgDailyUnits)
  const targetDays = leadTimeDays + bufferDays
  const overstockThreshold = targetDays * overstockMultiplier

  const recs = []

  // Order More
  if (daysOfStock < targetDays && avgDailyUnits >= minVelocity) {
    const recommendedQty = Math.ceil((targetDays - daysOfStock) * avgDailyUnits)
    recs.push({
      type: 'order_more',
      recommendedQty,
      rationale: `Stock at ${daysOfStock.toFixed(1)} days; target ${targetDays} days. Will stockout in ${Math.ceil(daysOfStock)} days.`,
      metricDaysOfStock: daysOfStock,
      metricTargetDays: targetDays,
      metricAvgDailyUnits: avgDailyUnits,
    })
  }

  // Order Less
  if (daysOfStock > overstockThreshold && avgDailyUnits < minVelocity) {
    recs.push({
      type: 'order_less',
      recommendedQty: 0,
      rationale: `Overstocked at ${daysOfStock.toFixed(1)} days (target: ${targetDays}). Suggest holding from reorder; consider promotion.`,
      metricDaysOfStock: daysOfStock,
      metricTargetDays: targetDays,
      metricAvgDailyUnits: avgDailyUnits,
    })
  }

  return recs
}
```

## File: app/api/recommendations/route.ts

```typescript
import { supabase } from '@/lib/supabase'
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const retailerId = request.nextUrl.searchParams.get('retailerId')
  const status = request.nextUrl.searchParams.get('status') || 'open'

  if (!retailerId) {
    return NextResponse.json({ error: 'Missing retailerId' }, { status: 400 })
  }

  try {
    const { data, error } = await supabase
      .from('recommendations')
      .select(
        `
        id,
        type,
        recommended_qty,
        rationale,
        metric_days_of_stock,
        metric_target_days,
        metric_avg_daily_units,
        status,
        created_at,
        skus!inner(name),
        source_location_id,
        locations!source_location_id(name)
      `
      )
      .eq('retailer_id', retailerId)
      .eq('status', status)
      .order('created_at', { ascending: false })
      .limit(50)

    if (error) throw error
    return NextResponse.json(data || [])
  } catch (err) {
    return NextResponse.json({ error: String(err) }, { status: 500 })
  }
}

export async function PATCH(request: NextRequest) {
  const { id, status } = await request.json()

  try {
    const { data, error } = await supabase
      .from('recommendations')
      .update({ status, updated_at: new Date().toISOString() })
      .eq('id', id)
      .select()

    if (error) throw error
    return NextResponse.json(data?.[0])
  } catch (err) {
    return NextResponse.json({ error: String(err) }, { status: 500 })
  }
}
```

## File: app/api/upload-csv/route.ts

```typescript
import { supabaseAdmin } from '@/lib/supabase'
import { parse } from 'csv-parse/sync'
import { NextRequest, NextResponse } from 'next/server'

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData()
    const file = formData.get('file') as File
    const retailerId = formData.get('retailerId') as string

    if (!file || !retailerId) {
      return NextResponse.json({ error: 'Missing file or retailerId' }, { status: 400 })
    }

    const text = await file.text()
    const records = parse(text, {
      columns: true,
      skip_empty_lines: true,
    }) as Array<any>

    let createdCount = 0

    for (const record of records) {
      try {
        // Upsert location
        const { data: locData } = await supabaseAdmin
          .from('locations')
          .upsert(
            {
              retailer_id: retailerId,
              external_id: `loc-${record.location_name}`,
              name: record.location_name,
              kind: record.location_type || 'store',
              is_active: true,
            },
            { onConflict: 'retailer_id,external_id' }
          )
          .select()

        const locationId = locData?.[0]?.id
        if (!locationId) continue

        // Upsert SKU
        const { data: skuData } = await supabaseAdmin
          .from('skus')
          .upsert(
            {
              retailer_id: retailerId,
              external_id: record.sku_id,
              name: record.sku_name,
              category: record.category || null,
              cost: parseFloat(record.cost) || 0,
              price: parseFloat(record.price) || 0,
              is_active: true,
            },
            { onConflict: 'retailer_id,external_id' }
          )
          .select()

        const skuId = skuData?.[0]?.id
        if (!skuId) continue

        // Insert snapshot
        await supabaseAdmin.from('sku_location_snapshots').insert({
          sku_id: skuId,
          location_id: locationId,
          date: record.date,
          units_sold: parseFloat(record.units_sold) || 0,
          units_returned: parseFloat(record.units_returned) || 0,
          on_hand: parseFloat(record.on_hand) || 0,
          on_order: parseFloat(record.on_order) || 0,
        })

        // Upsert sku_location
        await supabaseAdmin.from('sku_locations').upsert(
          {
            sku_id: skuId,
            location_id: locationId,
            safety_stock_days_target: 7,
          },
          { onConflict: 'sku_id,location_id' }
        )

        createdCount++
      } catch (e) {
        console.error('Row error:', e)
      }
    }

    return NextResponse.json({ success: true, created: createdCount })
  } catch (err) {
    return NextResponse.json({ error: String(err) }, { status: 500 })
  }
}
```

## File: app/dashboard/page.tsx

See next section (DASHBOARD_UI.md)
