# cs2-skin-tradeup-calculator
Calculate optimal CS2 trade-up contracts from any Steam inventory or from a personal input. Supports weapon skins trade-ups, knife/glove trade-ups, StatTrak items. 
# CS2 Trade-up Calculator API
Check it RapidAPI https://rapidapi.com/christiandamato487/api/cs2-skin-trade-up-calculator

Calculate optimal CS2 trade-up contracts from any Steam inventory. Supports weapon trade-ups, knife/glove trade-ups, StatTrak items, and uses Valve's official float normalization formula.

---

## How It Works

A CS2 trade-up contract lets you exchange 10 skins of the same rarity and collection for 1 skin of the next rarity tier. Since October 2024, 5 Covert skins from the same case can also yield a knife or glove.

This API:
1. Reads your CS2 inventory (or accepts a list of items directly)
2. Groups skins by rarity + collection
3. Finds all valid trade-up combinations
4. Calculates the output float using Valve's normalized formula
5. Fetches current market prices and ranks combos by profitability

---

## Authentication

All requests require your RapidAPI key in the headers:

```
X-RapidAPI-Key: YOUR_KEY
X-RapidAPI-Host: cs2-tradeup-calculator.p.rapidapi.com
```

---

## Endpoints

### 1. GET `/inventory/{steamid}`

Returns up to 300 tradable CS2 items from a public Steam inventory, enriched with rarity, collection and current market price. Also returns the total batch value.

**Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| steamid | string | Yes | 17-digit SteamID64. Find yours at steamid.io |
| cursor | string | No | Pagination cursor from previous response. Omit for first page. |

**Example Request**
```
GET /inventory/76561198093638873
```

**Example Response**
```json
{
  "meta": {
    "steamid": "76561198093638873",
    "items_in_batch": 23,
    "has_more": false,
    "next_cursor": null,
    "cursor_note": null,
    "batch_value": 5.84,
    "batch_value_note": null,
    "currency": "USD"
  },
  "items": [
    {
      "market_hash_name": "AK-47 | Redline (Field-Tested)",
      "rarity": "Classified",
      "rarity_id": "rarity_legendary_weapon",
      "price": 43.28,
      "currency": "USD"
    }
  ]
}
```

**Pagination**

For large inventories, use the cursor to fetch items in batches of 300:
```
GET /inventory/{steamid}              ← first 300 items
GET /inventory/{steamid}?cursor=XYZ   ← next 300 items
```
Sum `batch_value` across all pages to get the total inventory value.

**Notes**
- Inventory must be set to **Public** in Steam privacy settings
- Stickers, graffiti and cases show `rarity: null` as they are not tradable weapon skins

---

### 2. GET `/tradeups/{steamid}`

Automatically fetches the Steam inventory and calculates all valid trade-up combinations. Results are ranked from most to least profitable. The inventory is saved in a 2-hour session.

**Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| steamid | string | Yes | 17-digit SteamID64 |
| cursor | string | No | Load next batch of items and recalculate. Omit for first call. |

**Example Request**
```
GET /tradeups/76561198093638873
```

**Example Response**
```json
{
  "meta": {
    "steamid": "76561198093638873",
    "items_in_batch": 23,
    "total_items_loaded": 23,
    "tradeups_found": 2,
    "has_more": false,
    "next_cursor": null,
    "cursor_note": null,
    "currency": "USD",
    "float_source": "estimated",
    "float_note": "Floats are estimated based on each skin's wear and float range. Call PUT /tradeups/{steamid}/refine to recalculate with real float values."
  },
  "tradeups": [
    {
      "type": "weapon",
      "collection": "The Dust 2 Collection",
      "inputs": [
        {
          "market_hash_name": "AK-47 | Safari Mesh (Field-Tested)",
          "float_value": 0.265,
          "price": 0.10
        }
      ],
      "required_input_count": 10,
      "total_input_cost": 1.00,
      "possible_outputs": [
        {
          "name": "M4A1-S | VariCamo (Field-Tested)",
          "float": 0.166216,
          "wear": "Field-Tested",
          "price": 0.34,
          "price_available": true,
          "probability": 0.3333
        }
      ],
      "best_output": {
        "name": "M4A1-S | VariCamo (Field-Tested)",
        "float": 0.166216,
        "wear": "Field-Tested",
        "price": 0.34,
        "price_available": true,
        "probability": 0.3333
      },
      "pricing": {
        "all_outputs_priced": true,
        "unpriced_count": 0,
        "unpriced_outputs": [],
        "note": null
      },
      "expected_value": 0.29,
      "profit": -0.71,
      "profit_percentage": -71.00
    }
  ]
}
```

**Large Inventories**

For inventories with more than 300 items, use the cursor to process all items:
```
GET /tradeups/{steamid}              ← loads first 300, calculates
GET /tradeups/{steamid}?cursor=XYZ   ← adds next 300, recalculates
```
Results improve with each batch as more combinations become available.
.

---


### 3. POST `/tradeups`

Calculates trade-up combinations from a list of items you provide directly. Fast response.


**Format**

