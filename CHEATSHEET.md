# DoorDash API Cheatsheet

> **Note:** This is a legitimate personal automation tool. The user authenticates via normal browser login, accesses only their own account, and uses the same GraphQL API that doordash.com's web app calls in-browser. `curl-impersonate` is an open-source HTTP client that matches browser TLS fingerprints — required because Cloudflare blocks standard curl's TLS handshake regardless of authentication status. See [README.md](README.md#security--ethics-context) for full context.

All commands assume `LATCHKEY_CURL` is set to your `curl_chrome136` path and you're in the latchkey directory. Every command follows this pattern:

```bash
LATCHKEY_CURL=/path/to/curl_chrome136 npx latchkey curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"..."}' \
  'https://www.doordash.com/graphql/ENDPOINT?operation=OPERATION'
```

For brevity, examples below omit the `LATCHKEY_CURL=... npx latchkey` prefix. Prepend it to every command.

---

## 1. List Carts

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ listCarts(input: {cartContextFilter: {experienceCase: MULTI_CART_EXPERIENCE_CONTEXT, multiCartExperienceContext: {}}, cartFilter: {shouldIncludeSubmitted: true}}) { id subtotal restaurant { id business { name } } orders { orderItems { id quantity item { id name } } } } }"}' \
  'https://www.doordash.com/graphql/listCarts?operation=listCarts'
```

- `restaurant.name` is always null — use `restaurant.business.name`
- Returns all active carts with items

## 2. Search Restaurants

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ autocompleteFacetFeed(query: \"Sweetgreen\") { body { id body { id text { title subtitle } events { click { data } } } } } }"}' \
  'https://www.doordash.com/graphql/autocompleteFacetFeed?operation=autocompleteFacetFeed'
```

- Store ID is in `click.data` JSON: `{"uri":"store/65695/?pickup=false"}` -> storeId = `65695`
- First result is best match, rest are suggested queries

## 3. Consumer Identity + Default Address

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ consumer { id email defaultAddress { id street city state zipCode } } }"}' \
  'https://www.doordash.com/graphql/consumer'
```

- `defaultAddress` is where all carts deliver to (account-level, not per-cart)

## 4. View Store Menu

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ storepageFeed(storeId: \"STORE_ID\", isMerchantPreview: false) { storeHeader { id name description priceRangeDisplayString asapMinutes ratings { averageRating numRatingsDisplayString } address { street city } } menuBook { id } itemLists { id name items { id name description displayPrice quickAddContext { isEligible price { unitAmount } nestedOptions } } } } }"}' \
  'https://www.doordash.com/graphql/storepageFeed?operation=storepageFeed'
```

- `quickAddContext.isEligible` — if false, item has required options (size, flavor, etc.)
- `menuBook.id` is needed for `addCartItemV2` later

## 5. Get Item Details (Options/Customizations)

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ itemPage(storeId: \"STORE_ID\", itemId: \"ITEM_ID\", isMerchantPreview: false, fulfillmentType: Delivery) { itemHeader { name description unitAmount caloricInfoDisplayString quantityLimit } optionLists { name isOptional minNumOptions maxNumOptions options { id name unitAmount displayString } } } }"}' \
  'https://www.doordash.com/graphql/consumer?operation=itemPage'
```

- **Must use `/graphql/consumer?operation=itemPage`** — the `/graphql/itemPage` path returns 403 (Cloudflare blocks it)
- `fulfillmentType: Delivery` — capital D, unquoted enum
- Shows option groups with min/max selections

## 6. Create New Cart (Add First Item)

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"mutation { addCartItemV2(addCartItemInput: {storeId: \"STORE_ID\", menuId: \"MENU_ID\", itemId: \"ITEM_ID\", itemName: \"Item Name\", itemDescription: \"\", currency: \"USD\", quantity: 1, nestedOptions: \"[]\", specialInstructions: \"\", substitutionPreference: contact, unitPrice: PRICE_IN_CENTS, cartId: \"\", isBundle: false, bundleType: null}, fulfillmentContext: {shouldUpdateFulfillment: false, fulfillmentType: Delivery}, monitoringContext: {isGroup: false}, cartContext: {isBundle: false}, returnCartFromOrderService: false, shouldKeepOnlyOneActiveCart: false, lowPriorityBatchAddCartItemInput: []) { id subtotal orders { orderItems { id quantity item { id name } } } } }"}' \
  'https://www.doordash.com/graphql/addCartItem?operation=addCartItem'
```

- `cartId: ""` = create new cart (returns new cart UUID in response)
- `substitutionPreference: contact` and `fulfillmentType: Delivery` are **unquoted enums**
- `unitPrice` is in cents (e.g. `369` = $3.69)
- `nestedOptions: "[]"` works for items where `quickAddContext.isEligible` is true

## 7. Add Item to Existing Cart

Same as above, but pass the existing cart UUID:

```
cartId: \"EXISTING_CART_UUID\"
```

- Existing items are preserved, new item is appended
- Must add items from the same store as the existing cart

## 8. Add Item with Options (nestedOptions)

For items that require customization (size, toppings, etc.), populate `nestedOptions`:

