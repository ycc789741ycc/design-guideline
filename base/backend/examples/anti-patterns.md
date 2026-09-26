# Example: Backend Anti-Patterns

## Swallowed errors

```typescript
// ❌ Bad — error silently disappears
async function cancelOrder(id: string) {
  try {
    await orders.cancel(id);
  } catch (e) {
    console.log("something went wrong");
  }
}
```

```typescript
// ✅ Good — typed error propagated to caller
async function cancelOrder(id: string): Promise<Result<void, CancelOrderError>> {
  try {
    await orders.cancel(id);
    return Result.ok();
  } catch (e) {
    return Result.err(new CancelOrderError(id, e));
  }
}
```

## Business logic leaking into controllers

```typescript
// ❌ Bad — discount calculation, a domain rule, lives in the HTTP layer
router.post("/orders", async (req, res) => {
  let total = req.body.items.reduce((s, i) => s + i.price, 0);
  if (req.body.items.length > 10) total *= 0.9; // bulk discount — a domain rule
  const order = await db.orders.insert({ ...req.body, total });
  res.json(order);
});
```

```typescript
// ✅ Good — discount rule lives in the domain, controller just orchestrates
router.post("/orders", async (req, res) => {
  const result = await createOrder.execute(req.body);
  res.json(result);
});
```

## Reaching into a component's internals

```typescript
// ❌ Bad — the route imports the orders component's private repository
// src/api/routes/orders.ts
import { OrderRepository } from "../../bookstore/orders/order-repository";

router.get("/orders/:id", async (req, res) => {
  res.json(await orderRepository.get(req.params.id));
});
```

```typescript
// ✅ Good — only the component's entry point is imported
// src/api/routes/orders.ts
import { GetOrder } from "../../bookstore/orders";

router.get("/orders/:id", async (req, res) => {
  const result = await getOrder.execute(req.params.id);
  if (result.isErr()) return res.status(toHttpStatus(result.error)).json({ error: result.error.toResponse() });
  return res.json(result.value);
});
```

## Use case talking to the ORM directly

```typescript
// ❌ Bad — the use case imports the ORM, and the "repository" hands back
// ORM rows, so the domain is coupled to the table layout
import { db } from "../../infra/db";

export class CancelOrder {
  async execute(orderId: string) {
    const row = await db.selectFrom("orders").selectAll()
      .where("id", "=", orderId).executeTakeFirst();
    if (row?.status !== "paid") return;   // a domain rule on a DB row
    await db.updateTable("orders").set({ status: "cancelled" })
      .where("id", "=", orderId).execute();
  }
}
```

```typescript
// ✅ Good — the use case depends on a repository interface defined with the
// domain model; only the Postgres implementation knows the ORM
export interface OrderRepository {
  create(order: Order): Promise<Order>;
  get(id: OrderId): Promise<Order | null>;
  getList(filter: OrderFilter, page?: number, pageSize?: number | null): Promise<Page<Order>>;
  update(order: Order): Promise<Result<Order, OrderNotFoundError>>;
  delete(id: OrderId): Promise<Result<void, OrderNotFoundError>>;
}

export class CancelOrder {
  constructor(private orders: OrderRepository) {}

  async execute(orderId: OrderId): Promise<Result<void, CancelOrderError>> {
    const order = await this.orders.get(orderId);
    if (!order) return Result.err(new OrderNotFoundError(orderId));
    const cancelled = order.cancel();   // the rule lives on the entity
    if (cancelled.isErr()) return cancelled;
    const updated = await this.orders.update(order);
    return updated.isErr() ? updated : Result.ok();
  }
}
```

## A finder method per query

```typescript
// ❌ Bad — every new question grows the interface, each repository ends up
// with a different shape, and ordering/pagination differ per method
export interface BookRepository {
  findById(id: BookId): Promise<Book | null>;
  findByAuthor(authorId: AuthorId): Promise<Book[]>;
  findRecentByStatus(status: BookStatus, limit: number): Promise<Book[]>;
  findAllPublishedAfter(date: Date): Promise<Book[]>;
  save(book: Book): Promise<void>;
}
```

```typescript
// ✅ Good — the standard five methods; each question is a filter field, and
// every list comes back newest first and paginated the same way
export interface BookRepository {
  create(book: Book): Promise<Book>;
  get(id: BookId): Promise<Book | null>;
  getList(filter: BookFilter, page?: number, pageSize?: number | null): Promise<Page<Book>>;
  update(book: Book): Promise<Result<Book, BookNotFoundError>>;
  delete(id: BookId): Promise<Result<void, BookNotFoundError>>;
}

const recent = await books.getList(
  new BookFilter({ authorId, status: "published" }), 1, 20,
);
```

## N+1 queries

```typescript
// ❌ Bad
const orders = await db.orders.findAll();
for (const order of orders) {
  order.customer = await db.customers.findById(order.customerId);
}
```

```typescript
// ✅ Good
const orders = await db.orders.findAllWithCustomers(); // single joined query
```
