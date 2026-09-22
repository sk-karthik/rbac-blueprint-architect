# TECHNICAL ARCHITECTURE SPECIFICATION
## Engineering Paradigm: Clean Decoupled Multi-Layer Layered Architecture

## 1. System Integration Mapping

┌────────────────────────────────────────────────────────┐
│                   Frontend Client UI                   │
│   (React 19 + TypeScript + Tailwind CSS Layouts)       │
└───────────────────────────┬────────────────────────────┘
                            │ JSON / REST Core API Contracts
                            ▼
┌────────────────────────────────────────────────────────┐
│             Hono Node.js Core Backend API              │
│  • JWT Session Verification   • RLS Transaction Context│
│  • Credit Reservation Hooks   • Dynamic Menu Compilers │
└───────────────────────────┬────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼ (Exposes Clean Context)   ▼ (Triggers Async Tasks)
┌───────────────────────────────┐ ┌───────────────────────────────┐
│     Model Context Protocol     │ │     GoClaw Processing Agent   │
│         (MCP Server)          │ │      (State-Free Engine)      │
└───────────────────────────────┘ └───────────────────────────────┘

---

## 2. Complete Database Schema (Prisma Blueprint)
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id            String         @id @default(uuid())
  email         String         @unique
  passwordHash  String
  roleId        Int            @default(2)
  role          Role           @relation(fields: [roleId], references: [id])
  creditBalance Int            @default(500)
  createdAt     DateTime       @default(now())
  updatedAt     DateTime       @updatedAt
  ledgers       CreditLedger[]
}

model Role {
  id              Int              @id @default(autoincrement())
  roleName        String           @unique
  isSystemRole    Boolean          @default(false)
  users           User[]
  rolePermissions RolePermission[]
}

model Permission {
  id            Int              @id @default(autoincrement())
  permissionKey String           @unique
  rolePermissions RolePermission[]
}

model RolePermission {
  roleId       Int
  permissionId Int
  role         Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)

  @@id([roleId, permissionId])
}

model MenuItem {
  id                    Int        @id @default(autoincrement())
  title                 String
  route                 String
  icon                  String?
  requiredPermissionKey String?
  parentId              Int?
  displayOrder          Int        @default(0)
}

model CreditLedger {
  id             String       @id @default(uuid())
  userId         String
  user           User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  operationType  String
  creditsCharged Int
  actualCostUsd  Float        @default(0.0)
  status         LedgerStatus @default(RESERVED)
  timestamp      DateTime     @default(now())
}

enum LedgerStatus {
  RESERVED
  SETTLED
  FAILED
}
```

---

## 3. Database Security Hardening (SQL RLS Migration)
To guarantee security separation, the application must completely bypass traditional application trust configurations. Execute these statements directly on your PostgreSQL database node:

```sql
-- 1. Enforce Row Level Security Rules across tables
ALTER TABLE "User" ENABLE ROW LEVEL SECURITY;
ALTER TABLE "CreditLedger" ENABLE ROW LEVEL SECURITY;

-- 2. Build Isolation Assertions based on localized connection state variables
CREATE POLICY tenant_isolation_policy ON "User" 
  AS RESTRICTIVE USING (id = current_setting('app.current_user_id', true));

CREATE POLICY tenant_isolation_policy ON "CreditLedger" 
  AS RESTRICTIVE USING (user_id = current_setting('app.current_user_id', true));
```

### 3.1 Framework Connection Context Binding Middleware
Before any query lifecycle executes inside the API controller, the session verification layer must pass the tracking UUID down to the transaction connection instance:
```typescript
await prisma.$executeRawUnsafe(`SET LOCAL app.current_user_id = '${currentUser.id}';`);
```

---

## 4. Core API Logic Implementations

### 4.1 Recursive Dynamic Menu Matrix Compiler
```typescript
import { Hono } from 'hono';
const app = new Hono();

app.get('/menu', async (c) => {
  const user = c.get('user'); // Injected via JWT guard middleware
  
  // Resolve active keys
  const rolePermissions = await prisma.rolePermission.findMany({
    where: { roleId: user.roleId },
    include: { permission: true }
  });
  const activeKeys = rolePermissions.map(rp => rp.permission.permissionKey);

  // Pull all tracking rows
  const allMenuItems = await prisma.menuItem.findMany({
    orderBy: { displayOrder: 'asc' }
  });

  // Filter items matching permissions array
  const filteredItems = allMenuItems.filter(item => {
    if (!item.requiredPermissionKey) return true;
    return activeKeys.includes(item.requiredPermissionKey);
  });

  // Build recursive children tree
  const buildTree = (parentId: number | null = null) => {
    return filteredItems
      .filter(item => item.parentId === parentId)
      .map(item => ({
        ...item,
        children: buildTree(item.id)
      }));
  };

  return c.json({ menu: buildTree(null) });
});
```

### 4.2 Race-Condition Proof Credit Reservation Transaction Lifecycle
```typescript
app.post('/execute-task', async (c) => {
  const user = c.get('user');
  const MAX_POTENTIAL_COST = 50;

  try {
    const transactionResult = await prisma.$transaction(async (tx) => {
      // 1. Fetch live balance with explicit row locking
      const liveUser = await tx.$queryRaw<{ creditBalance: number }[]>`
        SELECT "creditBalance" FROM "User" WHERE id = ${user.id} FOR UPDATE
      `;
      
      if (!liveUser || liveUser.length === 0 || liveUser.creditBalance < MAX_POTENTIAL_COST) {
        throw new Error('Insufficient Funds');
      }

      // 2. Subtract maximum potential cost immediately
      await tx.user.update({
        where: { id: user.id },
        data: { creditBalance: { decrement: MAX_POTENTIAL_COST } }
      });

      // 3. Register transaction record state as RESERVED
      return await tx.creditLedger.create({
        data: {
          userId: user.id,
          operationType: "AGENT_RUN",
          creditsCharged: MAX_POTENTIAL_COST,
          status: "RESERVED"
        }
      });
    });

    // 4. Hand off execution package safely to the external GoClaw service instance...
    // 5. On webhook callback, calculate actual expenditure cost delta and return unused tokens.

    return c.json({ success: true, ledgerId: transactionResult.id });
  } catch (err) {
    return c.json({ error: err.message }, 400);
  }
});
```
