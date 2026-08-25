# Worked Example: One Feature, End to End

Every other file in this skill tells. This one shows. It follows a single feature from the bottom of the layer model to the top, in the order you would actually build it, so the dependency law is something you can read in the import lines rather than something you have to take on trust.

The stack is TypeScript and React, because it needs to be something. The Laws are language neutral and the shape below transfers unchanged. Translate the folder names with the Step 0 table.

**The feature:** a user sees a list of invoices and marks one as paid.

**Step 0 first.** Vite and React dictate no layout beyond `src/`, and this is a new project, so the base TypeScript layout from `lang-typescript.md` applies. Nothing here is invented for the example.

**Why build bottom up.** Each file below depends only on files already written. The project compiles at every step, and you never write an import to something that does not exist yet. This is the order Adding a New Feature and the refactor section both use, for the same reason.

**Aliases used below.** `@domain/*`, `@utils/*`, `@config/*`, `@services/*`, `@components/*`, `@features/*`, all declared in `tsconfig.json` as shown in `lang-typescript.md`.

---

## 1. The type layer

```ts
// src/types/result.ts
export type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };
```

```ts
// src/types/invoice.ts
export type InvoiceStatus = 'draft' | 'sent' | 'paid';

export type Invoice = {
  id: string;
  reference: string;
  amountCents: number;
  currency: string;
  status: InvoiceStatus;
  dueAt: string;
};
```

Both depend on nothing. Every layer above may read them. `Result` is Law 12's one error shape, and it lives here so every service imports the same one rather than declaring its own.

**Skip this hop and:** the same shape gets redeclared in the service, the hook, and the component. They drift, and the compiler cannot tell you which one is right because it has never seen them side by side.

---

## 2. The shared logic layer

Two files here. One formats money, one turns any thrown thing into a message. Both are pure, both are cheap to test, and both are imported by layers above without either knowing who calls them.

```ts
// src/utils/money.ts
export function formatMoney(amountCents: number, currency: string, locale: string): string {
  return new Intl.NumberFormat(locale, { style: 'currency', currency })
    .format(amountCents / 100);
}
```

```ts
// src/utils/money.test.ts
import { formatMoney } from './money';

test('renders cents as a currency amount', () => {
  // Intl separates the symbol with a non breaking space, so normalise before comparing.
  const output = formatMoney(125000, 'MYR', 'en-MY').replace(/ /g, ' ');
  expect(output).toBe('RM 1,250.00');
});
```

The locale is a parameter, not a default. `Intl` with no locale reads the machine's, so the same test passes on your laptop and fails in CI. A function whose output depends on where it runs is a function you cannot write a test for.

```ts
// src/utils/errorHandler.ts
export function parseError(cause: unknown): string {
  if (cause instanceof Response) return `Request failed with status ${cause.status}`;
  if (cause instanceof Error) return cause.message;
  return 'Something went wrong';
}
```

One function turns anything into a string a person can read. This is Law 12's single parser, and it is the reason no screen below ever inspects an error.

**Skip this hop and:** the division by 100 ends up inline in the component, untested, and every screen invents its own way of describing a failure.

---

## 3. The config layer

```ts
// src/config/env.ts
function required(value: string | undefined, name: string): string {
  if (!value) throw new Error(`Missing required env var: ${name}`);
  return value;
}

export const config = {
  apiUrl: required(import.meta.env.VITE_API_URL, 'VITE_API_URL'),
} as const;
```

Each variable is written out in full, for the bundler reason in the Config Pattern section of `lang-typescript.md`. This file throws at boot, which is the one place throwing is right: there is no caller to return a result to yet, and an app with no API URL should not start.

**Skip this hop and:** a missing variable surfaces as a fetch to `undefined/invoices` three screens in, on a device you do not have.

---

## 4. The service layer

