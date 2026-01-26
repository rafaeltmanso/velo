# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Common Commands

### Install dependencies

```bash
# using yarn (recommended in README)
yarn install

# alternatively, using npm
npm install
```

### Run the app in development

```bash
yarn dev
# or
npm run dev
```

The app runs by default at `http://localhost:5173`.

### Build and preview

```bash
# production build
yarn build
# or
npm run build

# preview the built app locally
yarn preview
# or
npm run preview
```

### Lint

```bash
yarn lint
# or
npm run lint
```

### Playwright end-to-end tests

Playwright is configured in `playwright.config.ts` with tests under `playwright/e2e`.

Run all tests (HTML report by default):

```bash
# ensure dependencies are installed first
yarn playwright test
# or
npx playwright test
```

Run a single test file:

```bash
npx playwright test playwright/e2e/example.spec.ts
```

Open the HTML report (after a test run):

```bash
npx playwright show-report
```

> Note: The `webServer` section in `playwright.config.ts` is commented out. If you add app-specific E2E tests, make sure the dev server is running separately (e.g. `yarn dev`) or configure `webServer` accordingly.

### Supabase CLI workflows (from README)

The README assumes `supabase` CLI is installed as a dev dependency and exposes the following flows:

```bash
# (from README; adjust to your actual scripts if they differ)
yarn install -D supabase   # add Supabase CLI as dev dependency

yarn login                 # supabase login
yarn link --project-ref SEU_PROJECT_ID

yarn db push               # apply database migrations
yarn supabase functions deploy
```

Supabase requires the following Vite env vars (set in `.env`):

- `VITE_SUPABASE_PROJECT_ID`
- `VITE_SUPABASE_PUBLISHABLE_KEY`
- `VITE_SUPABASE_URL`

These are documented in `README.md`.

---

## High-Level Architecture

### Overall

This repository is a React 18 + TypeScript Single Page Application built with Vite and Tailwind CSS. It implements the full flow for configuring and ordering the **Velô Sprint** electric vehicle:

> Landing → Configurador (`/configure`) → Checkout (`/order`) → Análise de Crédito → Confirmação (`/success`) → Consulta de pedidos (`/lookup`)

Supabase (PostgreSQL + Edge Functions) is used as the backend for persisting orders and running a credit-analysis function.

### Key Directories

- `src/pages/` — Route-level pages (each mapped in `App.tsx`).
- `src/components/` — Reusable UI and feature components:
  - `configurator/` — Car configurator UI (stage + side panel).
  - `landing/` — Marketing/landing sections used on `/`.
  - `ui/` — Shadcn/Radix-based design system components (button, card, toast, tooltip, etc.).
- `src/store/` — Global state managed with Zustand (`configuratorStore.ts`).
- `src/hooks/` — Custom hooks, including toast management and Supabase-backed order helpers.
- `src/integrations/supabase/` — Typed Supabase client and database types.

### Routing & App Shell

- `src/main.tsx` mounts the React app into `#root` and renders `<App />` inside `React.StrictMode`.
- `src/App.tsx` defines the provider and routing stack:
  - Creates a `QueryClient` (`@tanstack/react-query`) and wraps the app in `QueryClientProvider`.
  - Wraps all children with `TooltipProvider`, Radix-based `<Toaster />`, and Sonner `<Toaster />` for notifications.
  - Uses `BrowserRouter` with the following main routes:
    - `/` → `Landing`
    - `/configure` → `Configurator`
    - `/order` → `Order`
    - `/success` → `Success`
    - `/lookup` → `OrderLookup`
    - `/termos` → `Terms`
    - `/privacidade` → `Privacy`
    - `*` → `NotFound` (404 fallback)

Agents should treat `App.tsx` as the entry point for adding or modifying routes and global providers.

### Configurator Domain (Car Configuration)

**State and business logic: `src/store/configuratorStore.ts`**

- Uses `zustand` with `persist` middleware to hold the current car configuration and some user/order metadata in localStorage under `velo-configurator-storage`.
- Core domain types:
  - `CarConfiguration` — `exteriorColor`, `interiorColor`, `wheelType`, `optionals`.
  - `Order` — application-level order representation (id, configuration, totalPrice, customer details, payment method, status, createdAt).
- Key constants and helpers:
  - `BASE_PRICE`, `SPORT_WHEELS_PRICE`, and option pricing via `OPTIONAL_PRICES` and labels via `OPTIONAL_LABELS`.
  - `calculateTotalPrice(config)` — computes final car price from base + wheels + validated optionals.
  - `calculateInstallment(total)` — computes a 12x payment with 2% monthly compound interest.
  - `formatPrice(value)` — formats values as BRL (`pt-BR`).
