# Afford — Bookstore Example

Joyce Thinking in Real Time : I think I'm confusing a few things here with the intent design and the affordances..intent comes first.....I think what this example shows ....here's what happens when you try to bolt affordances onto a data-model-shaped API — you immediately see the operations are too granular which is an issue for agents having to spawn a bunch of requests....when each one is not atomic....

I was thinking "given the state a resource is in right now, what should the caller be able to do next, and who is the caller?"  

And how could I spec that thinking out.

Intent design answers "what verb does this expose" (Do,Know/etc). 

The affordance layer would answer...."given the current state and permissions, which of those verbs can actually be called"

e.g. get_author() would return "available_actions: get_coauthor_network(), "comapre-researchers()" 





**The corrected order of operations:** Define/Design decide operation shape and
count first (intent-first, atomic where a business outcome is single-shot) → Afford
decides, given that shape, what's reachable next and for which consumer. 

If the affordance table sprawls or feels like it's
compensating for something, that's a signal to go back and revisit Design's
granularity — not evidence that affordances is the wrong tool.

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

