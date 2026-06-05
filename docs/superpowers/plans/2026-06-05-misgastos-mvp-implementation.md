# MisGastos MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the MisGastos V1 mobile finance app with Expo, NestJS, PostgreSQL, JWT auth, unified financial movements, recurring expenses, backend OCR parsing, and a balanced Inicio dashboard.

**Architecture:** Use a small monorepo with `apps/api` for NestJS, `apps/mobile` for Expo, and `packages/shared` for shared TypeScript constants and schemas. Backend owns business rules and persistence; mobile owns navigation, forms, session storage, and visualization.

**Tech Stack:** React Native + Expo, NestJS, PostgreSQL, Prisma, TypeScript, TanStack Query, Zustand, React Hook Form, Zod, Expo SecureStore, Jest, Supertest.

---

## Scope Split

The approved spec spans several subsystems. Implement it as these working milestones:

1. Workspace, shared domain, and tooling.
2. Backend auth, users, and movement CRUD.
3. Backend recurring expenses and dashboard projection.
4. Backend OCR parse endpoint with no file/rawText persistence.
5. Mobile auth, session, navigation, and profile.
6. Mobile movement flows, OCR confirmation flow, recurring UI, history, and Inicio dashboard.

Each milestone should end with passing tests and a commit.

## Target File Structure

```text
apps/
  api/
    prisma/
      schema.prisma
    src/
      app.module.ts
      main.ts
      common/
        decorators/current-user.decorator.ts
        filters/http-exception.filter.ts
        guards/jwt-auth.guard.ts
      auth/
      users/
      movements/
      recurring-expenses/
      ocr/
      dashboard/
    test/
  mobile/
    app/
      _layout.tsx
      (auth)/
      (tabs)/
    src/
      api/
      components/
      features/
      navigation/
      store/
      theme/
  shared/
    src/
      catalogs.ts
      schemas.ts
      types.ts
package.json
pnpm-workspace.yaml
tsconfig.base.json
```

## Shared Domain Decisions

Use these exact enum values across backend and mobile:

```ts
export const movementTypes = ['EXPENSE', 'INCOME'] as const;
export const movementOrigins = ['MANUAL', 'OCR', 'RECURRENT'] as const;
export const recurringFrequencies = ['WEEKLY', 'MONTHLY', 'YEARLY'] as const;

export const expenseCategories = [
  'Supermercado',
  'Transporte',
  'Comidas y Bebidas',
  'Indumentaria',
  'Servicios Profesionales',
  'Salud y Cuidado Personal',
  'Shopping',
  'Suscripciones',
  'Entretenimiento',
  'Retiros de Efectivo',
  'Otros',
] as const;

export const paymentMethods = [
  'Efectivo',
  'Débito',
  'Crédito',
  'Transferencia',
  'Mercado Pago',
  'Otro',
] as const;
```

---

### Task 1: Scaffold Workspace And Shared Package

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `tsconfig.base.json`
- Create: `packages/shared/package.json`
- Create: `packages/shared/tsconfig.json`
- Create: `packages/shared/src/catalogs.ts`
- Create: `packages/shared/src/types.ts`
- Create: `packages/shared/src/schemas.ts`
- Modify: `.gitignore`

- [ ] **Step 1: Create root workspace files**

Create `package.json`:

```json
{
  "name": "misgastos",
  "private": true,
  "scripts": {
    "test": "pnpm -r test",
    "typecheck": "pnpm -r typecheck",
    "lint": "pnpm -r lint"
  },
  "devDependencies": {
    "typescript": "^5.8.3"
  }
}
```

Create `pnpm-workspace.yaml`:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

Create `tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

- [ ] **Step 2: Create shared package files**

Create `packages/shared/package.json`:

```json
{
  "name": "@misgastos/shared",
  "version": "0.1.0",
  "private": true,
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "node --test dist/*.test.js",
    "lint": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "typescript": "^5.8.3"
  }
}
```

Create `packages/shared/tsconfig.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 3: Add shared catalogs**

Create `packages/shared/src/catalogs.ts` with the enum arrays from "Shared Domain Decisions".

- [ ] **Step 4: Add shared types**

Create `packages/shared/src/types.ts`:

