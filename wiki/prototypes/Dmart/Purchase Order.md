

Create purchase order for these store STR1022 – Borivali, Mumbai with SKU10041 please 




create a purchase order for DMart Saheed Nagar Bhubaneswar for next month
- "Create a purchase order for 2026-09-07 for store 10, supplier 3"
- "Raise a PO for the recommended items this week"
- "Issue and send a purchase order for store 10"

Raise the PO for **STR1006** for critical priority please





Here's everything you need — nothing has been executed. This is **synthetic demo seed data**, grounded on real `sku_sid`/`loc_sid` combinations from the inventory/recommendation views, with clearly-marked placeholder unit costs.

### ⚠️ Read first
- **Unit costs are synthetic placeholders.** Replace them with real vendor-quoted costs before any PO built from this data is transmitted to a supplier.
- **Both steps below require a write-enabled session.** This current session is read-only (DML fails with `399517`), so the seed script and any subsequent PO creation must run under a role/warehouse that permits DML on `DMART_INDIA_DB.GOLD`.
- IDs are namespaced to **9001–9020** to avoid colliding with existing recommendations/approvals/POs.

---

### 1) End-to-end seed script (20 recommendations + matching approvals)

```sql
-- ============================================================
-- DMART DEMO SEED — 20 synthetic purchase recommendations + approvals
-- ⚠️ SYNTHETIC/DEMO DATA. ESTIMATED_UNIT_COST is a PLACEHOLDER — replace
--    with real vendor-quoted costs before transmitting any PO to a supplier.
-- Requires a WRITE-ENABLED session (DML on DMART_INDIA_DB.GOLD).
-- Idempotent: re-running cleans and re-seeds IDs 9001-9020.
-- ============================================================
USE DATABASE DMART_INDIA_DB;
USE SCHEMA GOLD;

-- 0) Clean any prior demo seed --------------------------------
DELETE FROM PURCHASE_APPROVAL       WHERE APPROVAL_ID       BETWEEN 9001 AND 9020;
DELETE FROM PURCHASE_RECOMMENDATION WHERE RECOMMENDATION_ID BETWEEN 9001 AND 9020;

-- 1) Build 20 grounded rows from real SKU/store combinations --
CREATE OR REPLACE TEMPORARY TABLE TMP_DEMO_SEED_20 AS
WITH base AS (
  SELECT
      sku_sid, sku_id, sku_ldesc, loc_sid, loc_id, loc_name,
      COALESCE(CEIL(recommended_order_qty), 30) AS rec_qty,
      ROUND(COALESCE(reorder_point, 20))        AS reorder_pt,
      ROUND(COALESCE(order_up_to_qty, 60))      AS order_up_to,
      COALESCE(lead_time_days, 7)               AS lead_days,
      priority, stock_status,
      ROW_NUMBER() OVER (
        ORDER BY CASE stock_status
                   WHEN 'OUT-OF-STOCK'   THEN 1
                   WHEN 'STOCK-OUT RISK' THEN 2 ELSE 3 END,
                 loc_sid, sku_sid)              AS rn
  FROM VW_DP_PURCHASE_RECOMMENDATION
  WHERE COALESCE(recommended_order_qty, 0) > 0
  QUALIFY rn <= 20
)
SELECT
    9000 + rn                                AS recommendation_id,
    9000 + rn                                AS approval_id,
    sku_sid, sku_id, sku_ldesc,
    loc_sid, loc_id, loc_name,
    CASE WHEN MOD(rn,2)=0 THEN 3 ELSE 6 END  AS vendor_site_sid,  -- known-valid vendors (3,6)
    rec_qty                                  AS recommended_qty,
    ROUND(50 + MOD(sku_sid,20)*37.5, 2)      AS est_unit_cost,    -- ⚠️ PLACEHOLDER cost
    reorder_pt, order_up_to, lead_days, priority, stock_status
FROM base;

-- 2) Insert recommendations -----------------------------------
INSERT INTO PURCHASE_RECOMMENDATION
  (RECOMMENDATION_ID, SKU_SID, LOC_SID, VENDOR_SITE_SID, RECOMMENDED_QTY,
   ESTIMATED_UNIT_COST, ESTIMATED_TOTAL_VALUE, REQUIRED_DATE, LEAD_TIME_DAYS,
   PRIORITY, STOCKOUT_RISK_FLAG, BUSINESS_REASON, FORECAST_REFERENCE,
   INVENTORY_REFERENCE, REORDER_POINT, ORDER_UP_TO_QTY, STATUS, CREATED_BY,
   CREATED_TIMESTAMP, EXPIRES_AT, IS_POC_FLAG)
SELECT
   recommendation_id, sku_sid, loc_sid, vendor_site_sid, recommended_qty,
   est_unit_cost, ROUND(recommended_qty * est_unit_cost, 2),
   DATEADD(day, 7, CURRENT_DATE), lead_days, priority,
   (stock_status IN ('OUT-OF-STOCK','STOCK-OUT RISK')),
   'DEMO SEED: ' || sku_ldesc || ' at ' || loc_name || ' — ' || stock_status ||
     '. Replenish to safe stock ahead of forecast demand.',
   'FORECAST-2026-DEMO',
   'VW_DP_INVENTORY_POSITION:SKU' || sku_sid || '_LOC' || loc_sid,
   reorder_pt, order_up_to, 'APPROVED', 'DEMO_SEED',
   CURRENT_TIMESTAMP(), DATEADD(day, 30, CURRENT_TIMESTAMP()), TRUE
FROM TMP_DEMO_SEED_20;

-- 3) Insert matching APPROVED approvals -----------------------
INSERT INTO PURCHASE_APPROVAL
  (APPROVAL_ID, RECOMMENDATION_ID, APPROVAL_STATUS, APPROVER_USER,
   APPROVAL_TIMESTAMP, APPROVED_QTY, MODIFIED_QTY, REJECTION_REASON,
   APPROVAL_NOTES, CREATED_TIMESTAMP, IS_POC_FLAG)
SELECT
   approval_id, recommendation_id, 'APPROVED', 'DEMO_APPROVER',
   CURRENT_TIMESTAMP(), recommended_qty, NULL, NULL,
   'Demo seed approval — approved for replenishment.',
   CURRENT_TIMESTAMP(), TRUE
FROM TMP_DEMO_SEED_20;

-- 4) Review what was seeded (note the store/supplier/product per rec) --
SELECT recommendation_id, approval_id, loc_id, loc_name, vendor_site_sid,
       sku_id, sku_ldesc, recommended_qty, est_unit_cost,
       ROUND(recommended_qty*est_unit_cost, 2) AS est_total_value, priority
FROM TMP_DEMO_SEED_20
ORDER BY recommendation_id;
```

