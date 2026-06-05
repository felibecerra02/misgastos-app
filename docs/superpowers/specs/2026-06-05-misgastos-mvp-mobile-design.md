# MisGastos MVP Mobile V1 - Design Spec

## 1. Product Scope

MisGastos V1 is a mobile personal finance application focused on quick expense and income registration, receipt OCR, recurring expenses, unified history, and a monthly financial dashboard.

The MVP uses:

- React Native + Expo for the mobile app.
- NestJS for the backend API.
- PostgreSQL for persistence.
- JWT authentication with register and login.

Password recovery, AI features, budgets, saving goals, banking integrations, custom categories, tags, receipt storage, background jobs, and advanced automations are out of scope for V1.

## 2. Core Experience

The main navigation is: Inicio | Historial | + | Recurrentes | Perfil.

The Home V1 direction is "Inicio balanceado". It makes projected balance the main signal, keeps quick actions visible, and shows monthly financial context without overloading the first screen.

The central `+` button opens:

- Nuevo gasto
- Escanear comprobante
- Nuevo ingreso

## 3. Architecture

The mobile app focuses on user experience, navigation, forms, validation feedback, loading states, and data visualization.

The backend owns business logic for:

- Users
- Financial movements
- Recurring expenses
- OCR parsing
- Dashboard calculations
- Financial projection

Expenses and incomes are modeled as financial movements with type `EXPENSE` or `INCOME`.

## 4. Domain Model

### User

Stores account and profile data:

- id
- name
- email
- passwordHash
- currency
- createdAt
- updatedAt

Changing `currency` only changes symbol and formatting. It does not convert historical amounts.

### Movement

Represents both expenses and incomes.

Fields:

- id
- userId
- type: `EXPENSE | INCOME`
- origin: `MANUAL | OCR | RECURRENT`
- amount
- date
- description optional
- category optional for incomes, required for expenses
- paymentMethod optional, fixed-list value
- recurringExpenseId optional, only for generated recurring expenses
- ocrConfidence optional
- ocrDetectedMerchant optional
- createdAt
- updatedAt

For V1, incomes do not need a category catalog. They can have `category = null` and rely on description.

### RecurringExpense

Represents a rule that can generate real expense movements.

Fields:

- id
- userId
- name
- category
- amount
- frequency: `WEEKLY | MONTHLY | YEARLY`
- startDate
- endDate optional
- active
- nextExpectedDate
- lastGeneratedDate optional
- createdAt
- updatedAt

When due recurring expenses are generated, the backend must verify that no `Movement` already exists for the same `recurringExpenseId` and expected date.

## 5. Fixed Catalogs

Categories are fixed in V1:

- Supermercado
- Transporte
- Comidas y Bebidas
- Indumentaria
- Servicios Profesionales
- Salud y Cuidado Personal
- Shopping
- Suscripciones
- Entretenimiento
- Retiros de Efectivo
- Otros

Payment method is optional and uses a fixed list:

- Efectivo
- Débito
- Crédito
- Transferencia
- Mercado Pago
- Otro

There are no tags in V1. Description is optional and searchable.

## 6. Main Flows

### Nuevo gasto

The user enters:

- amount
- date, default today
- category, required
- paymentMethod, optional
- description, optional

Saving creates a `Movement` with:

- type = `EXPENSE`
- origin = `MANUAL`

### Nuevo ingreso

The user enters:

- amount
- date, default today
- description, optional

Saving creates a `Movement` with:

- type = `INCOME`
- origin = `MANUAL`
- category = null

### Escanear comprobante

The user selects or captures an image/PDF. The app uploads it to the backend through `POST /ocr/parse-receipt`.

The backend processes OCR synchronously, returns suggestions, and discards the file. OCR never creates a movement directly.

The response includes:

- rawText
- confidence
- detectedFields
- suggestions
- message

Neither file nor `rawText` is persisted automatically.

The mobile app shows a confirmation form prefilled with OCR suggestions. If OCR detection is partial or weak, fields can be empty or partial with a discreet message. The user must confirm or edit before saving.

After confirmation, `POST /movements` saves only the confirmed movement:

- type = `EXPENSE`
- origin = `OCR`
- amount
- date
- category
- description
- paymentMethod
- optional `ocrConfidence`
- optional `ocrDetectedMerchant`