- Store API (`useConfiguratorStore`):
  - Mutators: `setExteriorColor`, `setInteriorColor`, `setWheelType`, `toggleOptional`, `setViewMode`, `resetConfiguration`.
  - Order-related: `addOrder`, `login`, `logout`, `getUserOrders`.
- Persistence `migrate` function ensures `configuration.optionals` stays a clean string array containing only valid keys from `OPTIONAL_PRICES`.

**Configurator UI: `src/components/configurator/` + `src/pages/Configurator.tsx`**

- `ConfigPanel.tsx` — right-hand configuration panel:
  - Reads and updates `configuration` via `useConfiguratorStore`.
  - Exposes color, wheel, and optional selections using:
    - `ColorSwatch` for colors (maps enum values to labels and hex codes).
    - `WheelOption` for wheel types with associated price.
    - `Checkbox` list for optional features; uses `OPTIONAL_PRICES` and labels from the store.
  - Computes `totalPrice` with `calculateTotalPrice` and displays it in the footer.
  - Navigates to `/order` when the user is ready to place an order.

- `CarStage.tsx` — main visual car stage:
  - Reads `configuration` and `viewMode` from `useConfiguratorStore`.
  - Maps `exteriorColor` + `wheelType` to specific imported images.
  - For interior view, renders a stylized card based on `INTERIOR_COLORS` mapping (no real interior image).

- `src/pages/Configurator.tsx` lays out `CarStage` and `ConfigPanel` in a 70/30 split (responsive).

When modifying the configurator flow or pricing, prefer changing:
- Pricing rules in `configuratorStore.ts`.
- Display and selection logic in `ConfigPanel.tsx` and `CarStage.tsx`.

### Order & Checkout Flow

**Checkout page: `src/pages/Order.tsx`**

- Entry point for converting a configured car into a persisted order.
- Reads `configuration` and `resetConfiguration` from `useConfiguratorStore`.
- Uses Zod schema `orderSchema` to validate personal data (name, surname, email, phone, CPF, store, and terms acceptance).
- Supports two payment modes via local state:
  - `'avista'` — cash payment.
  - `'financiamento'` — financing with dynamic entry (`entryValue`) and simplified financing calculations:
    - `amountToFinance = max(0, totalPrice - entryValue)`
    - `installmentValue = (amountToFinance / 12) * 1.02`
    - `totalFinanced = installmentValue * 12`
- Calls Supabase Edge Function `credit-analysis` for financing:
  - Sends `{ cpf }` and expects a numeric `score`.
  - Decision rules:
    - If entry ≥ 50% and score < 700 → `APROVADO`.
    - Else if score > 700 → `APROVADO`.
    - Else if 501 ≤ score ≤ 700 → `EM_ANALISE`.
    - Else (score ≤ 500) → `REPROVADO`.
- After computing `orderStatus` and `finalPrice`, it:
  - Sanitizes `configuration.optionals` against `OPTIONAL_PRICES`.
  - Calls `createOrder` (from `src/hooks/useOrders.ts`) to write an `orders` row in Supabase.
  - On success:
    - Adds `installmentValue` to the returned `order` object when financing.
    - Copies `store` into `order.customer.store`.
    - Calls `resetConfiguration()` to clear the local configurator state.
    - Navigates to `/success` via `useNavigate`, passing the `order` in `location.state`.
- Error handling:
  - Zod validation errors appear inline for each field.
  - Credit-analysis and DB errors log to `console.error` and show a destructive toast using `useToast()`.

**Order success: `src/pages/Success.tsx`**

- Reads `order` from `useLocation().state` and redirects to `/` via `<Navigate />` if absent.
- Computes `isApproved` based on `order.status === 'APROVADO'` and adjusts visuals accordingly.
- Shows a summary card with:
  - Car image (derived from `order.configuration.exteriorColor` and `wheelType`).
  - Color label, interior label, wheel type, and final price.
  - If financing, shows `12x de {formatPrice(order.installmentValue!)}` in addition to total.
- Displays key identifiers and customer info (order id, customer name/email, store).
- Provides CTA buttons to:
  - Navigate to `/lookup` (consult order).
  - Navigate back to `/configure` (configure another car).

### Order Persistence & Supabase Integration

**Supabase client and types**

