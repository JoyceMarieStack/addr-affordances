# Afford — Bookstore Example

NOTE: I think I'm confusing a few things here with the intent design and the affordances..intent comes first.....I think what this shows ....here's what happens when you try to bolt affordances onto a data-model-shaped API — you immediately see the operations are too granular which is an issue for agents having to spawn a bunch of requests....when each one is not atomic....

**Where the note above is right:** intent-shaping *is* a Design-phase decision (which
operations exist, how coarse or fine they are), and Afford is applied *after* that —
it decides what's reachable next given whatever operations Design already produced.
Get that ordering backwards and Afford has nothing to work with yet. Writing the
affordance table for the cart/order flow above is what surfaced that `AddBookToCart`
/ `RemoveBookFromCart` / `ModifyBookInCart` / `CreateNewOrder` / `SubmitPaymentDetailsForOrder`
is still five separate non-atomic calls for an agent that already knows what it wants
to buy — that's a real, correct diagnosis.

**Where it's easy to over-read that observation, though:**

- Afford didn't *cause* the granularity problem, and fixing Afford wouldn't fix it
  either. The operations were already too fine-grained for an agent before any
  `links`/`suggestedAction` field existed — Afford just makes an existing Design
  decision visible by forcing every state transition to be enumerated explicitly.
  It's a diagnostic side-effect, not a defect in the affordance approach itself.
- Afford doesn't reduce call count, and was never going to. Only Define/Design can
  do that, by collapsing `CreateNewCart` → `AddBookToCart`(×N) → `CreateNewOrder` →
  `SubmitPaymentDetailsForOrder` into a single atomic `PlaceOrder`/`place_order`
  intent. Afford's job stays the same either way: given whatever operations exist,
  say which are currently valid and for whom.
- Collapsing to `PlaceOrder` doesn't make Afford unnecessary — it makes the
  *table* shrink. `Order/New → {submitPayment, cancelOrder}` disappears entirely,
  because an order created via `PlaceOrder` is already `Paid`. What's left is
  `Paid → {cancelOrder, viewOrder}` and the terminal states — Afford still earns
  its keep on the states that survive the collapse, it just has fewer rows to cover.

**The corrected order of operations:** Define/Design decide operation shape and
count first (intent-first, atomic where a business outcome is single-shot) → Afford
decides, given that shape, what's reachable next and for which consumer → Refine
checks token/input efficiency. If the affordance table sprawls or feels like it's
compensating for something, that's a signal to go back and revisit Design's
granularity — not evidence that affordances is the wrong tool.

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
  guessing at intent. This ships as the full **`links`** set, JSON:API-style, one entry
  per reachable operation, keyed by `rel`.

Same underlying decision — same table — one more column telling you which shape a given
row ships as.

## The affordance table