```ts
import type {
  expenseCategories,
  movementOrigins,
  movementTypes,
  paymentMethods,
  recurringFrequencies,
} from './catalogs';

export type MovementType = (typeof movementTypes)[number];
export type MovementOrigin = (typeof movementOrigins)[number];
export type ExpenseCategory = (typeof expenseCategories)[number];
export type PaymentMethod = (typeof paymentMethods)[number];
export type RecurringFrequency = (typeof recurringFrequencies)[number];

export type MonthlySummary = {
  month: string;
  currency: string;
  totalIncome: number;
  totalExpense: number;
  pendingRecurringExpense: number;
  projectedBalance: number;
  movementCount: number;
  topExpenseCategory: ExpenseCategory | null;
  expensesByCategory: Array<{ category: ExpenseCategory; total: number }>;
  monthlyEvolution: Array<{ month: string; totalExpense: number; totalIncome: number }>;
  latestMovements: Array<{
    id: string;
    type: MovementType;
    amount: number;
    date: string;
    description: string | null;
    category: ExpenseCategory | null;
  }>;
  topCategories: Array<{ category: ExpenseCategory; total: number }>;
};
```

- [ ] **Step 5: Add shared schemas**

Create `packages/shared/src/schemas.ts`:

```ts
import { z } from 'zod';
import { expenseCategories, movementOrigins, movementTypes, paymentMethods, recurringFrequencies } from './catalogs';

export const movementSchema = z
  .object({
    type: z.enum(movementTypes),
    origin: z.enum(movementOrigins).default('MANUAL'),
    amount: z.number().positive(),
    date: z.string().datetime(),
    description: z.string().trim().max(300).optional().nullable(),
    category: z.enum(expenseCategories).optional().nullable(),
    paymentMethod: z.enum(paymentMethods).optional().nullable(),
  })
  .superRefine((value, ctx) => {
    if (value.type === 'EXPENSE' && !value.category) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, path: ['category'], message: 'Category is required for expenses.' });
    }
    if (value.type === 'INCOME' && value.category) {
      ctx.addIssue({ code: z.ZodIssueCode.custom, path: ['category'], message: 'Income category must be null in V1.' });
    }
  });

export const recurringExpenseSchema = z.object({
  name: z.string().trim().min(1).max(120),
  category: z.enum(expenseCategories),
  amount: z.number().positive(),
  frequency: z.enum(recurringFrequencies),
  startDate: z.string().datetime(),
  endDate: z.string().datetime().optional().nullable(),
  active: z.boolean().default(true),
});
```

- [ ] **Step 6: Export shared modules**

Create `packages/shared/src/index.ts`:

```ts
export * from './catalogs';
export * from './schemas';
export * from './types';
```

- [ ] **Step 7: Verify shared package**

Run: `pnpm install`

Expected: dependencies install successfully.

Run: `pnpm --filter @misgastos/shared typecheck`

Expected: TypeScript completes with no errors.

- [ ] **Step 8: Commit**

```bash
git add package.json pnpm-workspace.yaml tsconfig.base.json packages/shared .gitignore
git commit -m "chore: scaffold workspace and shared domain"
```

---

### Task 2: Scaffold NestJS API With Prisma

**Files:**
- Create: `apps/api/package.json`
- Create: `apps/api/tsconfig.json`
- Create: `apps/api/jest.config.ts`
- Create: `apps/api/.env.example`
- Create: `apps/api/prisma/schema.prisma`
- Create: `apps/api/src/main.ts`
- Create: `apps/api/src/app.module.ts`
- Create: `apps/api/src/prisma/prisma.service.ts`

- [ ] **Step 1: Create API package**

Create `apps/api/package.json` with scripts:

```json
{
  "name": "@misgastos/api",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "lint": "tsc -p tsconfig.json --noEmit",
    "test": "jest --runInBand",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  },
  "dependencies": {
    "@misgastos/shared": "workspace:*",
    "@nestjs/common": "^11.0.0",
    "@nestjs/config": "^4.0.0",
    "@nestjs/core": "^11.0.0",
    "@nestjs/jwt": "^11.0.0",
    "@nestjs/passport": "^11.0.0",
    "@nestjs/platform-express": "^11.0.0",
    "@prisma/client": "^6.8.0",
    "bcrypt": "^5.1.1",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "multer": "^1.4.5-lts.1",
    "passport": "^0.7.0",
    "passport-jwt": "^4.0.1",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.1",
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "@nestjs/cli": "^11.0.0",
    "@nestjs/testing": "^11.0.0",
    "@types/bcrypt": "^5.0.2",
    "@types/jest": "^29.5.14",
    "@types/multer": "^1.4.12",
    "@types/node": "^22.15.0",
    "@types/passport-jwt": "^4.0.1",
    "jest": "^29.7.0",
    "prisma": "^6.8.0",
    "supertest": "^7.1.0",
    "ts-jest": "^29.3.0",
    "typescript": "^5.8.3"
  }
}
```

- [ ] **Step 2: Add Prisma schema**

Create `apps/api/prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id                 String              @id @default(uuid())
  name               String
  email              String              @unique
  passwordHash       String
  currency           String              @default("ARS")
  movements          Movement[]
  recurringExpenses  RecurringExpense[]
  createdAt          DateTime            @default(now())
  updatedAt          DateTime            @updatedAt
}

model Movement {
  id                  String            @id @default(uuid())
  userId              String
  type                MovementType
  origin              MovementOrigin
  amount              Decimal           @db.Decimal(12, 2)
  date                DateTime
  description         String?
  category            ExpenseCategory?
  paymentMethod       PaymentMethod?
  recurringExpenseId  String?
  ocrConfidence       Float?
  ocrDetectedMerchant String?
  user                User              @relation(fields: [userId], references: [id], onDelete: Cascade)
  recurringExpense    RecurringExpense? @relation(fields: [recurringExpenseId], references: [id], onDelete: SetNull)
  createdAt           DateTime          @default(now())
  updatedAt           DateTime          @updatedAt

  @@index([userId, date])
  @@index([userId, type])
  @@unique([recurringExpenseId, date])
}

model RecurringExpense {
  id                String             @id @default(uuid())
  userId            String
  name              String
  category          ExpenseCategory
  amount            Decimal            @db.Decimal(12, 2)
  frequency         RecurringFrequency
  startDate         DateTime
  endDate           DateTime?
  active            Boolean            @default(true)
  nextExpectedDate  DateTime
  lastGeneratedDate DateTime?
  user              User               @relation(fields: [userId], references: [id], onDelete: Cascade)
  movements         Movement[]
  createdAt         DateTime           @default(now())
  updatedAt         DateTime           @updatedAt

  @@index([userId, active, nextExpectedDate])
}

enum MovementType {
  EXPENSE
  INCOME
}

enum MovementOrigin {
  MANUAL
  OCR
  RECURRENT
}

enum RecurringFrequency {
  WEEKLY
  MONTHLY
  YEARLY
}

enum ExpenseCategory {
  Supermercado
  Transporte
  Comidas_y_Bebidas
  Indumentaria
  Servicios_Profesionales
  Salud_y_Cuidado_Personal
  Shopping
  Suscripciones
  Entretenimiento
  Retiros_de_Efectivo
  Otros
}

enum PaymentMethod {
  Efectivo
  Debito
  Credito
  Transferencia
  Mercado_Pago
  Otro
}
```

Note: Prisma enum values avoid accents/spaces. Map them to UI labels in DTO mappers.

- [ ] **Step 3: Add Nest app shell**

Create `apps/api/src/app.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaService } from './prisma/prisma.service';

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
  providers: [PrismaService],
})
export class AppModule {}
```

Create `apps/api/src/prisma/prisma.service.ts`:

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

Create `apps/api/src/main.ts`:

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
  await app.listen(process.env.PORT ? Number(process.env.PORT) : 3000);
}

