# Layout: TypeScript and JavaScript

Covers plain TS, React, React Native, and Node backends. Read the framework overrides section first, because Next.js and Expo Router change the top of the tree.

---

## Folder Layout

```
src/
├── features/           # one folder per feature, sealed behind a barrel
├── components/         # reusable UI, cross feature, styles in their own file
├── services/           # all DB, API, and third party SDK calls
├── hooks/              # shared stateful hooks, cross feature
├── store/              # app wide state, only if two or more features share state
├── utils/              # pure helpers, dates, formatting, error parsing
├── constants/          # design tokens and fixed values, depends on nothing
├── types/              # shared domain types and the Result shape, lowest layer
├── config/             # client setup and one typed env module, no queries
└── navigation/         # routing config and route files, if the framework does not own them
```

Outside `src/`:

```
e2e/                    # end to end flows, they cross features
scripts/                # seeds, migrations, one off tooling
```

---

## The Dependency Law, Mapped

Law 1 translated into this layout's folder names. The diagram in `SKILL.md` stays the authoritative copy.

```
features/    can use →  store, services, hooks, components, utils, types
hooks/       can use →  store, services, utils, types
components/  can use →  store, utils, types
store/       can use →  services, utils, types
services/    can use →  utils, types, config
utils/       can use →  types only
types/       can use →  nothing
```

`constants/` sits beside `types/`: it depends on nothing, and every layer except `types/` and `config/` may read it. `config/` is read by `services/` only. `navigation/` sits above `features/`: it imports public doors and nothing else, and nothing imports it.

### State, and where it lives

The store rules are Law 1 in `SKILL.md`. The TypeScript mechanics:

```
src/store/              # or contexts/, or providers/, pick one and keep it
├── sessionStore.ts     # one holder per domain, not one for the app
└── cartStore.ts
```

- Export a hook shaped surface from the store file so components do not know which library is underneath. Swapping the library then touches one file per store rather than every consumer.
- A remote data cache sits in `hooks/` and wraps a function from `services/`. The query hook must call the service, never the API directly, or the service layer and its validation gate quietly disappear.

```ts
// src/hooks/useSlots.ts
import { useQuery } from '@tanstack/react-query';
import { fetchSlots } from '@services/bookingService';

export function useSlots(date: string) {
  return useQuery({ queryKey: ['slots', date], queryFn: () => fetchSlots(date) });
}
```

The library shown is one example. Swap it for any cache, keep the shape.

---

## Feature Anatomy

```
src/features/auth/
├── index.ts              # the only public door, exports only, no logic
├── AuthScreen.tsx        # orchestrator, composes, minimal logic
├── useAuth.ts            # bridges UI and services
├── authValidator.ts      # validation rules
├── authTypes.ts          # types and constants local to this feature
└── SocialLoginRow.tsx    # feature scoped sub component
```

```ts
// src/features/auth/index.ts
export { AuthScreen } from './AuthScreen';
```