- `src/integrations/supabase/types.ts` defines the typed `Database` schema, including the `public.orders` table (color, wheel_type, optionals, customer fields, payment_method, total_price, status, timestamps, etc.).
- `src/integrations/supabase/client.ts` exports a singleton `supabase` client created via `createClient<Database>(SUPABASE_URL, SUPABASE_PUBLISHABLE_KEY)` and configures auth persistence using `localStorage`.

**Order data access: `src/hooks/useOrders.ts`**

- `DbOrder` interface mirrors the `orders` table row structure.
- `generateOrderNumber()` creates IDs like `VLO-XXXXXX` using uppercase letters and digits.
- `dbOrderToOrder(dbOrder)` converts DB rows into the app-level `Order` type by:
  - Splitting `customer_name` into `name` and `surname`.
  - Mapping `color` and `wheel_type` into configuration.
  - Casting `optionals` to `OptionalFeature[]`.
  - Copying `total_price`, `payment_method`, `status`, and `created_at`.
  - Initializing `customer.store` as an empty string (callers can override, as done in `createOrder` and when displaying data).
- `createOrder(orderData)`:
  - Accepts configuration + price + customer + payment info + status.
  - Generates a new `orderNumber`.
  - Inserts a row into `public.orders` with the appropriate fields.
  - On success, converts to `Order`, copies `store` from input, and returns it.
- `getOrderByNumber(orderNumber)`:
  - Normalizes input via `trim().toUpperCase()`.
  - Queries `public.orders` for `order_number` and returns either `null` (no data) or an `Order` built by `dbOrderToOrder`.

Agents modifying DB-related behavior should update both the Supabase SQL or migrations (in `supabase/`) and these mapping helpers to keep types and runtime behavior in sync.

### Order Lookup Flow

**Lookup page: `src/pages/OrderLookup.tsx`**

- Allows users to consult an existing order by its order number.
- Local state tracks:
  - `orderId` (input string),
  - `searchedOrder` (resolved `Order` or null),
  - `notFound` flag,
  - `isLoading` flag.
- On form submit, calls `getOrderByNumber(orderId)` and updates `searchedOrder` / `notFound` accordingly.
- Renders:
  - A search card.
  - A “Pedido não encontrado” card when `notFound`.
  - A rich order detail card with car image, configuration, customer info, timestamps (`createdAt` formatted as `pt-BR`), and payment summary when `searchedOrder` is present.

### Landing, Legal Pages, and 404

- `src/pages/Landing.tsx` composes `Header`, `HeroSection`, `SpecsSection`, `CTASection`, `FAQSection`, and `Footer` from `src/components/landing/`. These are presentational and share the same design system.
- `src/pages/Terms.tsx` and `src/pages/Privacy.tsx` render static legal content in Portuguese, using `Header` and `Footer`. They both display “Última atualização” using `new Date().toLocaleDateString('pt-BR')` at render time.
- `src/pages/NotFound.tsx` logs any 404 route to `console.error` with the attempted pathname and shows a simple “404” page with a link back to `/`.

### Toasts and UI Infrastructure

- `src/hooks/use-toast.ts` implements a global toast manager as a reducer + listener pattern:
  - Maintains a `memoryState` with `toasts: ToasterToast[]` and a `TOAST_LIMIT` of 1.
  - Exposes a `toast()` function that creates a toast with an internal `id`, `open` state, and `onOpenChange` handler that dismisses when closed.
  - Exposes a `useToast()` hook that subscribes to updates and provides `toasts`, `toast`, and `dismiss`.
- `App.tsx` includes `<Toaster />` from `@/components/ui/toaster` and `<Toaster as Sonner />` from `@/components/ui/sonner`, making toast UIs globally available.

Agents adding new user feedback flows should reuse `useToast()` rather than introducing new notification mechanisms.

---

## README Highlights for Agents

Important points from `README.md` that affect how agents should work:

- The app is a SPA focused on the Velô Sprint configurator, credit analysis, and order lookup.
- Tech stack: React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui, Zustand, Zod, TanStack Query, Supabase (DB + Edge Functions).
- Pricing model and credit-analysis thresholds are documented in the README and implemented in `configuratorStore.ts` and `Order.tsx`:
  - Base and optional prices.
  - 12x financing with 2% monthly interest.
  - Score bands for `APROVADO`, `EM_ANALISE`, and `REPROVADO`, with special handling when entry ≥ 50%.
- The `orders` table schema and `order_number` format (`VLO-XXXXXX`) are specified and mirrored by `Database` types and `DbOrder`/`Order` mappings.

Future Warp agents should use this AGENTS file and `README.md` together as the main onboarding sources before editing architecture, flows, or Supabase integrations.
