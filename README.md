# Code Conventions

How I write code. Every rule has a yes/no example.

- [Style](#style)
- [Functional JavaScript](#functional-javascript)
- [Third party services](#third-party-services)
- [Database writes](#database-writes)
- [Error handling](#error-handling)
- [Async](#async)
- [Boundary validation](#boundary-validation)
- [Config and secrets](#config-and-secrets)
- [Scope of changes](#scope-of-changes)

## Style

**Functional over OOP.** Plain functions and modules. No classes, no inheritance,
no `this`, no instance state. Data goes in as arguments and comes out as return
values. If something needs shared state, it's a module-level singleton that gets
imported, not an object that gets constructed and passed around.

```js
// yes
const buildImageUrl = ({ imageId, width }) => `${CDN_BASE}/${imageId}?w=${width}`;

// no
class ImageUrlBuilder {
  constructor(imageId) {
    this.imageId = imageId;
  }
  withWidth(width) {
    this.width = width;
    return this;
  }
  build() {
    return `${CDN_BASE}/${this.imageId}?w=${this.width}`;
  }
}
```

**Named params, not positional.** Every function takes a single options object with
named keys, even single-arg helpers.

```js
// yes
updateOrderStatus({ orderId, status });
toInvoice({ order });

// no
updateOrderStatus(db, orderId, status);
```

**Declarative over imperative.** Reach for a lookup table before a branch chain.
If/else and switch ladders that map a key to a behavior are data pretending to be
control flow. Write them as data.

```js
// yes
const DEFAULT_HANDLER = ({ event }) => {
  throw new Error(`No handler for event type: ${event.type}`);
};

const handler =
  {
    order_created: handleOrderCreated,
    order_shipped: handleOrderShipped,
    order_refunded: handleOrderRefunded,
  }[event.type] || DEFAULT_HANDLER;

return handler({ event });

// no
if (event.type === "order_created") {
  ...
} else if (event.type === "order_shipped") {
  ...
} else if (event.type === "order_refunded") {
  ...
}
```

Same for status maps, plan tiers, provider names, error code to message, etc.
Adding a case should mean adding a row, not editing logic.

**Defaults come from `||`, not a branch.** A miss on the lookup falls through to a
default on the same expression. Don't follow the lookup with an `if (!x)` block to
fill it in.

```js
// yes
const label = { active: "Active", paused: "Paused" }[status] || "Unknown";

// no
let label = LABELS[status];
if (!label) {
  label = "Unknown";
}
```

A simple fallback value can sit inline. A fallback that is behavior gets pulled
out as a named constant, and for a handler map that should never miss, make it
throw or log rather than fail silently.

Use `??` instead when `0` or `""` is a legitimate result. `||` treats them as
misses and silently hands back the default.

```js
const retries = config.retries ?? 3; // 0 stays 0
const label = LABELS[status] || "Unknown"; // no falsy labels exist, || is fine
```

**Code reads downwards, not inwards.** Guard clauses and early returns, not nested
pyramids. Two levels of indentation inside a function is the ceiling. Past that,
extract a named helper. The happy path runs straight down the left edge so you
can read it top to bottom without tracking which branch you're in.

```js
// yes
if (!user) return null;
if (!user.subscription) return null;
if (user.subscription.status !== "active") return null;
return user.subscription;

// no
if (user) {
  if (user.subscription) {
    if (user.subscription.status === "active") {
      return user.subscription;
    }
  }
}
```

**Always document functions.** In JavaScript, every function gets a JSDoc block
above it: one line on what it does, `@param` for each key in the params object,
`@returns`. If the function has a non-obvious side effect (writes to the database,
calls a vendor API, mutates a cache) say so in the description.

```js
/**
 * Marks an order as shipped and records the tracking number.
 *
 * @param {Object} params
 * @param {string} params.orderId - id of the order
 * @param {string} params.trackingNumber - carrier tracking number
 * @returns {Promise<void>} resolves once the write lands
 */
const markShipped = async ({ orderId, trackingNumber }) => {
  ...
};
```

In TypeScript, the types are the signature. Don't restate them as `@param` and
`@returns` tags that drift out of sync with the real ones. Keep the one line on
what it does, plus side effects and anything the types can't say (units, valid
ranges, what a null means, what throws).

```ts
/**
 * Marks an order as shipped and records the tracking number.
 * Writes to the database. Throws if the order no longer exists.
 */
const markShipped = async ({ orderId, trackingNumber }: MarkShippedParams): Promise<void> => {
  ...
};
```

**Imports at the top.** All requires/imports at the top of the file, always. No
lazy require inside a function, no conditional require, no require halfway down
the module.

```js
// yes
const db = require("../config/db");

const markShipped = async ({ orderId, trackingNumber }) => { ... };

// no
const markShipped = async ({ orderId, trackingNumber }) => {
  const db = require("../config/db");
  ...
};
```

## Functional JavaScript

**Pure by default.** A function takes its inputs as arguments and returns a value.
It doesn't reach out to module state, mutate what it was handed, or fire off IO in
the middle of a transform. Push the side effects (database writes, vendor calls,
logging) to the edges and keep the shaping logic in between pure.

```js
// yes
const toInvoice = ({ order }) => ({
  orderId: order.id,
  total: order.total,
  issuedAt: order.completed_at,
});

const invoice = toInvoice({ order });
await saveInvoice({ invoice });

// no
const toInvoice = async ({ order }) => {
  order.invoiced_at = Date.now(); // mutates the input
  await db.update(...);           // IO inside a transform
  return order;
};
```

**Never mutate inputs.** No `push`, `splice`, `sort`, `reverse`, or property
assignment on anything a function was passed. Return a new value instead. `sort`
and `reverse` mutate in place, so copy first.

```js
// yes
const sorted = [...orders].sort(byCreatedAt);
const withFlag = { ...order, is_paid: true };

// no
orders.sort(byCreatedAt);
order.is_paid = true;
```

**Transform with map, filter, reduce.** Not an empty array plus a loop that pushes
into it. The method name says what the step does, which a `for` loop never does.

```js
// yes
const activeIds = subscriptions
  .filter(isActive)
  .map(({ id }) => id);

// no
const activeIds = [];
for (const subscription of subscriptions) {
  if (subscription.status === "active") {
    activeIds.push(subscription.id);
  }
}
```

`reduce` is the tool for folding a list into one value, and especially for turning
an array of records into a hash table you can look up by key. Wrap the object
literal in parens for the implicit return and it stays one line.

```js
// yes
const usersById = users.reduce((acc, user) => ({ ...acc, [user.id]: user }), {});

// then look up instead of scanning
const user = usersById[userId];

// no
const user = users.find(({ id }) => id === userId); // inside a loop, this is O(n^2)
```

Same shape for grouping and for counting:

```js
const orderIdsByUser = orders.reduce(
  (acc, { id, user_id }) => ({ ...acc, [user_id]: [...(acc[user_id] || []), id] }),
  {}
);

const countByStatus = orders.reduce(
  (acc, { status }) => ({ ...acc, [status]: (acc[status] || 0) + 1 }),
  {}
);
```

Spreading the accumulator copies it each pass, which is fine at everyday sizes. On
a genuinely large array (tens of thousands of rows, a full export) assign onto the
accumulator instead, since it's a local the reducer owns and not an input:

```js
const usersById = users.reduce((acc, user) => {
  acc[user.id] = user;
  return acc;
}, {});
```

**`const` always, effectively.** Reassignment should be rare enough to be
surprising. `var` never. A `let` is a signal that a value is being built up in
steps, and the fix is almost always a `map`/`filter`/`reduce`, a lookup with a
`||` default, a ternary, or extracting the steps into named functions that each
return a value.

```js
// yes
const tier = { active: "paid", in_trial: "trial" }[status] || "free";

// no
let tier = "free";
if (status === "active") {
  tier = "paid";
} else if (status === "in_trial") {
  tier = "trial";
}
```

The rare legitimate `let` is an accumulator in a genuine loop or a value assigned
once inside a try/catch. If you write one, it should be obvious why a `const`
wouldn't work.

**Arrow function consts, not `function` declarations.** Consistent with no `this`,
and it keeps every definition looking the same.

```js
// yes
const markShipped = async ({ orderId, trackingNumber }) => { ... };

// no
async function markShipped(orderId, trackingNumber) { ... }
```

**Name the steps, don't build one long chain.** Composition means small named
functions applied in sequence. A chain of anonymous callbacks reads inwards, which
the downwards rule already rules out.

```js
// yes
const isActive = ({ status }) => status === "active";
const toSummary = ({ id, plan_id }) => ({ id, plan: plan_id });

const summaries = subscriptions.filter(isActive).map(toSummary);

// no
const summaries = subscriptions
  .filter((s) => s.status === "active" && !s.deleted && s.plan_id)
  .map((s) => ({ id: s.id, plan: s.plan_id, ...(s.trial ? { trial: true } : {}) }));
```

**No shared mutable module state.** Module-level constants are fine: lookup
tables, config, wrapper singletons. A module-level `let` that different functions
reassign is the OOP instance field this convention exists to avoid. If state has
to live somewhere, pass it through arguments and return values.

```js
// yes
const PLAN_LABELS = { pro: "Pro", free: "Free" };
const toLabel = ({ planId }) => PLAN_LABELS[planId] || "Free";

// no
let lastSyncedAt = null; // reassigned from three different functions

const syncOrders = async () => {
  lastSyncedAt = Date.now();
  ...
};
```

**Optional chaining over existence ladders.** `user?.address?.city` instead of a
nested `if` chain, paired with `??` for the default.

```js
// yes
const city = user?.address?.city ?? "Unknown";

// no
let city = "Unknown";
if (user && user.address && user.address.city) {
  city = user.address.city;
}
```

**Transforms take data, not framework objects.** A shaping function takes a plain
object and returns a plain object. It doesn't take `req`, `res`, or a database
snapshot. Unwrap at the route handler and hand the pure function real data, which
is what makes it testable without stubbing an entire request.

```js
// yes
const toOrderPayload = ({ body }) => ({
  productId: body.product_id,
  quantity: body.quantity,
});

// in the route handler
return createOrder(toOrderPayload({ body: req.body }));

// no
const toOrderPayload = (req) => ({
  productId: req.body.product_id,
  quantity: req.body.quantity,
});
```

## Third party services

**Always go through a wrapper.** Never import a vendor SDK directly in feature
code. Every external service gets one wrapper module (e.g. under `config/`) and
everything else imports that.

```js
// yes
const payments = require("../config/payments");
await payments.refund({ chargeId });

// no
const Stripe = require("stripe");
const stripe = new Stripe(process.env.STRIPE_KEY);
await stripe.refunds.create({ charge: chargeId });
```

Why: swapping a vendor, adding retries, adding logging, or stubbing in a test is
one file instead of every call site. If a service has no wrapper yet, add one
rather than importing the SDK inline.

**Don't pass shared singletons as params.** A helper that needs the database
client requires it itself. It doesn't take `db` from every caller.

```js
// yes, in lib/invoices.js
const db = require("../config/db");

const saveInvoice = async ({ invoice }) =>
  db.set({ path: `invoices/${invoice.orderId}`, value: invoice });

// no
const saveInvoice = async ({ db, invoice }) => ...
```

## Database writes

**Scope every write to the record it changes.** No root-level or table-wide
multi-path updates to change one record. A scoped write can only ever touch that
record, so a bad key can't blast across the whole store. Existing broad writes in a
codebase are not license to add new ones. Applies to one-off scripts too.

```js
// yes
db.update({ path: `orders/${orderId}`, value: { status, updated_at } });

// no
db.update({ path: "/", value: { [`orders/${orderId}/status`]: status } });
```

## Error handling

**Throw for broken, return empty for absent.** A missing record is not an error:
return `null` or `[]` and let the caller decide. A broken invariant, a failed
vendor call, or bad input is an error: throw. Don't mix the two by returning
`{ error: "..." }` sentinel objects that every caller has to remember to check.

```js
// yes
const fetchOrder = async ({ orderId }) => {
  const order = await db.get({ path: `orders/${orderId}` });
  return order || null; // absent
};

const shipOrder = async ({ orderId }) => {
  const order = await fetchOrder({ orderId });
  if (!order) throw new Error(`Cannot ship missing order ${orderId}`); // broken
  ...
};

// no
return { error: "order not found" };
```

**Never swallow.** No empty `catch`, no `catch` that logs and continues as if
nothing happened. If you catch, you either recover meaningfully, or you add
context and rethrow.

```js
// yes
try {
  await payments.refund({ chargeId });
} catch (error) {
  throw new Error(`Failed to refund charge ${chargeId}: ${error.message}`, {
    cause: error,
  });
}

// no
try {
  await payments.refund({ chargeId });
} catch (error) {
  console.log(error);
}
```

**Catch at the boundary, not everywhere.** Controllers and helpers let errors
bubble. The route handler is where you turn an error into a status code and a
response body. One try/catch per request path, not one per function.

```js
// yes
router.post("/orders/:orderId/ship", async (req, res) => {
  try {
    return res.json(await shipOrder({ orderId: req.params.orderId }));
  } catch (error) {
    return res.status(500).json({ error: error.message });
  }
});
// shipOrder, fetchOrder, and saveInvoice have no try/catch at all

// no
// a try/catch in the handler, another in shipOrder, another in fetchOrder
```

**Error reporting happens at the boundary too.** Sentry (or whatever you use) is
wired up once, at the route and process level. Don't sprinkle manual
`captureException` calls through helpers that are about to rethrow anyway, that
just reports the same failure several times.

```js
// yes
} catch (error) {
  throw new Error(`Failed to ship order ${orderId}`, { cause: error });
}
// reported once, at the boundary

// no
} catch (error) {
  Sentry.captureException(error); // reported here
  throw error;                    // and again upstream
}
```

**Error messages name the thing.** Include the id, the operation, and the vendor.
`Failed to refund charge ch_123 in Stripe` beats `Request failed`. Never put a
token, key, cookie value, or full user record in the message.

```js
// yes
throw new Error(`Failed to refund charge ${chargeId} in Stripe: ${error.message}`);

// no
throw new Error("Request failed");
throw new Error(`Stripe rejected key ${apiKey}`);
```

## Async

**`await`, not `.then`.** No mixing the two in one function, no promise chains as
control flow, no `.then().catch()` where a try/catch belongs.

```js
// yes
const order = await fetchOrder({ orderId });
return toInvoice({ order });

// no
return fetchOrder({ orderId })
  .then((order) => toInvoice({ order }))
  .catch(() => null);
```

**Run independent work in parallel.** Sequential `await`s that don't depend on
each other are wasted latency.

```js
// yes
const [user, orders] = await Promise.all([
  fetchUser({ userId }),
  fetchOrders({ userId }),
]);

// no
const user = await fetchUser({ userId });
const orders = await fetchOrders({ userId });
```

**Never `await` inside a loop over a collection.** Map to promises and settle them
together. If the vendor has a rate limit, batch in chunks, don't fall back to a
serial loop by default.

```js
// yes
const orders = await Promise.all(orderIds.map((orderId) => fetchOrder({ orderId })));

// no
const orders = [];
for (const orderId of orderIds) {
  orders.push(await fetchOrder({ orderId }));
}
```

Use `Promise.allSettled` when one failure shouldn't kill the batch, and handle the
rejected entries explicitly rather than filtering them away silently.

**No floating promises.** Every promise is awaited or explicitly handed off. An
unawaited async call in a request handler finishes after the response is sent, and
its failure lands in the uncaught handler with no context. If work is genuinely
fire and forget, say so at the call site and attach a catch.

```js
// yes
await analytics.track({ userId, event: "order_shipped" });

// or, deliberately not awaited
analytics
  .track({ userId, event: "order_shipped" })
  .catch((error) => console.error("analytics track failed", error));

// no
analytics.track({ userId, event: "order_shipped" }); // settles after the response
return res.json({ ok: true });
```

**`async` only if it awaits.** A function that returns a promise without awaiting
anything doesn't need the keyword.

```js
// yes
const toInvoice = ({ order }) => ({ orderId: order.id, total: order.total });

// no
const toInvoice = async ({ order }) => ({ orderId: order.id, total: order.total });
```

## Boundary validation

**Validate once at the edge, then trust the shape inward.** Route handlers and
webhook receivers check and coerce. Everything below them assumes the data is
already the right shape and doesn't re-check.

```js
// yes, in the route handler
const { productId, note } = req.body;
if (!productId) return res.status(400).json({ error: "productId is required" });

return createOrder({ productId, note: String(note || "").trim() });
```

**Webhook payloads are untrusted input.** Payment providers and anything else
posting to you: verify the signature or shared secret first, read only the fields
you need, and never trust an id in the body to identify the user without looking
it up. Don't pass a raw webhook body into a write.

```js
// yes
if (!isValidSignature({ headers: req.headers, body: req.rawBody })) {
  return res.status(401).json({ error: "invalid signature" });
}

const { customer_id } = req.body.data;
const user = await findUserByCustomerId({ customerId: customer_id });
if (!user) return res.status(404).json({ error: "unknown customer" });

// no
const { user_id, plan } = req.body.data;
await db.update({ path: `users/${user_id}`, value: { plan } }); // trusts the payload
```

**Coerce at the edge, don't defensively coerce everywhere.** One `Number()`,
`String()`, or date parse at the boundary. Helpers downstream shouldn't be
guessing whether they got a string or a number.

```js
// yes, in the route handler
const limit = Number(req.query.limit) || DEFAULT_LIMIT;
return listOrders({ limit });
// listOrders treats limit as a number and never re-parses it

// no
const listOrders = ({ limit }) => {
  const parsed = typeof limit === "string" ? parseInt(limit, 10) : limit;
  ...
};
```

**Fail fast and specifically.** Return 400 with which field was wrong. Don't let
bad input travel three layers down and surface as a type error.

```js
// yes
if (!productId) return res.status(400).json({ error: "productId is required" });
if (!Array.isArray(items)) return res.status(400).json({ error: "items must be an array" });

// no
// no checks, and `items.length` throws "Cannot read properties of undefined"
// inside calculateTotal three calls later
```

**Never interpolate unvalidated input into a path or query.** A database path or
query built from a raw request value can escape its record. Validate the id shape
first.

```js
// yes
const ID = /^[A-Za-z0-9_-]+$/;
if (!ID.test(orderId)) return res.status(400).json({ error: "invalid orderId" });
db.update({ path: `orders/${orderId}`, value });

// no
db.update({ path: `orders/${req.body.orderId}`, value }); // "x/../../users" escapes the record
```

## Config and secrets

**One secrets manager is the source of truth.** Doppler, Vault, whatever the
project uses. Secrets get added and changed there and synced to the host. No
setting them by hand on the host, no committed `.env` with real values, no secrets
in code or in a lookup table.

```bash
# yes
doppler secrets set PAYMENTS_API_KEY

# no
heroku config:set PAYMENTS_API_KEY=sk_live_... -a my-app
```

**`process.env` is read in `config/` only.** Feature code imports from `config/`
and never touches `process.env` directly. That keeps the list of required env vars
discoverable in one place instead of scattered across controllers.

```js
// yes
const { features } = require("../config");
if (features.enableGiftCards) { ... }

// no
if (process.env.ENABLE_GIFT_CARDS === "true") { ... }
```

**Fail loudly at boot on a missing required secret.** A wrapper that needs a
credential should throw when the app starts, not return `undefined` and fail on
the first request an hour later.

```js
// yes, in config/payments.js
const { PAYMENTS_API_KEY } = process.env;
if (!PAYMENTS_API_KEY) throw new Error("PAYMENTS_API_KEY is not set");

// no
const client = new PaymentsClient({ apiKey: process.env.PAYMENTS_API_KEY });
// undefined key, first 401 shows up in production an hour later
```

**A value that never changes is a constant, not an env var.** Env vars are for
things that actually differ between environments. Don't add config plumbing for a
fixed URL.

```js
// yes
const DOCS_URL = "https://docs.example.com";

// no
const docsUrl = process.env.DOCS_URL || deriveFromRequest({ req }) || FALLBACK_URL;
```

**Never log or return secrets.** No keys, tokens, service account JSON, or cookie
values in logs, error messages, or API responses.

```js
// yes
console.error(`Payments auth failed for account ${accountId}`);

// no
console.error(`Payments auth failed with key ${apiKey}`);
return res.json({ user, serviceAccount: SERVICE_ACCOUNT_JSON });
```

## Scope of changes

**Change only what the task requires.** No refactoring adjacent code, no hoisting
variables, no tidying nearby lines, no fixing pre-existing bugs found along the
way. For an additive change, `git diff` should show additions and zero removed
lines. If you spot a real bug next door, say so and let it be its own task.

```bash
git diff --stat
# yes: 1 file changed, 12 insertions(+)
# no:  1 file changed, 12 insertions(+), 31 deletions(-)
```

**Simplest thing that works.** A value that never changes is a hardcoded constant.
No env var fallback chains, no request-derived plumbing, no new function params to
thread it through. Add abstraction when the value actually varies.

```js
// yes
const DEFAULT_PAGE_SIZE = 50;

// no
const getPageSize = ({ req, env = process.env }) =>
  Number(req.headers["x-page-size"]) || Number(env.PAGE_SIZE) || 50;
```

**"Remove this button" means comment it out.** Comment out the JSX with a short
"uncomment to restore" note and leave the handlers, state, prop plumbing, CSS, and
imports intact. Retire a feature by hiding its entry point and keeping the
infrastructure, so bringing it back is an uncomment instead of a rebuild. Only
delete supporting code when the ask is explicitly to delete it.

```jsx
// yes
{/* Export button hidden 2026-09. Uncomment to restore. */}
{/* <Button onClick={handleExport}>Export CSV</Button> */}
// handleExport, its state, its prop plumbing, and its CSS all stay

// no
// button deleted, along with the handler, the useState, the props, and the CSS
```