```ts
// src/services/invoiceService.ts
import { config } from '@config/env';
import { parseError } from '@utils/errorHandler';
import type { Result } from '@domain/result';
import type { Invoice, InvoiceStatus } from '@domain/invoice';

const STATUSES: InvoiceStatus[] = ['draft', 'sent', 'paid'];

function toInvoice(raw: unknown): Invoice {
  const row = raw as Record<string, unknown>;
  for (const key of ['id', 'reference', 'currency', 'dueAt'] as const) {
    if (typeof row?.[key] !== 'string') throw new Error(`Invoice ${key} is missing`);
  }
  if (typeof row.amountCents !== 'number') throw new Error('Invoice amount is not a number');
  if (!STATUSES.includes(row.status as InvoiceStatus)) throw new Error('Unknown invoice status');
  return row as Invoice;
}

export async function fetchInvoices(): Promise<Result<Invoice[]>> {
  try {
    const res = await fetch(`${config.apiUrl}/invoices`, { credentials: 'include' });
    if (!res.ok) return { ok: false, error: parseError(res) };
    const body = (await res.json()) as unknown[];
    return { ok: true, data: body.map(toInvoice) };
  } catch (cause) {
    return { ok: false, error: parseError(cause) };
  }
}

export async function markInvoicePaid(id: string): Promise<Result<Invoice>> {
  if (!id) return { ok: false, error: 'Invoice id is required' };
  try {
    const res = await fetch(`${config.apiUrl}/invoices/${id}/paid`, {
      method: 'POST',
      credentials: 'include',
    });
    if (!res.ok) return { ok: false, error: parseError(res) };
    return { ok: true, data: toInvoice(await res.json()) };
  } catch (cause) {
    return { ok: false, error: parseError(cause) };
  }
}
```

Four things are happening here and each one is a law.

`Result` comes from the type layer, so every service in the app returns the same shape and no caller ever writes `try`. `toInvoice` actually checks the response rather than asserting a type onto it, because `as Invoice` tells the compiler to stop looking and tells the runtime nothing. A malformed row now fails here, at the edge, instead of as a blank screen three components later. The host comes from `config.apiUrl`, so this file does not know or care where the API lives. And the identity of the user appears nowhere: the session cookie rides along and the server decides who this is. A client sent `userId` would be a field a hostile caller can type by hand.

In a real project a schema library does `toInvoice` better than hand written checks. The point is that something checks.

**Skip this hop and:** the fetch sits in the component. The validation gate goes with it, error handling is duplicated per screen, and swapping the backend means editing every screen that touched it.

---

## 5. Feature scoped state

```ts
// src/features/invoices/useInvoices.ts
import { useCallback, useEffect, useState } from 'react';
import { fetchInvoices, markInvoicePaid } from '@services/invoiceService';
import type { Invoice } from '@domain/invoice';

export function useInvoices() {
  const [invoices, setInvoices] = useState<Invoice[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  const load = useCallback(async () => {
    setLoading(true);
    const result = await fetchInvoices();
    if (result.ok) {
      setInvoices(result.data);
      setError(null);
    } else {
      setError(result.error);
    }
    setLoading(false);
  }, []);

  useEffect(() => { void load(); }, [load]);

  async function pay(id: string) {
    const result = await markInvoicePaid(id);
    if (!result.ok) return setError(result.error);
    await load();
  }

  return { invoices, error, loading, pay };
}
```

This state belongs to one feature, so it stays inside that feature. The day a second feature needs the invoice list, this hook moves to `hooks/` as a shared cache over the service, and not before. It does not go to the store layer: Law 1 says a cache of server data is a thin wrapper over the service, not a store.

Notice it decides nothing about appearance. It exposes `error` as a string and lets the screen decide whether that is a toast, a banner, or a retry button. And because `Result` is a discriminated union, the compiler will not let it read `result.data` until it has checked `result.ok`.

**Skip this hop and:** loading and error state ends up in the screen, mixed with layout, and the screen crosses its length signal for reasons that have nothing to do with UI.

---

## 6. The feature's own component

```tsx
// src/features/invoices/InvoiceRow.tsx
import { formatMoney } from '@utils/money';
import type { Invoice } from '@domain/invoice';

export function InvoiceRow({ invoice, onPay }: { invoice: Invoice; onPay: () => void }) {
  return (
    <article>
      <span>{invoice.reference}</span>
      <span>{formatMoney(invoice.amountCents, invoice.currency, navigator.language)}</span>
      {invoice.status !== 'paid' && <button onClick={onPay}>Mark paid</button>}
    </article>
  );
}
```