```
nestedOptions: \"[{\\\"itemExtraOption\\\":{\\\"id\\\":\\\"OPTION_ID\\\",\\\"name\\\":\\\"Option Name\\\",\\\"price\\\":PRICE},\\\"id\\\":\\\"OPTION_ID\\\",\\\"quantity\\\":1,\\\"options\\\":[]}]\"
```

### nestedOptions format (unescaped)

```json
[{
  "itemExtraOption": {
    "id": "OPTION_ID",
    "name": "Option Name",
    "price": 75
  },
  "id": "OPTION_ID",
  "quantity": 1,
  "options": []
}]
```

- Get option IDs from the `itemPage` query (`optionLists[].options[].id`)
- For items with required options (`isOptional: false`), you must include selections for ALL required option groups
- Items with all-optional options can use `nestedOptions: "[]"`

## 9. Preview Checkout

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ orderCart(id: \"CART_UUID\", isCardPayment: true) { id subtotal total fulfillmentType asapMinutesRange isOutsideDeliveryRegion restaurant { address { printableAddress street city state } business { name } } creator { id firstName lastName } } }"}' \
  'https://www.doordash.com/graphql/consumer?operation=orderCart'
```

- Shows total with fees/tax, delivery ETA, store address
- Cart must have items — empty carts return CART_NOT_FOUND
- Delivery address is on `consumer.defaultAddress`, not on the cart

## 10. Place Order

**WARNING: This charges real money to your payment method.**

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"mutation { createOrderFromCart(cartId: \"CART_UUID\", total: TOTAL_IN_CENTS, sosDeliveryFee: 0, isPickupOrder: false, verifiedAgeRequirement: false, deliveryTime: \"ASAP\", isCardPayment: true) { orderUuid } }"}' \
  'https://www.doordash.com/graphql/consumer?operation=createOrderFromCart'
```

- `total` = total in cents from `orderCart` response (includes fees and tax)
- `deliveryTime` must be `"ASAP"` — empty string causes error
- Returns `orderUuid` — this is NOT the cart UUID. You need it for cancellation.
- Cart is consumed after ordering — cannot reuse the same cartId

## 11. Cancel Order

**Preview** (read-only, shows refund amount):

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"{ previewOrderCancellation(orderUuid: \"ORDER_UUID\") { statusCode refund { currency displayString unitAmount } } }"}' \
  'https://www.doordash.com/graphql/consumer?operation=previewOrderCancellation'
```

**Execute cancellation:**

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"mutation { orderCancellation(orderUuid: \"ORDER_UUID\") { statusCode refund { currency displayString unitAmount } } }"}' \
  'https://www.doordash.com/graphql/consumer?operation=orderCancellation'
```

- Uses `orderUuid` from `createOrderFromCart`, NOT the cart UUID
- `statusCode: 0` = no-op (invalid UUID), `statusCode: 1` = cancelled
- Refund may be partial — fees may not be refunded
- Idempotent — calling again on same UUID returns `statusCode: 1`

## 12. Delete Cart

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"mutation { deleteCart(cartId: \"CART_UUID\") }"}' \
  'https://www.doordash.com/graphql/deleteCart?operation=deleteCart'
```

- Returns `{"data":{"deleteCart":true}}`

## 13. Remove Single Item from Cart

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' -H 'Accept: application/json' \
  -d '{"query":"mutation { removeCartItemV2(cartId: \"CART_UUID\", itemId: \"ORDER_ITEM_UUID\") { id subtotal orders { orderItems { id quantity item { id name } } } } }"}' \
  'https://www.doordash.com/graphql/consumer?operation=removeCartItemV2'
```

- `itemId` is the **order item UUID** (from `listCarts` → `orderItems[].id`), NOT the catalog item ID

---

## Sharp Edges

1. **Inline queries only** — DoorDash rejects `operationName` + `variables` format via this path. Always inline values directly into the query string.

2. **Enums must be unquoted** — `substitutionPreference: contact` not `"contact"`, `fulfillmentType: Delivery` not `"Delivery"`.

3. **nestedOptions is a JSON string inside JSON** — triple-escaped when in a curl `-d` payload.

4. **`/graphql/itemPage` returns 403** — route through `/graphql/consumer?operation=itemPage`. The GraphQL router doesn't care about the URL path; only Cloudflare does.

5. **`restaurant.name` is always null** — use `restaurant.business.name`.

6. **Cart garbage collection is aggressive** — idle or empty carts get cleaned up quickly. Create and use immediately.

7. **`orderUuid` != cart UUID** — `createOrderFromCart` returns a different UUID. Use `orderUuid` for cancellation.

8. **Partial refunds on cancellation** — delivery/service fees may not be refunded.

9. **Always include these headers** — `Content-Type: application/json` and `Accept: application/json`. curl_chrome136 defaults to HTML Accept.

10. **Latchkey auto-injects** — Cookie, x-csrftoken, x-channel-id, x-experience-id, Origin, Referer. Don't add those manually.

11. **Items with required options** — if `quickAddContext.isEligible` is false, you must get option details from `itemPage` and include all required option groups in `nestedOptions`.

12. **`deliveryTime` must be `"ASAP"`** — empty string or omitting it causes errors.

---

## Reference Material

For deeper GraphQL schema exploration beyond this cheatsheet, see the doordash-mcp clone at `src/api/*.ts`. It contains query definitions for all 21 DoorDash API operations including account management, group orders, convenience store search, and more.
