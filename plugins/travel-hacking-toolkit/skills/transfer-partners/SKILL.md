---
name: transfer-partners
description: Credit card to loyalty program transfer ratios. Cross-references seats.aero award prices with transfer ratios to find the cheapest path from UR, MR, TYP, Capital One, or Bilt to any mileage program.
category: loyalty
summary: Cheapest transfer path from credit card points to mileage programs.
api_key: None (local data)
---

# Transfer Partner Optimizer

Given award flight prices (from seats.aero or manual input) and your transferable point balances, find the cheapest path to book.

**No API key needed.** Uses local JSON data files.

## Data Files

| File | Purpose |
|------|---------|
| `data/transfer-partners.json` | Credit card → loyalty program transfer ratios |
| `data/points-valuations.json` | CPP valuations per program (floor/ceiling from TPG, UP, OMAAT, VFTW) |
| `data/partner-awards.json` | Which programs can book which airlines |

## When to Use

- User has award availability (from seats.aero cached search, live search, or manual input)
- User wants to know "which of my points should I use?"
- User wants to compare the effective cost across their transferable currencies
- User asks about transfer partners for a specific airline or program

## Workflow

### Step 0: Check Current Transfer Bonuses (Always)

Before recommending any transfer, load the `transfer-bonuses` skill and check `data/transfer-bonuses.json` for active bonuses on the relevant currency-to-program pair. A 30% bonus turns a 1:1 ratio into 1:1.3, which can flip the cheapest-currency calculation in Step 3 entirely. Bonuses also have expiration dates that affect timing decisions.

### Step 1: Get Award Availability

Use seats.aero (API or web) to search for award flights. Each result includes:
- `Source` (the loyalty program: "united", "aeroplan", "flyingblue", etc.)
- Points cost per cabin (e.g., 55,000 business)

### Step 2: Map Seats.aero Sources to Transfer Partners

Seats.aero source names map to `transfer-partners.json` keys:

| Seats.aero Source | Transfer Partner Key | Programs That Transfer In |
|-------------------|---------------------|---------------------------|
| `united` | `united` | Chase UR (1:1), Bilt (1:1) |
| `aeroplan` | `aeroplan` | Chase UR (1:1), Amex MR (1:1), Bilt (1:1), Capital One (1:1) |
| `flyingblue` | `flying_blue` | Chase UR (1:1), Amex MR (1:1), Bilt (1:1), Capital One (1:1), Citi TY (1:1), Wells Fargo (1:1) |
| `american` | `american` | Citi TY (1:1), Bilt (1:1) |
| `alaska` | `alaska_hawaiian` | Bilt (1:1) |
| `virginatlantic` | `virgin_atlantic` | Chase UR (1:1), Amex MR (1:1), Bilt (1:1), Citi TY (1:1) |
| `delta` | `delta` | Amex MR (1:1) |
| `emirates` | `emirates` | Bilt (1:1), Amex MR (5:4), Capital One (4:3), Citi TY (5:4) |
| `etihad` | `etihad` | Amex MR (1:1), Bilt (1:1), Capital One (1:1), Citi TY (1:1) |
| `singapore` | `singapore` | Chase UR (1:1), Amex MR (1:1), Capital One (1:1), Citi TY (1:1) |
| `jetblue` | `jetblue` | Chase UR (1:1), Citi TY (1:1), Wells Fargo (1:1), Amex MR (250:200), Capital One (5:3) |
| `qatar` | `qatar` | Amex MR (1:1), Bilt (1:1), Capital One (1:1), Citi TY (1:1) |
| `turkish` | `turkish` | Bilt (1:1), Capital One (1:1), Citi TY (1:1) |
| `eurobonus` | (no direct transfer) | N/A |
| `aeromexico` | `aeromexico` | Amex MR (1:1.6), Capital One (1:1), Citi TY (1:1) |
| `smiles` | (no direct transfer) | N/A |
| `finnair` | `finnair` | Capital One (1:1) |
| `lufthansa` | (no direct transfer) | N/A |
| `ethiopian` | (no direct transfer) | N/A |
| `saudia` | (no direct transfer) | N/A |

Programs with "no direct transfer" can only be booked by earning miles directly or through alliance partner bookings.

### Step 3: Calculate Effective Cost

For each award option, calculate the cost in each transferable currency:

```
effective_cost = award_miles / transfer_ratio
```

Example: Aeroplan business at 55,000 miles
- Chase UR: 55,000 / 1.0 = **55,000 UR points**
- Amex MR: 55,000 / 1.0 = **55,000 MR points**
- Capital One: 55,000 / 1.0 = **55,000 Cap One miles**

Example: Emirates business at 72,500 miles
- Bilt: 72,500 / 1.0 = **72,500 Bilt points**
- Amex MR: 72,500 / 0.8 = **90,625 MR points** (worse ratio)
- Capital One: 72,500 / 0.75 = **96,667 Cap One miles** (worst ratio)

### Step 4: Calculate Opportunity Cost (CPP)

Use `points-valuations.json` to assess what each currency is "worth":

```
opportunity_cost = effective_cost × point_value_cpp / 100
```

This tells you the "cash equivalent" you're giving up. Lower is better.

### Step 5: Present Results

**Always use markdown tables.**