bootstrap();
```

- [ ] **Step 4: Verify API shell**

Run: `pnpm install`

Expected: dependencies install.

Run: `pnpm --filter @misgastos/api prisma:generate`

Expected: Prisma client generated.

Run: `pnpm --filter @misgastos/api typecheck`

Expected: no TypeScript errors.

- [ ] **Step 5: Commit**

```bash
git add apps/api package.json pnpm-lock.yaml
git commit -m "chore: scaffold api with prisma"
```

---

### Task 3: Implement Auth And Profile

**Files:**
- Create: `apps/api/src/auth/*`
- Create: `apps/api/src/users/*`
- Create: `apps/api/src/common/guards/jwt-auth.guard.ts`
- Create: `apps/api/src/common/decorators/current-user.decorator.ts`
- Modify: `apps/api/src/app.module.ts`
- Test: `apps/api/src/auth/auth.service.spec.ts`
- Test: `apps/api/src/users/users.controller.spec.ts`

- [ ] **Step 1: Write auth service tests**

Test behaviors:

- register hashes passwords and rejects duplicate email
- login returns JWT for valid credentials
- login rejects invalid password

- [ ] **Step 2: Implement auth module**

Implement:

- `AuthController` with `POST /auth/register` and `POST /auth/login`
- `AuthService` using bcrypt and JwtService
- DTOs with class-validator
- JWT payload `{ sub: user.id, email: user.email }`

- [ ] **Step 3: Implement users profile module**

Implement:

- `GET /users/me`
- `PATCH /users/me`
- allowed profile updates: `name`, `currency`
- no password recovery in V1

- [ ] **Step 4: Verify**

Run: `pnpm --filter @misgastos/api test auth users`

Expected: auth and users tests pass.

Run: `pnpm --filter @misgastos/api typecheck`

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/auth apps/api/src/users apps/api/src/common apps/api/src/app.module.ts
git commit -m "feat: add auth and user profile api"
```

---

### Task 4: Implement Movements API

**Files:**
- Create: `apps/api/src/movements/movements.module.ts`
- Create: `apps/api/src/movements/movements.controller.ts`
- Create: `apps/api/src/movements/movements.service.ts`
- Create: `apps/api/src/movements/dto/*`
- Create: `apps/api/src/movements/movement.mapper.ts`
- Modify: `apps/api/src/app.module.ts`
- Test: `apps/api/src/movements/movements.service.spec.ts`

- [ ] **Step 1: Write movement validation tests**

Test:

- expense requires category
- income must have null category
- amount must be positive
- payment method is optional
- user can only read/write own movements

- [ ] **Step 2: Implement DTOs**

Create DTOs for:

- create movement
- update movement
- list query filters

Use backend enum values and map labels where necessary.

- [ ] **Step 3: Implement service**

Service methods:

- `create(userId, dto)`
- `findAll(userId, filters)`
- `findOne(userId, id)`
- `update(userId, id, dto)`
- `remove(userId, id)`

All methods scope by `userId`.

- [ ] **Step 4: Implement controller**

Routes:

- `GET /movements`
- `POST /movements`
- `GET /movements/:id`
- `PATCH /movements/:id`
- `DELETE /movements/:id`

- [ ] **Step 5: Verify**

Run: `pnpm --filter @misgastos/api test movements`

Expected: movement tests pass.

Run: `pnpm --filter @misgastos/api typecheck`

Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/movements apps/api/src/app.module.ts
git commit -m "feat: add movement api"
```

---

### Task 5: Implement Recurring Expenses

**Files:**
- Create: `apps/api/src/recurring-expenses/*`
- Modify: `apps/api/src/app.module.ts`
- Test: `apps/api/src/recurring-expenses/recurring-expenses.service.spec.ts`

- [ ] **Step 1: Write recurring sync tests**

Test:

- creates due movement with `origin = RECURRENT`
- sets `recurringExpenseId`
- updates `lastGeneratedDate`
- advances `nextExpectedDate`
- does not duplicate if a movement already exists for same recurring expense and expected date
- respects inactive rules

- [ ] **Step 2: Implement recurring CRUD**

Routes:

- `GET /recurring-expenses`
- `POST /recurring-expenses`
- `PATCH /recurring-expenses/:id`
- `DELETE /recurring-expenses/:id`

- [ ] **Step 3: Implement sync**

Route:

- `POST /recurring-expenses/sync-due`

The sync method loops active rules due on or before today, creates missing movements, updates `lastGeneratedDate`, and advances `nextExpectedDate` by frequency.

- [ ] **Step 4: Verify**

Run: `pnpm --filter @misgastos/api test recurring-expenses`

Expected: recurring tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/recurring-expenses apps/api/src/app.module.ts
git commit -m "feat: add recurring expense sync"
```

---

### Task 6: Implement Dashboard Summary

**Files:**
- Create: `apps/api/src/dashboard/*`
- Modify: `apps/api/src/app.module.ts`
- Test: `apps/api/src/dashboard/dashboard.service.spec.ts`

- [ ] **Step 1: Write dashboard projection tests**

Test:

- returns `month` and user `currency`
- calculates `totalIncome`
- calculates `totalExpense`
- calculates `pendingRecurringExpense`
- calculates `projectedBalance`
- generated recurring expenses are not double counted
- dashboard calls recurring sync before calculating

- [ ] **Step 2: Implement dashboard service**

Implement `getMonthlySummary(userId, month)`:

1. Sync due recurring expenses.
2. Query movements for month.
3. Query active recurring rules still pending in month.
4. Build totals and chart payloads.

- [ ] **Step 3: Implement dashboard controller**

Route:

- `GET /dashboard/monthly-summary?month=YYYY-MM`

- [ ] **Step 4: Verify**

Run: `pnpm --filter @misgastos/api test dashboard`

Expected: dashboard tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/dashboard apps/api/src/app.module.ts
git commit -m "feat: add monthly dashboard summary"
```

---

### Task 7: Implement OCR Parse Endpoint

**Files:**
- Create: `apps/api/src/ocr/*`
- Modify: `apps/api/src/app.module.ts`
- Test: `apps/api/src/ocr/ocr.service.spec.ts`

- [ ] **Step 1: Write OCR parser tests**

Test:

- parser returns `rawText`, `confidence`, `detectedFields`, `suggestions`, and `message`
- parser can return partial fields
- endpoint does not create a movement
- endpoint does not persist file or rawText

- [ ] **Step 2: Implement OCR service boundary**

Create an `OcrService` with:

```ts
type OcrParseResult = {
  rawText: string;
  confidence: number;
  detectedFields: {
    merchant?: string;
    date?: string;
    total?: number;
  };
  suggestions: {
    description?: string;
    date?: string;
    amount?: number;
    category?: string;
  };
  message: string;
};
```

Use `tesseract.js` behind an `OcrTextExtractor` interface. Unit-test field parsing with fixed text fixtures, and integration-test the controller with a mocked extractor so the no-persistence behavior is deterministic.

- [ ] **Step 3: Implement upload endpoint**

Route:

- `POST /ocr/parse-receipt`

Use in-memory file handling. Do not write the file to disk. Return the parse result.

- [ ] **Step 4: Verify**

Run: `pnpm --filter @misgastos/api test ocr`

Expected: OCR tests pass.

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/ocr apps/api/src/app.module.ts
git commit -m "feat: add receipt ocr parse endpoint"
```

---

### Task 8: Scaffold Expo Mobile App

**Files:**
- Create: `apps/mobile/package.json`
- Create: `apps/mobile/app.json`
- Create: `apps/mobile/tsconfig.json`
- Create: `apps/mobile/app/_layout.tsx`
- Create: `apps/mobile/src/api/client.ts`
- Create: `apps/mobile/src/store/session.store.ts`

- [ ] **Step 1: Create Expo package**

Use Expo Router, TanStack Query, Zustand, React Hook Form, Zod, and SecureStore.

- [ ] **Step 2: Implement API client**

Create an API client that:

- reads JWT from session state
- sends `Authorization: Bearer <token>`
- handles JSON errors without exposing internals in UI

- [ ] **Step 3: Implement session store**

Persist JWT in Expo SecureStore. Do not use AsyncStorage for JWT/session.

- [ ] **Step 4: Verify**

Run: `pnpm --filter @misgastos/mobile typecheck`

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add apps/mobile package.json pnpm-lock.yaml
git commit -m "chore: scaffold mobile app"
```

---

### Task 9: Mobile Auth, Tabs, And Profile

**Files:**
- Create: `apps/mobile/app/(auth)/login.tsx`
- Create: `apps/mobile/app/(auth)/register.tsx`
- Create: `apps/mobile/app/(tabs)/_layout.tsx`
- Create: `apps/mobile/app/(tabs)/profile.tsx`
- Create: `apps/mobile/src/features/auth/*`
- Create: `apps/mobile/src/features/profile/*`

- [ ] **Step 1: Implement auth screens**

Login and register call the backend and store JWT in SecureStore.

- [ ] **Step 2: Implement tab navigation**

Tabs must be:

```text
Inicio | Historial | + | Recurrentes | Perfil
```

The `+` tab opens an action sheet with:

- Nuevo gasto
- Escanear comprobante
- Nuevo ingreso

- [ ] **Step 3: Implement profile screen**

Profile calls:

- `GET /users/me`
- `PATCH /users/me`

Editable fields:

- name
- currency

- [ ] **Step 4: Verify**

Run: `pnpm --filter @misgastos/mobile typecheck`

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add apps/mobile/app apps/mobile/src/features/auth apps/mobile/src/features/profile
git commit -m "feat: add mobile auth navigation and profile"
```

---

### Task 10: Mobile Movements, OCR Confirmation, History, Recurring, And Inicio

**Files:**
- Create: `apps/mobile/src/features/movements/*`
- Create: `apps/mobile/src/features/ocr/*`
- Create: `apps/mobile/src/features/recurring/*`
- Create: `apps/mobile/src/features/home/*`
- Create: `apps/mobile/app/(tabs)/index.tsx`
- Create: `apps/mobile/app/(tabs)/history.tsx`
- Create: `apps/mobile/app/(tabs)/recurring.tsx`

- [ ] **Step 1: Implement movement forms**

Forms:

- Nuevo gasto
- Nuevo ingreso

Use React Hook Form + Zod. Enforce:

- expense category required
- income category null
- payment method optional
- positive amount

- [ ] **Step 2: Implement OCR confirmation flow**

Flow:

1. Select image/PDF.
2. Call `POST /ocr/parse-receipt`.
3. Show confirmation form with partial suggestions.
4. User confirms.
5. Call `POST /movements` with `origin = OCR`.

Never create a movement directly from OCR parse.

- [ ] **Step 3: Implement unified history**

History lists both `EXPENSE` and `INCOME` movements and supports filters:

- type
- date
- category
- payment method
- amount
- free text

- [ ] **Step 4: Implement recurring screens**

Screens:

- recurring list
- create/edit recurring expense
- activate/deactivate
- delete

- [ ] **Step 5: Implement Inicio**

Inicio calls:

- `GET /dashboard/monthly-summary?month=YYYY-MM`

Show:

- projected balance
- total income
- total expense
- pending recurring expense
- expenses by category
- monthly evolution
- latest movements
- top categories

Include loading, empty, recoverable error, and refresh states.

- [ ] **Step 6: Verify**

Run: `pnpm --filter @misgastos/mobile typecheck`

Expected: no errors.

Run: `pnpm --filter @misgastos/mobile test`

Expected: mobile tests pass.

- [ ] **Step 7: Commit**

```bash
git add apps/mobile/app apps/mobile/src/features
git commit -m "feat: add mobile finance flows"
```

---

### Task 11: End-To-End Verification

**Files:**
- Modify: `README.md`
- Create: `apps/api/.env.example`
- Create: `apps/mobile/.env.example`

- [ ] **Step 1: Document local setup**

Create `README.md` with:

- dependency installation
- PostgreSQL setup
- Prisma migration
- API dev server command
- mobile dev server command
- test commands

- [ ] **Step 2: Run full verification**

Run:

```bash
pnpm typecheck
pnpm test
```

Expected: all workspace typechecks and tests pass.

- [ ] **Step 3: Manual smoke test**

Start backend and mobile. Verify:

- register
- login
- profile currency update
- create expense
- create income
- OCR parse returns suggestions and requires confirmation
- recurring sync creates a movement once
- Inicio dashboard reflects totals
- Historial shows unified timeline

- [ ] **Step 4: Commit**

```bash
git add README.md apps/api/.env.example apps/mobile/.env.example
git commit -m "docs: add local setup and verification guide"
```