```json
{
  "items": [
    {
      "market_hash_name": "AK-47 | Redline (Field-Tested)",
      "float_value": 0.25 
    },
    {
      "market_hash_name": "AK-47 | Redline (Field-Tested)",
      "float_value": 0.31 
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| items | array | Yes | List of skins |
| items[].market_hash_name | string | Yes | Exact skin name including wear. Example: `M4A1-S | Cyrex (Field-Tested)` |
| items[].float_value | number | No | Float between 0 and 1. If omitted, estimated from wear + skin range. |



## Response Schema

### TradeupResult

| Field | Type | Description |
|-------|------|-------------|
| type | `"weapon"` or `"knife_glove"` | Weapon trade-up (10 skins) or knife/glove trade-up (5 Covert skins) |
| collection | string | Collection or case name the skins belong to |
| inputs | array | The skins selected for the trade-up (cheapest valid combo) |
| required_input_count | number | 10 for weapons, 5 for knives/gloves |
| total_input_cost | number | Total USD cost of the 10 or 5 input skins |
| possible_outputs | array | All possible output skins with float, price and probability |
| best_output | object | The highest-priced output with available price data |
| pricing | object | Price availability summary (see below) |
| expected_value | number | Weighted average value across all priced outputs |
| profit | number | `expected_value - total_input_cost` |
| profit_percentage | number | Profit as percentage of input cost |

### PricingInfo

| Field | Type | Description |
|-------|------|-------------|
| all_outputs_priced | boolean | True if every possible output has a known market price |
| unpriced_count | number | Number of outputs with no price data |
| unpriced_outputs | array | Names of skins with no price data — check Steam Market manually |
| note | string or null | Warning message if any outputs are unpriced |

### OutputSkin

| Field | Type | Description |
|-------|------|-------------|
| name | string | Full skin name including wear |
| float | number | Calculated output float  |
| wear | string | Wear tier: Factory New, Minimal Wear, Field-Tested, Well-Worn, Battle-Scarred |
| price | number | Current market price in USD. 0 if not available. |
| price_available | boolean | False if no price data exists for this skin |
| probability | number | Chance of getting this specific output (uniform across valid outputs) |

---

## Error Codes

All errors return a consistent structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "explanation"
  }
}
```

| Code | HTTP | Description |
|------|------|-------------|
| INVALID_STEAMID | 400 | steamid must be a 17-digit number. Use steamid.io to convert a profile URL. |
| INVENTORY_EMPTY_OR_PRIVATE | 404 | No tradable CS2 items found. Set inventory to Public in Steam settings. |
| INVALID_INVENTORY_FORMAT | 400 | POST body must contain assets + descriptions arrays |
| INVALID_ITEM_FORMAT | 400 | One or more items missing market_hash_name |
| INVALID_JSON | 400 | Request body is not valid JSON |
| NO_SESSION_FOUND | 404 | Session expired or GET /tradeups/{steamid} was never called |
| INVALID_REFINE_FORMAT | 400 | floats array is missing or empty |
| INVALID_FLOAT_ENTRY | 400 | float_value must be a number between 0 and 1 |
| NO_VALID_ITEMS | 400 | No valid tradable items found in provided input |
| INTERNAL_ERROR | 500 | Unexpected server error. Retry the request. |

---


## Steam Rate Limiting & Reliability
* **Steam API Constraints:** Steam enforces its own rate limits independent of this API. While we have implemented internal mechanisms and safeguards to handle these constraints smoothly, Steam's endpoints can still be unpredictable during high-traffic periods.
* **High-Volume Optimization:** For high-volume use, we highly recommend prioritizing our `POST /tradeups` endpoint to process data through our optimized layer, significantly reducing direct pressure on Steam's services.
* **Our Commitment:** We actively monitor the infrastructure and Steam's API behavior. If any unexpected issues, connection drops, or rate-limiting changes occur on Steam's side, **we are fully committed to investigating the issue and deploying fixes as quickly as possible** to keep the service stable.
* **Continuous Improvement:** We are constantly working on improving our data quality and features. We welcome your feedback and are always ready to implement improvements to make this tool better for everyone.
## Rate Limits

| Plan | Requests/month | Requests/minute
------|---------------|-----------------|
| Free | 50 | 5|
| Pro | 20.000 | 15|
| Ultra | 50.000 | 25 |
| Mega | 150.000| 30 |.
## Data Disclaimer

Pricing Data & Disclaimer
Prices are updated periodically using the Counter-Strike Price Tracker by ByMykel.

**Data Accuracy**: While we strive to provide the most accurate and up-to-date information, the market is constantly changing. Please treat these values as a helpful guide rather than absolute financial data.

**Rare Items**: For certain rare skins price data might occasionally be unavailable . In such cases, we recommend verifying the value directly on the Steam Community Market.

**Non-Affiliation**: This is an independent project and is not affiliated with, endorsed by, or sponsored by Valve Corporation or Steam.

**Continuous Improvement**: We are constantly working on improving our data quality and features. We welcome your feedback and are always ready to implement improvements to make this tool better for everyone

*Price data is provided "as is" for informational purposes only. This API is not affiliated with Valve Corporation or Steam.*

---
MIT License
Copyright (c) 2026 ByMykel
Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