| Program | Miles Needed | Best Currency | Points Needed | CPP Value | Cash Equivalent |
|---------|-------------|---------------|---------------|-----------|-----------------|
| Aeroplan | 55,000 | Chase UR | 55,000 | 1.7¢ | $935 |
| United | 55,000 | Chase UR | 55,000 | 1.7¢ | $935 |
| Alaska | 37,500 | Bilt | 37,500 | 1.7¢ | $638 |
| Flying Blue | 57,000 | Any 1:1 | 57,000 | 1.7¢ | $969 |

After the table:
- **Recommendation:** "Book via Alaska using Bilt points. 37,500 Bilt = $638 opportunity cost vs next best United/Aeroplan at $935."
- **Check balance:** Verify the user has enough points in the recommended currency.
- **Transfer bonus?** Check if any current transfer bonuses apply (Roame.travel or TPG bonus tracker).

## jq Recipes

### Find all currencies that transfer to a program

```bash
jq -r '
  to_entries | .[] | select(.key != "_meta") |
  .key as $currency | .value.display_name as $name |
  (.value.airlines // {}) + (.value.hotels // {}) |
  to_entries[] | select(.key == "PROGRAM_KEY") |
  "\($name): \(.value.ratio):1 → \(.value.program)"
' data/transfer-partners.json
```

Replace `PROGRAM_KEY` with the key (e.g., `united`, `aeroplan`, `flying_blue`).

### Calculate effective costs for an award

```bash
# Given: 55000 miles via aeroplan
PROGRAM="aeroplan"
MILES=55000
jq -r --arg prog "$PROGRAM" --argjson miles $MILES '
  to_entries | .[] | select(.key != "_meta") |
  .key as $currency | .value.display_name as $name |
  ((.value.airlines // {}) + (.value.hotels // {}))[$prog] // null |
  select(. != null) |
  "\($name): \(($miles / .ratio) | floor) points (ratio \(.ratio):1)"
' data/transfer-partners.json
```

### Find cheapest path for multiple award options

```bash
# Given: united at 55000, aeroplan at 55000, alaska at 37500
echo '[
  {"program": "united", "miles": 55000},
  {"program": "aeroplan", "miles": 55000},
  {"program": "alaska_hawaiian", "miles": 37500}
]' | jq -r --slurpfile tp data/transfer-partners.json '
  .[] | .program as $prog | .miles as $miles |
  ($tp[0] | to_entries | .[] | select(.key != "_meta") |
    .value.display_name as $name |
    ((.value.airlines // {}) + (.value.hotels // {}))[$prog] // null |
    select(. != null) |
    {currency: $name, program: $prog, miles: $miles, points_needed: (($miles / .ratio) | floor), ratio: .ratio}
  )
' | jq -s 'sort_by(.points_needed)'
```

## Cross-Alliance Optimization

Sometimes the cheapest path involves booking through a different program than the obvious one. Check `data/partner-awards.json` for cross-alliance highlights:

- **Virgin Atlantic → ANA:** 52.5K business from West Coast. Cheaper than any Star Alliance program.
- **Etihad → American Airlines:** Fixed chart often beats AA's dynamic pricing.
- **Flying Blue → Delta:** Often cheaper than SkyMiles, plus free stopovers.
- **Alaska → Starlux:** Only points booking option for Starlux.

## Transfer Timing Risks

**Always flag these before recommending a transfer.** Points transfers are irreversible. Timing problems are the most common cause of lost awards.

### Business day delays
Most credit card → loyalty program transfers are processed on business days. A transfer submitted on a weekend or public holiday may not land until the next business day — or the business day after a holiday weekend. For US banks: Saturday/Sunday submissions often process Monday or Tuesday. A Friday night submission before a Monday US federal holiday (Memorial Day, Labor Day, etc.) may not land until Tuesday.

**Rule of thumb:** If seats are at risk of disappearing, transfer before 5pm ET on the last business day before any weekend or holiday — not the night before.

### Per-program transfer speeds (typical, not guaranteed)
| Program | Typical speed | Notes |
|---------|--------------|-------|
| Aeroplan | Minutes to 2 hours | Usually instant via API. "Next business day" warning is CYA language but can be real on weekends. |
| Flying Blue | 2-12 hours | Generally fast. |
| United MileagePlus | Minutes to 2 hours | Usually fast. |
| Hyatt | Minutes to 2 hours | Usually fast. |
| Turkish Miles & Smiles | 1-3 days | Slower than most. |
| Virgin Atlantic | Minutes to 2 hours | Usually fast. |
| Alaska Mileage Plan | Minutes to 2 hours | Usually fast. |

### New loyalty account delays
If the traveler is transferring to a **newly opened loyalty account**, most programs impose a 24-72 hour hold before the account can receive transfers. Opening an Aeroplan account 30 minutes before transferring points is a common mistake.

**Rule:** Open the loyalty account at least 3 days before the transfer if it's a new account. Verify the account number and name match exactly before transferring.

### Never transfer speculatively
If no hold is available (Aeroplan, United, Delta, Alaska do not allow holds), only transfer when:
1. You have confirmed availability on the airline's own booking site
2. You're prepared to book immediately after the transfer lands
3. You've checked `ComputedLastSeen` on seats.aero — stale cache means the seat may already be gone

## Notes

- Transfer ratios rarely change, but verify against issuer websites before large transfers.
- Transfers are irreversible. See Transfer Timing Risks above before committing.
- Some programs run transfer bonuses (10-30% extra). Use the `transfer-bonuses` skill (live data, weekly auto-refresh) instead of guessing.
- The "best" currency depends on what you have the most of AND what you value it at. A 1:1 transfer from a currency you value at 2.0 cpp costs more in opportunity than a 1:1 from one you value at 1.5 cpp.
