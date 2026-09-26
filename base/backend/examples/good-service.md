# Example: Good Service Pattern

Illustrates package by component and layering (see [`../architecture.md`](../architecture.md)),
repository interfaces (see [`../data-access.md`](../data-access.md#repositories)), and
error handling (see [`../../shared/error-handling.md`](../../shared/error-handling.md)).

```typescript
// src/bookstore/orders/order.ts — private; no framework/DB dependency
export class Order {
  constructor(private items: OrderItem[]) {}

  calculateTotal(): Money {
    return this.items.reduce((sum, i) => sum.add(i.price), Money.zero());
  }
}

// src/bookstore/orders/order-repository.ts — private; the repository
// interface, defined with the domain model and speaking only domain types
export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// src/bookstore/orders/cancel-order.ts — private; orchestration only,
// depends on the interface, never on the ORM
export class CancelOrder {
  constructor(
    private orders: OrderRepository,
    private payments: PaymentGateway,
  ) {}

  async execute(orderId: OrderId): Promise<Result<void, CancelOrderError>> {
    const order = await this.orders.findById(orderId);
    if (!order) return Result.err(new OrderNotFoundError(orderId));
    if (!order.isCancellable()) {
      return Result.err(new OrderNotCancellableError(orderId, order.status));
    }

    await this.payments.refund(order.paymentId);
    order.markCancelled();
    await this.orders.save(order);

    return Result.ok();
  }
}

// src/bookstore/orders/postgres-order-repository.ts — private; the only
// file that knows the ORM. Maps rows ↔ domain; ORM types never leave it.
export class PostgresOrderRepository implements OrderRepository {
  constructor(private db: Kysely<Database>) {}

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.selectFrom("orders").selectAll()
      .where("id", "=", id).executeTakeFirst();
    return row ? toDomain(row) : null;
  }

  async save(order: Order): Promise<void> {
    const row = toRow(order);
    await this.db.insertInto("orders").values(row)
      .onConflict((oc) => oc.column("id").doUpdateSet(row)).execute();
  }
}

// src/bookstore/orders/index.ts — the component's public API: use cases,
// their error types, and a factory that takes infrastructure handles
export type { CancelOrderError } from "./cancel-order";
export function createOrders(deps: { db: Kysely<Database>; payments: PaymentGateway }) {
  const orders = new PostgresOrderRepository(deps.db);
  return { cancelOrder: new CancelOrder(orders, deps.payments) };
}

// src/api/main.ts — composition root: builds infrastructure, wires components
const orders = createOrders({ db: createDb(config.databaseUrl), payments });
app.use(orderRoutes(orders));

// src/api/routes/orders.ts — delivery mechanism: maps to HTTP, no business logic
import type { createOrders } from "../../bookstore/orders"; // entry point only

export const orderRoutes = ({ cancelOrder }: ReturnType<typeof createOrders>) =>
  Router().post("/orders/:id/cancel", async (req, res) => {
    const result = await cancelOrder.execute(OrderId.parse(req.params.id));
    if (result.isErr()) return res.status(toHttpStatus(result.error)).json({ error: result.error.toResponse() });
    return res.status(204).send();
  });
```

Why this is good:
- Domain (`Order`) has zero framework dependencies — testable in isolation.
- `CancelOrder` orchestrates without containing business rules itself.
- `OrderRepository` is defined with the domain model and speaks only domain
  types; `CancelOrder` depends on it, not on the ORM. The Postgres
  implementation is the only file that imports the ORM, and it maps rows to
  `Order` so no ORM type crosses the boundary. A unit test can pass an
  in-memory fake instead.
- The composition root (`api/main.ts`) supplies the database handle; the
  component's factory picks the implementation, so it stays private.
- The route imports only the `orders` component's entry point; `Order` and
  the repository stay private, so they can change without touching `api/`.
- Errors are typed and mapped to stable HTTP responses at the boundary, per
  the shared error-handling contract — no raw exceptions leaking out.