Used by one feature, so it lives in that feature rather than in `components/`. It calls `formatMoney` itself, because this is the point where the number becomes something a person reads. The locale comes from the browser here, at the edge of the app, rather than being baked into the helper.

---

## 7. The orchestrator

```tsx
// src/features/invoices/InvoiceListScreen.tsx
import { useInvoices } from './useInvoices';
import { InvoiceRow } from './InvoiceRow';
import { ErrorBanner } from '@components/ErrorBanner';
import { Spinner } from '@components/Spinner';

export function InvoiceListScreen() {
  const { invoices, error, loading, pay } = useInvoices();

  if (loading) return <Spinner />;

  return (
    <main>
      {error && <ErrorBanner message={error} />}
      {invoices.map((invoice) => (
        <InvoiceRow key={invoice.id} invoice={invoice} onPay={() => pay(invoice.id)} />
      ))}
    </main>
  );
}
```

It composes and nothing else. No fetch, no arithmetic, no formatting. `ErrorBanner` and `Spinner` come from the shared UI layer because more than one feature shows a spinner.

**Skip this hop and:** there is no seam. The screen becomes the service, the state, and the layout at once, and it is the file every future change has to touch.

---

## 8. The public door

```ts
// src/features/invoices/index.ts
export { InvoiceListScreen } from './InvoiceListScreen';
```

One line, and it is the whole contract. `useInvoices` and `InvoiceRow` stay private and can be renamed, split, or deleted without a single caller noticing.

**Skip this hop and:** something imports `features/invoices/useInvoices` directly, and now that hook is public API you did not agree to.

---

## 9. The route

```tsx
// src/navigation/InvoicesRoute.tsx
import { InvoiceListScreen } from '@features/invoices';

export default function InvoicesRoute() {
  return <InvoiceListScreen />;
}
```

Routing lives in `navigation/` because Vite owns no routing folder. The route imports one door and renders it. On Next.js or Expo Router this file lives in `app/` instead and looks the same.

---

## The whole feature

```
src/
├── types/result.ts                         no imports
├── types/invoice.ts                        no imports
├── utils/money.ts                          nothing
├── utils/money.test.ts                     the file beside it
├── utils/errorHandler.ts                   nothing
├── config/env.ts                           nothing
├── services/invoiceService.ts              config, utils, types
├── components/ErrorBanner.tsx              already existed, takes a string prop
├── components/Spinner.tsx                  already existed, takes nothing
├── features/invoices/
│   ├── useInvoices.ts                      services, types
│   ├── InvoiceRow.tsx                      utils, types
│   ├── InvoiceListScreen.tsx               feature files, shared components
│   └── index.ts                            the door
└── navigation/InvoicesRoute.tsx            the door only
```

Twelve new files for this feature, plus two shared components that any second feature would have needed anyway. Read the right hand column downward: every file imports only entries listed above it, never below. That is the entire dependency law, and it is now checkable by eye as well as by the lint rule.

`ErrorBanner` and `Spinner` are not written out here because there is nothing to see. They take a prop and render. That is the whole point of the shared UI layer: it does not know what an invoice is, so it survives every change to this feature.

## What each hop bought you

| If this changes | You edit | You do not touch |
|---|---|---|
| The API moves to a new host | `config/env.ts` | anything else |
| The response shape changes | `types/invoice.ts` and `toInvoice` | no component |
| The list needs pagination | `useInvoices.ts` | no service, no screen layout |
| The design changes | the screen and the row | no logic, no service |
| A second feature needs invoices | the hook moves to `hooks/` | the service, unchanged |
| The backend starts returning bad rows | `toInvoice` alone | every screen still shows one clean message |

None of those rows is free in a single file version of this feature. That is what the split bought.

---

**On this example.** Invoices are an illustration. Nothing here is a rule about billing, and none of these file names is required. What transfers is the order of construction, the direction of every import, and the fact that each layer could be deleted and rewritten without disturbing the one below it.