### Recurrentes

The user can create, edit, activate, deactivate, and delete recurring expense rules.

Due recurring expenses are generated automatically when the app opens or when the dashboard is requested. Generated movements are real `EXPENSE` movements with `origin = RECURRENT` and `recurringExpenseId`.

Editing or deleting a generated movement affects only that movement, not the recurring rule.

### Historial

Historial is a unified movement timeline with both expenses and incomes.

Filters:

- type
- date
- category
- paymentMethod
- amount
- free text

## 7. Dashboard And Projection

`GET /dashboard/monthly-summary?month=YYYY-MM` returns a monthly dashboard payload.

Before calculating, the backend syncs due recurring expenses idempotently so Inicio shows updated information.

Response includes:

- month
- currency
- totalIncome
- totalExpense
- pendingRecurringExpense
- projectedBalance
- movementCount
- topExpenseCategory
- expensesByCategory
- monthlyEvolution
- latestMovements
- topCategories

Projection formula:

```text
projectedBalance =
  total income for the month
  - expenses already registered
  - pending recurring expenses not yet generated
```

When a due recurring expense has already been generated as a movement, it is counted as an expense and no longer as pending. This avoids double counting.

## 8. Backend API

Modules:

- AuthModule
- UsersModule
- MovementsModule
- RecurringExpensesModule
- OcrModule
- DashboardModule

Endpoints:

```text
POST /auth/register
POST /auth/login

GET /users/me
PATCH /users/me

GET /movements
POST /movements
GET /movements/:id
PATCH /movements/:id
DELETE /movements/:id

POST /ocr/parse-receipt

GET /recurring-expenses
POST /recurring-expenses
PATCH /recurring-expenses/:id
DELETE /recurring-expenses/:id
POST /recurring-expenses/sync-due

GET /dashboard/monthly-summary?month=YYYY-MM
```

Validations:

- Amount must be positive.
- Date must be valid.
- Category is required for expenses and optional/null for incomes.
- Payment method is optional but must match the fixed list when present.
- Private resources require JWT.
- All reads and writes are scoped by `userId`.

Errors:

- Backend errors use a consistent shape with code, message, and optional details.
- Backend errors must not expose internal data, stack traces, or information from other users.
- OCR partial detection does not block the flow; it returns partial suggestions and a message so the UI can ask for confirmation.

## 9. Frontend Mobile

Feature structure:

- auth
- home
- movements
- ocr
- recurring
- profile
- shared components

State and libraries:

- TanStack Query for remote data, caching, loading states, refetch, and mutations.
- Zustand only for small global local state such as session/preferences/UI.
- React Hook Form + Zod for forms and validation.
- Expo SecureStore for JWT/session persistence.

Main screens:

- Login
- Registro
- Inicio
- Historial
- Crear gasto
- Crear ingreso
- Escanear comprobante / confirmar gasto
- Recurrentes
- Crear / editar recurrente
- Perfil

Required UX states:

- loading
- empty
- recoverable error
- delete confirmation
- OCR processing
- OCR partial result
- dashboard refresh while recurring expenses sync

## 10. Security And Privacy

Security:

- All private endpoints require JWT.
- Every private read/write is scoped by `userId`.
- Passwords are stored as hashes.
- JWT/session data is stored in Expo SecureStore.
- Users cannot access movements, recurring expenses, dashboard data, or profile data from another user.

OCR privacy:

- Uploaded files are used only during `POST /ocr/parse-receipt`.
- Files are not stored.
- `rawText` is not persisted automatically.
- Confirmed movements store only user-confirmed fields and minimal optional OCR metadata.

## 11. Quality And Tests

Backend tests should cover:

- auth register/login
- movement creation and filtering
- expense vs income validation rules
- recurring expense generation and duplicate prevention
- dashboard projection and no double counting
- OCR parser response shape and partial results
- user isolation

Frontend tests should cover:

- critical form validations
- auth/session persistence behavior
- main navigation
- OCR confirmation flow
- recurring expense UI states
- dashboard loading/error/empty states

Important edge cases:

- generated recurring movement is not duplicated
- currency change does not convert historical values
- expenses can omit payment method
- incomes can omit category
- OCR can return partial results
- backend errors do not leak internals