**Notes / assumptions you may want to tweak:**
- `vendor_site_sid` alternates between **3** and **6** (the only vendor SIDs proven to exist). Change if you have others.
- `STATUS` is set to `'APPROVED'`; if `SP_CREATE_PURCHASE_ORDER` requires a different ready-state value, adjust here.
- `REQUIRED_DATE` = today + 7 days. Edit the `DATEADD` if you want a fixed date.
- Run query **#4** after seeding to see the exact `recommendation_id`, `approval_id`, store, supplier, and product for each of the 20 rows — you'll reference those in the prompt below.

---

### 2) Prompt you can give me to create a PO for a specific one

After you've run the seed in a write-enabled session, use one of these (fill in the ID or store from query #4):

> **"Create a purchase order for recommendation 9007 — use its seeded store, supplier, product, approved quantity and unit cost, Net 30 terms. Confirm the lines with me before creating, then create, generate the document, and pause before sending."**

Or by store/supplier:

> **"Create a purchase order for store STR10xx / supplier 3 from the demo seed (recommendation 90xx) for required date 2026-09-08, Net 30. Show me the derived lines first."**

When you send that, I'll: derive the line from the seeded recommendation/approval → confirm with you → call `SP_CREATE_PURCHASE_ORDER` → `SP_GENERATE_PO_DOCUMENT` → and stop for your explicit approval before `SP_SEND_PO_NOTIFICATION` (the mandatory send gate). All of that will need the same write-enabled session.

