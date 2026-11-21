# InkOS customer API

Connect a print shop’s CRM, estimating system or production dashboard to the shop’s InkOS records. This proposed read interface covers quote revisions, accepted orders and work orders. It also covers sign production, CNC routing and laser cutting. A shop can use it with or without a storefront.

**Draft for public review.** The API base path proposed here is `/api/customer/v1`. Public service addresses, external credentials, production limits and sandbox enrollment have yet to be announced. Use the contract to build against a local mock. The InkOS dashboard’s session API and its time-limited buyer review links serve separate purposes. [Download the OpenAPI contract](openapi.yaml)

## Connect one shop

An administrator would authorize a server-side integration for one shop and grant the read permissions it needs. Send its bearer token in the HTTPS Authorization header. Keep a separate token, stored records and sync checkpoint for each shop. A company-wide CRM connecting three print shops would keep three independent connections.

| Permission | Records available |
| --- | --- |
| `quotes:read` | Issued revisions, quantities, customer-facing selling prices, expiry and acceptance state |
| `orders:read` | Accepted order, frozen quote reference and agreed selling price, customer reference and work-order IDs |
| `work_orders:read` | Process, material, thickness, instructions, proof status and the released artwork revision ID |

A known record ID cannot grant access to another shop. The contract exposes retail selling prices and production specifications. Shop cost formulas, supplier bids and private file links use separate permissions or services.

## Sync a production dashboard

1. Start with `GET /sync?resource=work_orders&limit=100`. The first pages describe the records visible to the token when the sync starts.
2. For each `upsert`, read `GET /work-orders/{id}` to fetch the current ticket. Upsert by stable work-order ID. A `delete`, or a 404 on a record that was previously visible, removes it from your local copy.
3. Commit those updates and the returned `nextCursor` together. While `hasMore` is true, request the next page immediately. When false, keep the cursor and poll again later. The cursor bridges the initial snapshot and changes committed during the initial scan.
4. After a connection failure, reuse the last committed cursor. Redelivered changes are safe to process again. A 410 means the checkpoint expired. Build a replacement snapshot in parallel and swap it in once complete so stale records are removed.

Set the example base variable to your own mock server, including the proposed base path. The values below are fictional.

```sh
curl "$INKOS_CUSTOMER_BASE/work-orders/wo_demo_18" \
  -H "Authorization: Bearer $INKOS_SHOP_TOKEN"
```

```json
{
  "id": "wo_demo_18",
  "number": "WO-0018",
  "orderId": "o_demo_18",
  "status": "released",
  "process": "CNC routing",
  "material": "Cast acrylic",
  "thickness": "6 mm",
  "productionInstructions": "Route the paths on the CUT layer and polish the exposed edges.",
  "approvedRevisionId": "art_demo_2",
  "releasedRevisionId": "art_demo_2",
  "updatedAt": "2026-11-21T16:30:00Z"
}
```

If someone later uploads artwork revision 3, the released work order still identifies revision 2. A quote revision likewise has a stable revision ID and shares a series ID with its predecessors. An accepted order keeps its original revision and price. Monetary amounts are integers in the currency’s minor unit, so `6600` in CAD means CAD 66.00. Evaluate an open quote’s expiry using its `expiresAt` timestamp even between sync polls.

## Status and errors

| Resource | States in this draft |
| --- | --- |
| Quote revision | `issued`, `accepted`, `superseded`, `expired` |
| Order | `accepted`, `released` |
| Work order | `awaiting_proof`, `approved`, `released` |

A 401 indicates missing, invalid or expired credentials. A 403 indicates missing permission. A 404 can mean either the record does not exist or the shop cannot see it. A 400 indicates invalid inputs or a cursor from another stream. On 429 or 503, respect the Retry-After header when supplied. Include the response request ID when reporting integration issues. Clients should tolerate additional fields and preserve unfamiliar status values during synchronization.

## Trade integrations

For publishing supplier capabilities, discovering wholesale services, obtaining firm offers, dispatching jobs and exchanging delivery updates, use InkOS Trade. The supplier invoices the buying shop directly. [InkOS trade API](https://github.com/InkOS-io/trade-api)
