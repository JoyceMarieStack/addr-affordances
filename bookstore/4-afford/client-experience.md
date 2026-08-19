# Client Experience — Bookstore Example

A single shopper's flow, walked through twice — once as an agent client, once as a
human/chat client — against the same operations in
[Order-Creation-API-REST.afford.oas3.yaml](Order-Creation-API-REST.afford.oas3.yaml).
Same state machine, same endpoints; the only thing that changes is the `Accept`
header, and therefore the shape that comes back. See [affordances.md](affordances.md)
for the table these responses are derived from.

## 1. Browse → view book details

```
GET /books/12345
Accept: application/vnd.bookstore.human+json
```
```json
{
  "bookId": "12345", "isbn": "978-0321834577",
  "title": "Implementing Domain-Driven Design",
  "suggestedAction": { "label": "Add to cart", "href": "/carts", "method": "POST" }
}
```
A chat UI renders one button: **[ Add to cart ]**. It doesn't need to know what else
is possible — the server already decided.

```
GET /books/12345
Accept: application/vnd.bookstore.agent+json
```
```json
{
  "bookId": "12345", "isbn": "978-0321834577",
  "title": "Implementing Domain-Driven Design",
  "links": {
    "addToCart": { "href": "/carts", "method": "POST" },
    "viewAuthor": { "href": "/authors/765", "method": "GET" }
  }
}
```
An agent gets both options and picks based on its own goal — maybe it wants author
details before deciding, something the human flow never exposes as a choice.

## 2. Add to cart (empty → non-empty)

Same call, `POST /carts/{cartId}/items`, both consumers — but the *next*
representation differs by `cartStatus`/`items`:

- Human: `suggestedAction: { "label": "Checkout", "href": "/orders", "method": "POST" }`
- Agent: `links: { addItem, removeItem, modifyItem, clearCart, checkout }`

The chat client never shows "remove item" as a top-level nudge even though it's a
legitimate operation — that's the "narrower than the agent set" rule from the table.

## 3. Checkout → pay → track

The state machine plays out identically for both consumers; the shape stays split
at every step.

```
POST /orders/98765/payments  →  paymentStatus: Failed
```
- Human: `suggestedAction: "Try a different card" → POST /orders/98765/payments`
- Agent: `links: { retryPayment, cancelOrder }` — it can choose to retry
  automatically or escalate, without the server pre-deciding for it

```
GET /orders/98765  →  orderStatus: Shipped
```
- Human: `suggestedAction: "Track shipment"` — no `href`, since it's out of scope
  (external carrier)
- Agent: `links: { viewOrder }` only — `cancelOrder` has already dropped out of the
  set server-side; an agent that kept a stale link from an earlier `Paid`-state
  response and tried it now gets a 4xx, not a silent no-op

## What the split actually buys

The human client is dumb by design — it renders whatever `suggestedAction` says and
never branches. The agent client has to branch on `orderStatus`/`paymentStatus`
itself by inspecting which rels are present in `links`, since nothing tells it
"you're in state X" except the shape of what came back. That's the trade being
made: humans get pre-decided simplicity, agents get raw material to reason over.

## Open gap

Nothing here stops a client from requesting the "wrong" `Accept` for its kind — a
chat surface could ask for `agent+json` and get a rel set it has no UI for. The spec
doesn't enforce the pairing; that's a client-side discipline problem, not something
OAS can gate.
