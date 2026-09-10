**Overall Stock Status: ⚠️ AT-RISK (not critical).** As of the latest snapshot (2026‑08‑14), DMart's network holds **271,366 units worth ₹7.58 crore (₹75.78M)** across 50 stores and 100 SKUs (5,000 store×SKU positions). **88.6% of positions are Healthy**, but **572 positions (11.5%) are below their minimum threshold** — and critically, **~80% of those have no replenishment PO open**. There is **zero excess/overstock** (no position exceeds the max of 150 units).

A note on scope: this environment can query and visualize the data and assemble a complete, export‑ready report, but it cannot emit a binary PDF file for direct download (file/stage creation is disabled here). Below is the full, print-ready **DMart Inventory Health Report** — you can save/export it to PDF from your browser, and I've flagged how to get a scheduled version delivered automatically.

---

# DMart Inventory Health Report
**Snapshot date:** 2026‑08‑14 · **Coverage:** 50 stores, 100 SKUs, 4 divisions, 20 departments · **Report generated:** 2026‑08‑19

## 1. Executive Summary
| Metric | Value |
|---|---|
| Total inventory value | ₹75,781,456 (₹7.58 Cr) |
| Total regular units on hand | 271,366 |
| Healthy positions | 4,428 (88.6%) |
| Below-min positions (Low + Critical) | 572 (11.5%) |
| — of which have NO purchase order | 459 (≈80%) |
| Critical positions (≤50% of min) | 48 (1.0%) |
| Excess/overstock positions | 0 |
| YTD inventory-adjustment loss | ₹26,215,722 across 33,739 events (net −101,138 units) |
| Clearance positions / value | 506 positions · ₹7.77M |

**Bottom line:** Availability risk, not overstock, is the dominant issue. Loss/shrinkage (₹26.2M YTD) and a large replenishment gap on below-min items are the two areas needing action.

## 2. Inventory Health Distribution
Classification per store×SKU vs. thresholds (min = 20, max = 150 units, uniform across catalog): **Critical** = SOH ≤ 50% of min; **Low‑Stock** = below min; **Healthy** = between min and max; **Excess** = above max.





## 3. Inventory by Product Category
General Merchandise (Home Appliances, Home & Kitchen, Travel) concentrates the most capital; Food staples carry the most below-min exposure.





## 4. Store-Level Health
At-risk (below-min) items are spread fairly evenly, but a cluster of metro stores lead the list.





## 5. 🚨 URGENT — Critical Items Below Minimum
These items are at half of minimum stock (10 units vs. 20 min). **44 of the 48 critical positions have no open PO** and need immediate manual replenishment.



## 6. Replenishment Coverage Gap (key finding)
Of 572 below-min positions, only 113 have a PO in flight — **459 (≈80%) have NO purchase order**, including **44 of 48 critical items**. This is the single most actionable risk in the network.

## 7. Loss, Damage & Defective Stock
- **YTD adjustments:** ₹26.22M loss value, net **−101,138 units** over 33,739 events, driven by shrinkage, theft, damage, and cycle-count/system corrections (each type contributing ~₹2.0–2.3M).
- **Defective stock on hand** is concentrated in fast-moving Food staples/snacks (e.g., Madhur Sugar 1kg, Tata Tea Gold 1kg, Parle‑G 1kg — 110–119 defective units each, ~4–5% of holdings).



## 8. Key Findings
1. **Availability risk dominates** — 11.5% of positions below min; **no overstock** anywhere.
2. **Replenishment gap** — ~80% of below-min items (and 44 of 48 critical) have no PO.
3. **Loss leakage is material** — ₹26.2M YTD across shrinkage/theft/damage/corrections.
4. **Defective concentration** in high-velocity Food staples signals handling/storage issues.
5. **Metro stores** (Chennai-Velachery, Hyderabad-Kukatpally, Mumbai-Malad/Borivali) carry the most at-risk items.

## 9. Recommendations
1. **Raise emergency POs today** for the 48 critical positions (44 uncovered) and the 415 uncovered low-stock positions — prioritize Food staples and Personal Care.
2. **Fix the auto-replenishment trigger** — investigate why 80% of below-min items generated no PO; enforce min-based reorder.
3. **Launch a loss-prevention review** targeting the ₹26.2M YTD leakage, focused on stores with high defective counts and cycle-count variance.
4. **Address defective staples** — audit handling/storage for sugar, tea, biscuits, atta lines.
5. **Right-size max thresholds** — with zero overstock and uniform 20/150 limits, review whether maxes are set too high for capital efficiency.

---

### ✅ Recommended immediate action
Issue replenishment POs for the **459 uncovered below-min positions (44 critical)** within 24 hours, starting with the URGENT list in Section 5.

Would you like me to **automate this inventory health report on a recurring schedule** (e.g., a daily 8:00 AM IST run since positions refresh daily, with the below-min/critical and replenishment-gap sections)? I can set that up if you'd like.