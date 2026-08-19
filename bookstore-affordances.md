# Afford — Bookstore Example

ADDR becomes ADDAR: **Align → Define → Design → Afford → Refine**. Afford sits between
Design and Refine, and answers one question the operation table in
[3a-design-rest/ShoppingAPI.design.md](../3a-design-rest/ShoppingAPI.design.md) doesn't:
*given the state a resource is in right now, what should the caller be able to do next,
and who is the caller?*

The thread through the phases, stated plainly:

- **Align** → the job story exists (`1-align/1-job-stories.md`)
- **Define** → the resource exists (`2-define/resources.md`)
- **Design** → the operation exists to satisfy the job story (`3a-design-rest/ShoppingAPI.design.md`)
- **Afford** → per operation, decide what becomes reachable next, and for which kind of
  consumer

"Reachable next" is state-gated, not just resource-gated. A `Cart` in `Active` status
affords adding and removing items; the same `Cart` in `Converted` status affords neither —
its only path forward is viewing the order it became. An `Order` in `New` status affords
payment and cancellation; an `Order` in `Shipped` status affords neither.

## Two kinds of consumer, two shapes for the same affordance

- **Human / chat** — a person, or an LLM rendering a conversational turn, doesn't want
  an exhaustive rel-typed link set. It wants the one thing worth doing next, already
  disambiguated. This ships as a single curated **suggestion** field on the
  representation (e.g. `suggestedAction`) — a plain-language nudge plus the one URL/method
  that satisfies it.
- **Agent** — a program (or an LLM driving the API directly, not through a chat surface)
  needs to reason over *all* valid transitions and pick one deterministically, without
  guessing at intent. This ships as the full **`_links`** set, HAL-style, one entry per
  reachable operation, keyed by `rel`.

Same underlying decision — same table — one more column telling you which shape a given
row ships as.

## The affordance table

| Resource / State | Just Completed | Condition | Affordance (rel) | Points To (Operation) | Consumer | Ships As |
|---|---|---|---|---|---|---|
| Book | `ListBooks` / `searchBooks` | any | `viewDetails` | `ViewBookDetails` | agent | `_links` |
| Book | `ListBooks` / `searchBooks` | any | *(none — too many equally-relevant books to pick one)* | — | human | *(no suggestion field)* |
| Book | `ViewBookDetails` | any | `addToCart` | `AddBookToCart` (creates cart if none exists) | agent | `_links` |
| Book | `ViewBookDetails` | any | "Add to cart" | `AddBookToCart` | human | `suggestedAction` |
| Book | `ViewBookDetails` | `authors` present | `viewAuthor` | `getAuthorDetails` | agent | `_links` |
| Cart | `CreateNewCart` / `AddBookToCart` | `status = Active`, `cartItems` empty | `addItem` | `AddBookToCart` | agent | `_links` |
| Cart | `CreateNewCart` / `AddBookToCart` | `status = Active`, `cartItems` empty | "Keep browsing" | `ListBooks` | human | `suggestedAction` |
| Cart | `AddBookToCart` / `ModifyBookInCart` | `status = Active`, `cartItems` non-empty | `addItem`, `removeItem`, `modifyItem`, `clearCart`, `checkout` | `AddBookToCart`, `RemoveBookFromCart`, `ModifyBookInCart`, `clearCart`, `CreateNewOrder` | agent | `_links` |
| Cart | `AddBookToCart` / `ModifyBookInCart` | `status = Active`, `cartItems` non-empty | "Checkout" | `CreateNewOrder` | human | `suggestedAction` |
| Cart | `CreateNewOrder` (cart converted) | `status = Converted` | `viewOrder` | `ViewOrderDetails` | agent | `_links` |
| Cart | `CreateNewOrder` (cart converted) | `status = Converted` | "View your order" | `ViewOrderDetails` | human | `suggestedAction` |
| Cart | *(inactivity timeout)* | `status = Abandoned` | *(none)* | — | agent | `_links` *(empty)* |
| Order | `CreateNewOrder` | `status = New` | `submitPayment`, `cancelOrder` | `SubmitPaymentDetailsForOrder`, `CancelOrder` | agent | `_links` |
| Order | `CreateNewOrder` | `status = New` | "Pay now" | `SubmitPaymentDetailsForOrder` | human | `suggestedAction` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Failed`, order `status = New` | `retryPayment`, `cancelOrder` | `SubmitPaymentDetailsForOrder`, `CancelOrder` | agent | `_links` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Failed`, order `status = New` | "Try a different card" | `SubmitPaymentDetailsForOrder` | human | `suggestedAction` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Success`, order `status = Paid` | `cancelOrder`, `viewOrder` | `CancelOrder`, `ViewOrderDetails` | agent | `_links` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Success`, order `status = Paid` | "Track your order" | `ViewOrderDetails` | human | `suggestedAction` |
| Order | `ViewOrderDetails` | `status ∈ {Preparing, Prepared}` | `cancelOrder`, `viewOrder` | `CancelOrder`, `ViewOrderDetails` | agent | `_links` |
| Order | `ViewOrderDetails` | `status ∈ {Preparing, Prepared}` | *(none — cancellation is a support action, not the customer's primary next step)* | — | human | *(no suggestion field)* |
| Order | `ViewOrderDetails` | `status = Shipped` | `viewOrder` | `ViewOrderDetails` | agent | `_links` (self only) |
| Order | `ViewOrderDetails` | `status = Shipped` | "Track shipment" | *(external carrier tracking, out of scope)* | human | `suggestedAction` |
| Order | `ViewOrderDetails` | `status = Received` | *(none — terminal, success)* | — | agent | `_links` (self only) |
| Order | `ViewOrderDetails` | `status = Received` | "Browse new arrivals" | `ListBooks` | human | `suggestedAction` |
| Order | `ViewOrderDetails` / `CancelOrder` | `status = Cancelled` | *(none — terminal)* | — | agent | `_links` (self only) |
| Order | `ViewOrderDetails` / `CancelOrder` | `status = Cancelled` | "Start a new order" | `ListBooks` | human | `suggestedAction` |

Notes on reading the table:

- A row with no affordance isn't a gap — it's a design decision. Terminal states
  (`Received`, `Cancelled`) and states with too many equally valid next steps (a search
  results page) are supposed to come back empty for at least one consumer.
- `cancelOrder` disappears once `status = Shipped`, matching the constraint already
  documented on `CancelOrder` in
  [Order-Creation-API-REST.oas3.yaml](../3a-design-rest/Order-Creation-API-REST.oas3.yaml):
  refund path if paid, hard-cancel if not — neither applies once the order has left the
  warehouse.
- The human suggestion is deliberately narrower than the agent's link set in every case —
  it's a curated recommendation ("do this one thing"), not a filtered copy of the full
  set. `CancelOrder` in a `Preparing` state is a legitimate agent-reachable transition
  but a bad thing to nudge a customer toward, so it's omitted from `suggestedAction`
  entirely.

## Representation examples

Two representations of the same `Order`, same underlying state (`status = Paid`),
shaped for the two consumers described above:

- [order-agent-hal.json](representation-examples/order-agent-hal.json) — full `_links`
  rel set, for a program deciding what to do next
- [order-human-suggestion.json](representation-examples/order-human-suggestion.json) —
  single `suggestedAction`, for a chat surface or human-facing UI

Compare against the existing REST representation styles in
[3a-design-rest/representation-examples](../3a-design-rest/representation-examples/) —
Afford doesn't replace those serialization choices, it adds a state-driven layer on top
of whichever one you pick.