Want me to also add a matching **cleanup/rollback script** (to remove the seeded POs, approvals, and recommendations after the demo)?

---

### Recommended order data (critical + high priority)

| LOC_ID   | LOC_NAME                                  | SKU_ID   | SKU_DESC                                                | PRIORITY | RECOMMENDED_ORDER_QTY | CURRENT_AVAILABLE_QTY | AVG_DAILY_DEMAND |
| -------- | ----------------------------------------- | -------- | ------------------------------------------------------- | -------- | --------------------- | --------------------- | ---------------- |
| STR1006  | DMart Adajan - Surat, Gujarat             | SKU10076 | DMart Men Cotton Brief 3-Pack                           | CRITICAL | 77                    | 0                     | 4.79             |
| STR1007  | DMart Alkapuri - Vadodara, Gujarat        | SKU10071 | Philips LED Bulb 9W Cool Day Light Pack of 4            | CRITICAL | 55                    | 0                     | 3.25             |
| STR1009  | DMart Kukatpally - Hyderabad, Telangana   | SKU10041 | DMart Mens Round Neck Cotton T-Shirt Plain              | CRITICAL | 74                    | 0                     | 4.44             |
| STR1022  | DMart Borivali - Mumbai, Maharashtra      | SKU10041 | DMart Mens Round Neck Cotton T-Shirt Plain              | CRITICAL | 86                    | 0                     | 5.13             |
| STR1032  | DMart Bannerghatta - Bangalore, Karnataka | SKU10041 | DMart Mens Round Neck Cotton T-Shirt Plain              | CRITICAL | 54                    | 0                     | 2.96             |
| STR1044  | DMart Kolar Road - Bhopal, Madhya Pradesh | SKU10099 | DMart Women Palazzo Pants Cotton                        | CRITICAL | 78                    | 0                     | 4.75             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10020 | Amul Pasteurised Butter 500g Carton                     | HIGH     | 66                    | 46                    | 6.72             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10021 | McCain French Fries Crispy Happy 425g Frozen            | HIGH     | 64                    | 40                    | 6.35             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10024 | Colgate Strong Teeth Toothpaste with Calcium 200g       | HIGH     | 129                   | 26                    | 9.35             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10031 | Harpic Power Plus Toilet Cleaner Original 1L            | HIGH     | 80                    | 45                    | 7.72             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10032 | Lizol Disinfectant Surface & Floor Cleaner Citrus 975ml | HIGH     | 75                    | 30                    | 6.45             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10033 | Odonil Bathroom Air Freshener Jasmine 75g Block         | HIGH     | 85                    | 53                    | 8.48             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10034 | Milton Thermosteel Flip Lid Flask 1 Litre Stainless     | HIGH     | 77                    | 21                    | 5.74             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10035 | Prestige Omega Deluxe Non-Stick Kadai 240mm             | HIGH     | 79                    | 31                    | 6.59             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10036 | Cello Fit & Fresh Airtight Container Set 5 Pieces       | HIGH     | 110                   | 17                    | 7.68             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10037 | Relaxo Flite Hawaii Rubber Slippers Men                 | HIGH     | 90                    | 5                     | 5.64             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10038 | Bata Comfit Outdoor Sandals Men Brown                   | HIGH     | 90                    | 15                    | 6.29             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10039 | Funskool Giggles Building Blocks 60 Pieces              | HIGH     | 88                    | 12                    | 6.07             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10040 | Classmate Single Line Notebook 172p 6-Pack              | HIGH     | 98                    | 8                     | 6.22             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10041 | DMart Mens Round Neck Cotton T-Shirt Plain              | HIGH     | 114                   | 10                    | 7.46             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10042 | DMart Mens Formal Trouser Polyester Blend               | HIGH     | 105                   | 13                    | 7.26             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10043 | DMart Womens Printed Cotton Kurti A-Line                | HIGH     | 122                   | 9                     | 8.01             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10044 | DMart Womens Pure Cotton Saree with Blouse              | HIGH     | 102                   | 9                     | 6.78             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10045 | DMart Kids Graphic Print Cotton T-Shirt                 | HIGH     | 85                    | 23                    | 6.51             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10050 | Bisleri Mineral Water 2L Pack of 6                      | HIGH     | 68                    | 40                    | 6.67             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10052 | Tropicana Mixed Fruit Delight 1L Tetra                  | HIGH     | 56                    | 37                    | 5.34             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10054 | Parle-G Gold Biscuit 1kg Family Pack                    | HIGH     | 73                    | 44                    | 7.17             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10059 | Cerelac Baby Cereal Wheat Honey 300g                    | HIGH     | 86                    | 32                    | 7.35             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10063 | Parachute Advansed Body Lotion 400ml                    | HIGH     | 84                    | 29                    | 7.06             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10069 | Havells Reo 500W Mixer Grinder 3 Jar                    | HIGH     | 108                   | 17                    | 7.72             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10070 | Havells Dry Iron Insta 750W                             | HIGH     | 74                    | 33                    | 6.42             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10071 | Philips LED Bulb 9W Cool Day Light Pack of 4            | HIGH     | 94                    | 7                     | 6.08             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10072 | Safari Poly Cabin Trolley Bag 55cm                      | HIGH     | 91                    | 10                    | 6.07             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10073 | Wildcraft Backpack 35L Laptop Bag                       | HIGH     | 98                    | 7                     | 6.30             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10074 | Hawkins Futura Pressure Cooker 3L                       | HIGH     | 81                    | 34                    | 6.82             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10075 | Borosil Glass Lunch Box Set 3pc                         | HIGH     | 92                    | 24                    | 7.23             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10076 | DMart Men Cotton Brief 3-Pack                           | HIGH     | 89                    | 22                    | 6.86             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10077 | DMart Women Cotton Camisole 2-Pack                      | HIGH     | 97                    | 5                     | 6.36             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10078 | DMart Men Polo T-Shirt Cotton                           | HIGH     | 98                    | 13                    | 6.55             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10079 | DMart Women Embroidered Kurta Set                       | HIGH     | 102                   | 15                    | 7.22             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10080 | DMart Men Denim Jeans Slim Fit                          | HIGH     | 85                    | 20                    | 6.28             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10087 | DMart Kids Denim Shorts                                 | HIGH     | 109                   | 11                    | 7.33             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10088 | DMart Girls Printed Frock                               | HIGH     | 110                   | 10                    | 7.36             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10089 | Sparx Men Running Shoes                                 | HIGH     | 104                   | 10                    | 6.98             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10090 | Paragon Women PU Chappal                                | HIGH     | 125                   | 11                    | 8.39             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10091 | Hot Wheels 5-Car Gift Pack                              | HIGH     | 79                    | 24                    | 6.46             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10092 | Doms Colour Pencils 24-Shade Pack                       | HIGH     | 99                    | 9                     | 6.60             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10098 | Havells Instanio Prime 3L Instant Water Heater          | HIGH     | 84                    | 13                    | 5.85             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10099 | DMart Women Palazzo Pants Cotton                        | HIGH     | 83                    | 11                    | 5.59             |
| ECOM9001 | DMart Ready - Pan India Ecommerce         | SKU10100 | DMart Men Vest Cotton 3-Pack                            | HIGH     | 84                    | 22                    | 6.44             |

### Critical stock-out stores (ranked by recommended qty)

| Store | SKU | Product | Recommended Qty |
|---|---|---|---|
| STR1022 – Borivali, Mumbai | SKU10041 | Mens Round Neck Cotton T-Shirt | 87 |
| STR1044 – Kolar Road, Bhopal | SKU10099 | Women Palazzo Pants Cotton | 79 |
| STR1006 – Adajan, Surat | SKU10076 | Men Cotton Brief 3-Pack | 77 |
| STR1009 – Kukatpally, Hyderabad | SKU10041 | Mens Round Neck Cotton T-Shirt | 74 |
| STR1007 – Alkapuri, Vadodara | SKU10071 | Philips LED Bulb 9W Pack of 4 | 55 |
| STR1032 – Bannerghatta, Bangalore | SKU10041 | Mens Round Neck Cotton T-Shirt | 54 |