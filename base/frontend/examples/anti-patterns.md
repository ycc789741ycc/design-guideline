# Example: Frontend Anti-Patterns

## Business logic in an "atomic" component

```tsx
// ❌ Bad — a "reusable" button that isn't actually reusable
function Button({ orderId }: { orderId: string }) {
  const cancelOrder = useCancelOrder();
  return <button onClick={() => cancelOrder(orderId)}>Cancel</button>;
}
```

```tsx
// ✅ Good — generic button, business logic pushed to the caller
function Button({ children, onClick }: ButtonProps) {
  return <button onClick={onClick}>{children}</button>;
}
```

## Server data stuffed into global state

```tsx
// ❌ Bad — manual global store duplicating the server as source of truth
const useOrdersStore = create((set) => ({
  orders: [],
  fetchOrders: async () => set({ orders: await api.getOrders() }),
}));
```

```tsx
// ✅ Good — a data-fetching layer with built-in caching/invalidation
function useOrders() {
  return useQuery(["orders"], () => api.getOrders());
}
```

## Div-as-button

```tsx
// ❌ Bad — no keyboard access, no semantics
<div onClick={handleClick}>Submit</div>
```

```tsx
// ✅ Good
<button onClick={handleClick}>Submit</button>
```
