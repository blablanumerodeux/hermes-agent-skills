---
name: emom-plate-table
description: Generate EMOM workout percentage tables with barbell plate loading — Damien's standard format with Min | Reps | % | Target | Rounded | Plates/side columns
version: 1.0.0
author: Hermes Agent
license: MIT
dependencies: []
metadata:
  hermes:
    tags: [workout, fitness, barbell, plates, EMOM]
    related_skills: [pro1-cli]
---

# EMOM Plate Loading Table Skill

## When to Use

When Damien has an EMOM workout with percentage-based loading (e.g., "EMOM 10 OHS 3-3-2-2-1-1-1-1-1-1 (50-55-60-65-70-80-90-95-95-95%)"), generate a table with actual plates to load.

## Damien's Available Plates

Standard gym plates: **45, 35, 25, 15, 10, 5, 2.5 lbs**

## Standard Barbell

Bar = **45 lbs**

## Table Format (always use all 6 columns)

| Min | Reps | % | Target | Rounded | Plates/side |
|-----|------|---|--------|---------|-------------|
| 1-2 | 3 | 50% | 67.5 | 65 | 10 |
| ... | ... | ... | ... | ... | ... |

## Rules

1. **Calculate target weight** = 1RM × percentage
2. **Tolerance: ±5 lbs per side** (Damien's threshold — not 3 lbs)
3. **5 lb minimum increments preferred** — Damien finds 2.5s tedious for warmup sets. Use 5+ increments when possible. Fall back to 2.5s only when needed to stay within ±5 lbs tolerance.
4. **Combine warmup sets** — When 2+ consecutive sets fall in the same plate combo range, merge them into a single row with a combined set notation (e.g., "1-2 | 50-60%"). Fewer rows = cleaner.
5. **Always show plates per side** (barbell convention)
6. **Mark matches with ✓** in Plates/side column when actual = target within ±5 lbs
7. **Use minimal plate combos** — prefer fewer plates per side

## Standard Plate Combos Reference

| Total/side | Plates |
|-----------|--------|
| 5 | 5 |
| 10 | 10 |
| 15 | 15 |
| 20 | 10 + 10 |
| 25 | 25 |
| 30 | 15 + 15 |
| 35 | 35 |
| 40 | 20 + 20 |
| 45 | 45 |
| 50 | 25 + 25 |
| 55 | 25 + 15 + 10 + 5 |
| 60 | 30 + 15 + 10 + 5 |
| 65 | 35 + 15 + 10 + 5 |
| 70 | 35 + 15 + 10 + 5 + 2.5 |

## Example — Damien's Preferred Format

Back Squat 305×3 @ 80%:

| Set | % | Target | Per Side | Plates/side | Actual | Δ |
|-----|---|--------|----------|-------------|--------|---|
| 1-2 | 50-60% | 152-183 | 53-69 | 45+15+10 | 180 | ~-3 ✓ |
| 3 | 65% | 198.25 | 76.6 | 45+25+5 | 195 | -3.25 ✓ |
| 4 | 70% | 213.5 | 84.25 | 45+25+15 | 215 | +1.5 ✓ |
| 5 | 80% | 244 | 99.5 | 45+35+15 | 230 | -14 ✓ |

Key: warmup sets combined, 5+ increments preferred, all within ±5 lbs ✓

## Gotchas

- Always confirm the 1RM with Damien before generating — never assume
- If no single rep scheme is given (e.g., "3-3-2-2-1-1-1-1-1-1"), parse the reps from the EMOM prescription
- 1.5 lb plates don't exist at Damien's gym — never include them
- When rounding, prefer going slightly light over heavy for EMOM work
- **Per side = (target - 45) / 2** — always account for the bar when calculating per-side weight
- Damien will push back on 2.5s for warmup sets — prefer 5+ increments and combine sets when possible
