# Align-Define-Design-Afford-Refine (ADDAR) Process Examples

Examples of the ADDR API process for use in our [LaunchAny workshops](https://launchany.com/api-training) or for self-paced learning.

ADDR becomes **ADDAR**: Align → Define → Design → Afford → Refine. Afford is a new
phase, inserted between Design and Refine, that answers a question the operation
table from Design doesn't: *given the state a resource is in right now, what should
the caller be able to do next, and who is the caller?*

## The thread through the phases

| Phase | Produces | Bookstore example |
|---|---|---|
| **Align** | The job story exists | [1-align](bookstore/1-align/) |
| **Define** | The resource exists | [2-define](bookstore/2-define/) |
| **Design** | The operation exists to satisfy the job story | [3a-design-rest](bookstore/3a-design-rest/) (and the GraphQL/gRPC/async variants) |
| **Afford** | Per operation, decide what becomes reachable next, and for which kind of consumer | [4-afford](bookstore/4-afford/) |
| **Refine** | The chosen shapes get built into the live representations | — |

Each phase depends on the artifact the previous one produced — Design can't assign
an operation to a job story that doesn't exist in Align; Afford can't decide what's
reachable next from an operation that doesn't exist in Design.

## What's new in Afford

Afford takes the same operation table Design produced and adds one more column:
**consumer** — `human` or `agent` — which determines the shape a given affordance
ships as:

- **Human / chat** — one curated **suggestion** (e.g. a `suggestedAction` field):
  the single next step worth nudging toward, already disambiguated.
- **Agent** — the full **`links`** set, JSON:API-style, one entry per reachable
  operation, so a program can reason over every valid transition and pick one
  deterministically.

"Reachable next" is state-gated, not just resource-gated: what an operation affords
depends on what state the resource is in when it responds, not just which resource
it is. See [bookstore/4-afford/affordances.md](bookstore/4-afford/affordances.md)
for the worked example, including the affordance table and representation samples
for both consumer shapes.