The screen is the whole contract; `useAuth` stays private, because nothing above a feature should reach its hook (session state two features need lives in the store layer, not behind one feature's door). Outside code imports from `features/auth`, never from `features/auth/authValidator`. That is what lets you rewrite the internals without breaking a caller.

### Orchestrator rules
- Imports and composes sub components and hooks.
- May hold local UI state.
- No raw API calls, no DB logic, no business rules.
- Past roughly 120 lines, extract a sub component or hook (Law 13).

### Barrel cost, worth knowing
One barrel per feature is correct. A barrel in every folder is not. Deep barrel chains cause circular imports, break tree shaking, and slow down dev server reloads. Keep exactly one barrel per feature and none anywhere else.

---

## Naming

| Type | Convention | Example |
|---|---|---|
| Component | PascalCase | `BookingCard.tsx` |
| Hook | camelCase, `use` prefix | `useBooking.ts` |
| Logic or util | camelCase | `dateFormatter.ts` |
| Service | camelCase noun | `bookingService.ts` |
| Style file | matches component, `.styles` | `bookingCard.styles.ts` |
| Type file | camelCase, `Types` suffix | `bookingTypes.ts` |
| Feature folder | camelCase | `auth/`, `bookingSlots/` |
| Shared component folder | PascalCase | `components/Button/` |
| Public door | `index.ts` | barrel only, never logic |

---

## The Result Shape and the Service Pattern

Law 12: one error shape, declared once in the type layer, imported by every service. A discriminated union, so the compiler makes the caller handle the failure branch.

```ts
// src/types/result.ts
export type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };
```

Services are named async functions only. Each handles its own failure and returns a `Result`. No UI, no navigation, no state.

```ts
// src/services/bookingService.ts
import { db } from '@config/dbClient';
import { parseError } from '@utils/errorHandler';
import type { Result } from '@domain/result';
import type { Slot } from '@domain/booking';

function toSlot(raw: unknown): Slot {
  const row = raw as Record<string, unknown>;
  if (typeof row?.id !== 'string') throw new Error('Slot is missing an id');
  return row as Slot;
}

export async function fetchSlots(date: string): Promise<Result<Slot[]>> {
  try {
    const response = await db.from('slots').select('*').eq('date', date);
    if (response.error) return { ok: false, error: parseError(response.error) };
    return { ok: true, data: response.data.map(toSlot) };
  } catch (cause) {
    return { ok: false, error: parseError(cause) };
  }
}
```

`toSlot` is the Law 10 gate: rows are checked here, before anything above this layer sees them. Imports cross layers by alias rather than by `../`, so the layer is visible in the import line and a moved file does not change it. The client shown is one example. Swap it for any backend, keep the shape.

---

## Config Pattern

One file reads raw env. Nothing else does.

```ts
// src/config/env.ts
function required(value: string | undefined, name: string): string {
  if (!value) throw new Error(`Missing required env var: ${name}`);
  return value;
}

export const config = {
  apiUrl: required(process.env.EXPO_PUBLIC_API_URL, 'EXPO_PUBLIC_API_URL'),
  isProd: process.env.NODE_ENV === 'production',
} as const;
```

Read each variable by its literal name, and pass the name in as a second argument only so the error message can say it. A helper that does `process.env[name]` with a variable key looks tidier and does not work: Metro, Vite, and Webpack replace `process.env.SOMETHING` and `import.meta.env.SOMETHING` by matching that exact text at build time. There is no environment object left at runtime to index into, so every lookup returns `undefined` and a config module written that way throws on every variable in a production build. Use `import.meta.env.VITE_X` on Vite and `process.env.EXPO_PUBLIC_X` on Expo, always spelled out.

On React Native and any web bundler, remember that everything in the bundle is readable. `EXPO_PUBLIC_` and `NEXT_PUBLIC_` values are shipped to the user. Law 9: never put a secret behind one.

---

## Styling

A component file describes behavior and structure, not a wall of appearance.

```
src/components/Button/
├── Button.tsx
└── button.styles.ts
```

- Web React: a CSS module or a dedicated styles file per component.
- React Native: short utility classes or a small inline `StyleSheet` is fine, but extract to a styles file the moment it grows past a few lines.
- A component used by only one feature stays in that feature folder, styles file included.

### Design tokens, the one thing that is not per component

Colours, spacing, radii, and font sizes are values with no dependencies, read by almost every file. That places them at the bottom of the layer model beside types, not in a folder of their own halfway up the tree.

```
src/constants/theme.ts     # or src/styles/tokens.ts, one file, exported as const
```

Keep this to tokens. The moment a global styles folder starts collecting whole component stylesheets, it has become a shared dumping ground, and you can no longer tell which of those files anything still uses. A stylesheet belongs beside the component it dresses. A colour that three features share does not.

---

## Types

- Shared domain types and `Result` in `types/`, depending on nothing.
- Feature only types stay in the feature.
- `strict: true` in `tsconfig.json`. Ban `any`: each one is a hole where the compiler stops protecting you. Use `unknown` and narrow it.
- Where a tool can generate types from your database schema or API spec, use them so code and schema cannot drift.

---

## Enforce the Dependency Law Automatically

Do not rely on discipline. Fail the build.

Path aliases first, so imports read as layers:

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@features/*":   ["src/features/*"],
      "@services/*":   ["src/services/*"],
      "@components/*": ["src/components/*"],
      "@hooks/*":      ["src/hooks/*"],
      "@store/*":      ["src/store/*"],
      "@utils/*":      ["src/utils/*"],
      "@constants/*":  ["src/constants/*"],
      "@config/*":     ["src/config/*"],
      "@domain/*":     ["src/types/*"]
    }
  }
}
```

Declare an alias for every layer you import across, or the aliased imports and the relative ones will sit side by side and neither reads as a layer.

Name the types alias `@domain/*` or `@apptypes/*`, never `@types/*`: `node_modules/@types` is where DefinitelyTyped lives, so that prefix already means something to every reader and to some tools, and you are one `import type { X } from '@types/node'` away from an ambiguous specifier.

Whatever you choose, every tool that resolves modules needs the same list: `tsconfig.json`, plus the Vite, Webpack, or Metro config, plus Jest's `moduleNameMapper`. An alias that exists in only one of them fails at the least convenient moment.

Then block every illegal direction from the map above with `eslint-plugin-boundaries`, or with the built in `import/no-restricted-paths` from `eslint-plugin-import`. Both resolve import paths through a resolver, and the default one does not understand `tsconfig` paths: install `eslint-import-resolver-typescript` and point it at `tsconfig.json`, otherwise every `@services/*` style import is unresolvable and the rule skips it without a word. One zone per layer, listing what that layer may not import:

```js
// eslint.config.js
// target = the layer being restricted, from = what it may not import
import importPlugin from 'eslint-plugin-import';

export default [
  {
    files: ['src/**/*.{ts,tsx}'],
    plugins: { import: importPlugin },
    settings: {
      'import/resolver': { typescript: { project: './tsconfig.json' } },
    },
    rules: {
      'import/no-restricted-paths': ['error', {
        zones: [
          { target: './src/features',   from: ['./src/navigation', './src/config'] },
          { target: './src/hooks',      from: ['./src/features', './src/components', './src/navigation', './src/config'] },
          { target: './src/components', from: ['./src/features', './src/services', './src/hooks', './src/navigation', './src/config'] },
          { target: './src/store',      from: ['./src/features', './src/components', './src/hooks', './src/navigation', './src/config'] },
          { target: './src/services',   from: ['./src/features', './src/components', './src/hooks', './src/store', './src/navigation'] },
          { target: './src/navigation', from: ['./src'], except: ['./navigation', './features'] },
          { target: './src/utils',      from: ['./src'], except: ['./utils', './constants', './types'] },
          { target: './src/config',     from: ['./src'], except: ['./config'] },
          { target: './src/constants',  from: ['./src'], except: ['./constants'] },
          { target: './src/types',      from: ['./src'], except: ['./types'] },
        ],
      }],
    },
  },
];
```

`except` paths are relative to `from`. Prove the rule is live before trusting it: add one deliberately illegal aliased import, run the linter, and watch it fail. Blocking sibling feature imports and deep imports past a barrel needs `eslint-plugin-boundaries`, which understands element types rather than raw paths. Add it on any codebase with more than a handful of features.

---

## Testing

Mirror the source tree. Tests sit beside the file they cover.

```
src/services/bookingService.ts
src/services/bookingService.test.ts
src/utils/dateFormatter.ts
src/utils/dateFormatter.test.ts
```

End to end flows live in a top level `e2e/`, because they cross features.

---

## Framework Overrides

The framework wins. Apply the Laws inside its shape.

### Next.js App Router
`app/` is routing, owned by the framework. It is not your feature layer.

```
app/                    # routes only, thin. A page imports one feature and renders it.
src/
├── features/
├── components/
├── services/
└── ...
```

A `page.tsx` should be a few lines that import a feature's public door and render it. Server actions and route handlers belong to the service layer in spirit, so keep the data logic in `src/services/` and let the route handler call it.

### Expo Router
Same idea. `app/` is file based routing. Keep `src/features/` beside it and let each route render one feature.

```
app/
├── _layout.tsx
├── index.tsx           # imports and renders BookingScreen from features/bookingSlots
src/
├── features/
└── ...
```

### Plain React Native with React Navigation
No override. Use `src/navigation/` as shown in the base layout.

### Node backends, Express, Nest, Fastify
Nest dictates modules, controllers, providers. Follow Nest. For Express and Fastify, the base layout applies, with route registration playing the role of `navigation/`.