| Resource / State | Just Completed | Condition | Affordance (rel) | Points To (Operation) | Consumer | Ships As |
|---|---|---|---|---|---|---|
| Book | `ListBooks` / `searchBooks` | any | `viewDetails` | `ViewBookDetails` | agent | `links` |
| Book | `ListBooks` / `searchBooks` | any | *(none — too many equally-relevant books to pick one)* | — | human | *(no suggestion field)* |
| Book | `ViewBookDetails` | any | `addToCart` | `AddBookToCart` (creates cart if none exists) | agent | `links` |
| Book | `ViewBookDetails` | any | "Add to cart" | `AddBookToCart` | human | `suggestedAction` |
| Book | `ViewBookDetails` | `authors` present | `viewAuthor` | `getAuthorDetails` | agent | `links` |
| Cart | `CreateNewCart` / `AddBookToCart` | `status = Active`, `cartItems` empty | `addItem` | `AddBookToCart` | agent | `links` |
| Cart | `CreateNewCart` / `AddBookToCart` | `status = Active`, `cartItems` empty | "Keep browsing" | `ListBooks` | human | `suggestedAction` |
| Cart | `AddBookToCart` / `ModifyBookInCart` | `status = Active`, `cartItems` non-empty | `addItem`, `removeItem`, `modifyItem`, `clearCart`, `checkout` | `AddBookToCart`, `RemoveBookFromCart`, `ModifyBookInCart`, `clearCart`, `CreateNewOrder` | agent | `links` |
| Cart | `AddBookToCart` / `ModifyBookInCart` | `status = Active`, `cartItems` non-empty | "Checkout" | `CreateNewOrder` | human | `suggestedAction` |
| Cart | `CreateNewOrder` (cart converted) | `status = Converted` | `viewOrder` | `ViewOrderDetails` | agent | `links` |
| Cart | `CreateNewOrder` (cart converted) | `status = Converted` | "View your order" | `ViewOrderDetails` | human | `suggestedAction` |
| Cart | *(inactivity timeout)* | `status = Abandoned` | *(none)* | — | agent | `links` *(empty)* |
| Order | `CreateNewOrder` | `status = New` | `submitPayment`, `cancelOrder` | `SubmitPaymentDetailsForOrder`, `CancelOrder` | agent | `links` |
| Order | `CreateNewOrder` | `status = New` | "Pay now" | `SubmitPaymentDetailsForOrder` | human | `suggestedAction` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Failed`, order `status = New` | `retryPayment`, `cancelOrder` | `SubmitPaymentDetailsForOrder`, `CancelOrder` | agent | `links` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Failed`, order `status = New` | "Try a different card" | `SubmitPaymentDetailsForOrder` | human | `suggestedAction` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Success`, order `status = Paid` | `cancelOrder`, `viewOrder` | `CancelOrder`, `ViewOrderDetails` | agent | `links` |
| Order | `SubmitPaymentDetailsForOrder` | `paymentStatus = Success`, order `status = Paid` | "Track your order" | `ViewOrderDetails` | human | `suggestedAction` |
| Order | `ViewOrderDetails` | `status ∈ {Preparing, Prepared}` | `cancelOrder`, `viewOrder` | `CancelOrder`, `ViewOrderDetails` | agent | `links` |
| Order | `ViewOrderDetails` | `status ∈ {Preparing, Prepared}` | *(none — cancellation is a support action, not the customer's primary next step)* | — | human | *(no suggestion field)* |
| Order | `ViewOrderDetails` | `status = Shipped` | `viewOrder` | `ViewOrderDetails` | agent | `links` (self only) |
| Order | `ViewOrderDetails` | `status = Shipped` | "Track shipment" | *(external carrier tracking, out of scope)* | human | `suggestedAction` |
| Order | `ViewOrderDetails` | `status = Received` | *(none — terminal, success)* | — | agent | `links` (self only) |
| Order | `ViewOrderDetails` | `status = Received` | "Browse new arrivals" | `ListBooks` | human | `suggestedAction` |
| Order | `ViewOrderDetails` / `CancelOrder` | `status = Cancelled` | *(none — terminal)* | — | agent | `links` (self only) |
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

## Applied to Design

The table above isn't just documentation — it's applied back onto the Design-phase
spec in [Order-Creation-API-REST.afford.oas3.yaml](Order-Creation-API-REST.afford.oas3.yaml),
a copy of
[3a-design-rest/Order-Creation-API-REST.oas3.yaml](../3a-design-rest/Order-Creation-API-REST.oas3.yaml)
with Afford's decisions applied. No paths or operationIds changed. What did:

- Every schema that used to describe its hypermedia links only in prose
  (`"...the following hypermedia links are offered..."`) now carries a real `links`
  property (agent) or `suggestedAction` property (human), with a `description` that
  states exactly which rels/labels apply for which resource state — traceable
  1:1 back to the rows above.
- Every response that returns an affordance-bearing resource now content-negotiates
  on `Accept`: `application/vnd.bookstore.agent+json` vs.
  `application/vnd.bookstore.human+json`. This is the piece the table alone doesn't
  settle — OAS can't shrink a schema conditionally by caller identity, so something
  the request carries has to select the shape. Media type was picked here because
  content negotiation already exists for exactly this; a custom header or query
  param would work as well but isn't standard REST practice for representation
  selection.

## Client experience

[client-experience.md](client-experience.md) walks a single shopper through the
flow — browse, add to cart, checkout, pay, track — as both an agent client and a
human/chat client, showing how the same operations return different shapes at each
step.

## Representation examples

Two representations of the same `Order`, same underlying state (`status = Paid`),
shaped for the two consumers described above, both built on the JSON:API-style
envelope from
[3a-design-rest/representation-examples/book-apijson.json](../3a-design-rest/representation-examples/book-apijson.json):

- [order-agent-apijson.json](representation-examples/order-agent-apijson.json) — full
  `links` rel set, for a program deciding what to do next
- [order-human-suggestion-apijson.json](representation-examples/order-human-suggestion-apijson.json) —
  single `suggestedAction`, for a chat surface or human-facing UI

Afford doesn't replace the serialization choice made in Design — it adds a
state-driven layer on top of whichever envelope you pick. This example standardizes
on the JSON:API-style format; the same table would drive identical content into a
HAL or Uber envelope instead.
