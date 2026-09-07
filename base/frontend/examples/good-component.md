# Example: Good Component Pattern

Illustrates the atomic/composite split from
[`../component-hierarchy.md`](../component-hierarchy.md).

```tsx
// components/Button.tsx — atomic, purely presentational
export function Button({ children, variant = "primary", ...props }: ButtonProps) {
  return (
    <button className={buttonStyles[variant]} {...props}>
      {children}
    </button>
  );
}

// components/ConfirmDialog.tsx — composite, local UI state only
export function ConfirmDialog({ title, message, onConfirm, onCancel }: ConfirmDialogProps) {
  return (
    <Dialog onClose={onCancel}>
      <h2>{title}</h2>
      <p>{message}</p>
      <Button variant="secondary" onClick={onCancel}>Cancel</Button>
      <Button variant="danger" onClick={onConfirm}>Confirm</Button>
    </Dialog>
  );
}

// features/orders/CancelOrderButton.tsx — feature, owns the business logic
export function CancelOrderButton({ orderId }: { orderId: string }) {
  const [showConfirm, setShowConfirm] = useState(false);
  const { mutate: cancelOrder, isLoading } = useCancelOrder(); // server state, isolated

  return (
    <>
      <Button onClick={() => setShowConfirm(true)} disabled={isLoading}>
        Cancel order
      </Button>
      {showConfirm && (
        <ConfirmDialog
          title="Cancel this order?"
          message="This can't be undone."
          onConfirm={() => cancelOrder(orderId)}
          onCancel={() => setShowConfirm(false)}
        />
      )}
    </>
  );
}
```

Why this is good:
- `Button` has zero knowledge of orders — fully reusable.
- `ConfirmDialog` is generic, reused for any confirmation, not just cancel-order.
- Business logic (`useCancelOrder`, the actual API call) lives only in the
  feature layer.
