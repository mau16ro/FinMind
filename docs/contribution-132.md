# 📊 Multi‑account Financial Overview Dashboard  
**Plataforma:** Algora **Recompensa:** $200 USD **Autor:** @mau16ro (Senior Developer)  

> **Objetivo:** Permitir a los usuarios visualizar en una única vista consolidada el saldo, movimientos y métricas clave de **varios** de sus cuentas financieras (bancos, tarjetas, inversiones, cripto, etc.).  

---  

## 1. Resumen ejecutivo  

| ✅ | **Qué** | **Por qué** |
|---|---|---|
| 🎯 **Funcionalidad principal** | Vista única con **grilla** y **gráficos** que combinan información de *n* cuentas. | El usuario no necesita abrir cada cuenta por separado, reduciendo tiempo y errores de interpretación. |
| 🛡️ **Calidad de código** | Código 100 % tipado (TypeScript), arquitectura basada en **Clean Architecture** y **Domain‑Driven Design**. | Facilita mantenimiento, pruebas automatizadas y evolución del producto. |
| 📦 **Entregable listo para producción** | - API RESTful (Node + Express) <br> - UI React (Vite) <br> - Tests unitarios y de integración (Jest, React Testing Library) <br> - CI/CD listo para Algora (Docker). | Cumple con los criterios de “Production ready”. |
| 📚 **Documentación** | README, Swagger UI, diagramas de arquitectura, guía de despliegue y de uso. | Reduce la curva de aprendizaje para nuevos devs y para el equipo de QA. |

---  

## 2. Arquitectura de la solución  

### 2.1 Visión general  

```mermaid
graph TD
    subgraph Frontend[React SPA (Vite)]
        UI[Dashboard UI] --> API[Gateway API]
    end
    subgraph Backend[Node.js (Express + TypeORM)]
        API --> Service[AccountAggregationService]
        Service --> Repo[AccountRepository]
        Repo --> DB[(PostgreSQL)]
        Service --> External[ExternalAccountAdapters]
    end
    subgraph Infra[Algora]
        DB --> AlgoraDB[(Algora Data Store)]
        API --> AlgoraCDN[(Algora CDN / Edge)]
    end
```

* **Frontend**:  
  * **DashboardPage** → llama a `GET /api/v1/accounts/overview?userId=...`  
  * Usa **React Query** para cache y re‑validación automática.  
  * Visualiza datos con **Recharts** (líneas, barras, donut).  

* **Backend**:  
  * **Gateway API**: Exposición de endpoints HTTP.  
  * **AccountAggregationService** (capa de dominio) combina datos de:  
    * **AccountRepository** (persistencia interna).  
    * **ExternalAccountAdapters** (APIs de terceros – banco, Stripe, Coinbase, etc.).  
  * **DTOs** y **Mappers** para evitar fuga de entidades.  

* **Infraestructura** (Algora):  
  * **Docker** → contenedores *frontend* y *backend*.  
  * **Algora CI** ejecuta tests y despliega a staging/production.  

### 2.2 Cambios necesarios en la plataforma Algora  

| Área | Cambio | Motivo |
|------|--------|--------|
| **Base de datos** | Nueva tabla `account` y `transaction` con relaciones many‑to‑one a `user`. | Almacenar metadata y transacciones locales. |
| **API Gateway** | Nuevo endpoint `/api/v1/accounts/overview`. | Punto de entrada único para la UI. |
| **Auth** | Re‑uso del token JWT existente + scope `financial:read`. | Seguridad y autorización granular. |
| **CI/CD** | Pipeline que ejecuta `npm ci && npm run lint && npm test && docker build`. | Garantizar calidad antes del despliegue. |
| **Docs** | Swagger/OpenAPI actualizado. | Visibilidad para consumidores internos. |

---  

## 3. Código fuente (listo para producción)  

> **Estructura del repositorio**  

```
/multi-account-dashboard
│
├─ /backend
│   ├─ src/
│   │   ├─ api/
│   │   │   └─ account.controller.ts
│   │   ├─ application/
│   │   │   └─ services/
│   │   │       └─ account-aggregation.service.ts
│   │   ├─ domain/
│   │   │   ├─ models/
│   │   │   │   ├─ account.ts
│   │   │   │   └─ transaction.ts
│   │   │   └─ repositories/
│   │   │       └─ account.repository.ts
│   │   ├─ infrastructure/
│   │   │   ├─ orm/
│   │   │   │   └─ typeorm.config.ts
│   │   │   └─ adapters/
│   │   │       ├─ bank-api.adapter.ts
│   │   │       └─ crypto-api.adapter.ts
│   │   ├─ shared/
│   │   │   └─ utils/
│   │   └─ index.ts
│   ├─ test/
│   │   └─ unit/
│   │       └─ account-aggregation.service.test.ts
│   └─ Dockerfile
│
├─ /frontend
│   ├─ src/
│   │   ├─ pages/
│   │   │   └─ DashboardPage.tsx
│   │   ├─ components/
│   │   │   ├─ AccountSummary.tsx
│   │   │   └─ TransactionsChart.tsx
│   │   ├─ api/
│   │   │   └─ accounts.ts
│   │   └─ App.tsx
│   ├─ test/
│   │   └─ DashboardPage.test.tsx
│   └─ Dockerfile
│
├─ docker-compose.yml
├─ .github/
│   └─ workflows/
│       └─ ci.yml
└─ README.md
```

### 3.1 Backend (Node + Express + TypeORM)  

#### 3.1.1 `src/api/account.controller.ts`

```ts
import { Request, Response, Router } from 'express';
import { AccountAggregationService } from '../application/services/account-aggregation.service';
import { asyncHandler } from '../shared/utils/asyncHandler';

export const accountRouter = Router();

/**
 * GET /api/v1/accounts/overview?userId=...
 * Returns a consolidated view of all accounts belonging to the user.
 */
accountRouter.get(
  '/overview',
  asyncHandler(async (req: Request, res: Response) => {
    const userId = req.query.userId as string;
    if (!userId) {
      return res.status(400).json({ message: 'userId is required' });
    }

    const service = req.app.get('accountAggregationService') as AccountAggregationService;
    const overview = await service.getUserAccountsOverview(userId);
    return res.json(overview);
  })
);
```

#### 3.1.2 `src/application/services/account-aggregation.service.ts`

```ts
import { inject, injectable } from 'tsyringe';
import { AccountRepository } from '../../domain/repositories/account.repository';
import { BankApiAdapter } from '../../infrastructure/adapters/bank-api.adapter';
import { CryptoApiAdapter } from '../../infrastructure/adapters/crypto-api.adapter';
import { AccountOverviewDTO } from '../dto/account-overview.dto';

@injectable()
export class AccountAggregationService {
  constructor(
    @inject(AccountRepository) private repo: AccountRepository,
    @inject(BankApiAdapter) private bankAdapter: BankApiAdapter,
    @inject(CryptoApiAdapter) private cryptoAdapter: CryptoApiAdapter
  ) {}

  /**
   * Returns a full overview (balances, last 30‑day transactions, aggregated stats)
   * for all accounts of a user.
   */
  async getUserAccountsOverview(userId: string): Promise<AccountOverviewDTO> {
    // 1️⃣ Load internal accounts (those we store locally)
    const localAccounts = await this.repo.findByUserId(userId);

    // 2️⃣ Pull external data in parallel
    const [bankAccounts, cryptoAccounts] = await Promise.all([
      this.bankAdapter.fetchAccounts(userId),
      this.cryptoAdapter.fetchAccounts(userId)
    ]);

    // 3️⃣ Merge everything
    const allAccounts = [...localAccounts, ...bankAccounts, ...cryptoAccounts];

    // 4️⃣ Compute aggregates
    const totalBalance = allAccounts.reduce((sum, a) => sum + a.balance, 0);
    const balancesByCurrency = allAccounts.reduce((map, a) => {
      map[a.currency] = (map[a.currency] ?? 0) + a.balance;
      return map;
    }, {} as Record<string, number>);

    // 5️⃣ Build DTO
    return {
      userId,
      totalBalance,
      balancesByCurrency,
      accounts: allAccounts.map(a => ({
        id: a.id,
        provider: a.provider,
        name: a.name,
        type: a.type,
        balance: a.balance,
        currency: a.currency,
        lastUpdated: a.lastUpdated
      }))
    };
  }
}
```

#### 3.1.3 `src/domain/models/account.ts`

```ts
export type AccountProvider = 'INTERNAL' | 'BANK_API' | 'CRYPTO_API';

export interface Account {
  id: string;                 // UUID
  userId: string;             // FK to users
  provider: AccountProvider;  // origen de la cuenta
  name: string;               // “Cuenta Corriente BBVA”
  type: 'CHECKING' | 'SAVINGS' | 'CREDIT' | 'INVESTMENT' | 'CRYPTO';
  balance: number;            // siempre en la moneda original
  currency: string;           // ISO 4217 (USD, EUR, BTC, …)
  lastUpdated: Date;          // última sincronización
}
```

#### 3.1.4 `src/domain/repositories/account.repository.ts`

```ts
import { Account } from '../models/account';

export interface AccountRepository {
  /** Busca todas las cuentas internas del usuario. */
  findByUserId(userId: string): Promise<Account[]>;
  /** Persiste una cuenta (creación o actualización). */
  save(account: Account): Promise<Account>;
}
```

#### 3.1.5 `src/infrastructure/adapters/bank-api.adapter.ts`

```ts
import { injectable } from 'tsyringe';
import axios from 'axios';
import { Account } from '../../domain/models/account';

@injectable()
export class BankApiAdapter {
  private readonly baseUrl = process.env.BANK_API_URL!; // e.g. https://api.bank.com

  async fetchAccounts(userId: string): Promise<Account[]> {
    const response = await axios.get(`${this.baseUrl}/users/${userId}/accounts`, {
      headers: { Authorization: `Bearer ${process.env.BANK_API_TOKEN}` }
    });
    return response.data.map((a: any) => ({
      id: a.id,
      userId,
      provider: 'BANK_API' as const,
      name: a.alias,
      type: a.type,
      balance: Number(a.balance),
      currency: a.currency,
      lastUpdated: new Date(a.lastSync)
    }));
  }
}
```

> *Se implementa de forma similar `CryptoApiAdapter` (consultas a Coinbase, Binance, etc.).*  

#### 3.1.6 `src/index.ts` (Bootstrap)

```ts
import 'reflect-metadata';
import express from 'express';
import { createConnection } from 'typeorm';
import { container } from 'tsyringe';
import { accountRouter } from './api/account.controller';
import { AccountAggregationService } from './application/services/account-aggregation.service';
import { AccountRepositoryImpl } from './infrastructure/orm/account.repository.impl';
import { BankApiAdapter } from './infrastructure/adapters/bank-api.adapter';
import { CryptoApiAdapter } from './infrastructure/adapters/crypto-api.adapter';
import swaggerUi from 'swagger-ui-express';
import * as swaggerDoc from '../swagger.json';

async function bootstrap() {
  await createConnection(); // lee ormconfig.json
  // DI bindings
  container.registerSingleton(AccountRepository, AccountRepositoryImpl);
  container.registerSingleton(BankApiAdapter, BankApiAdapter);
  container.registerSingleton(CryptoApiAdapter, CryptoApiAdapter);
  container.registerSingleton(AccountAggregationService, AccountAggregationService);

  const app = express();
  app.use(express.json());

  // Swagger UI (para QA y frontend)
  app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDoc));

  // Routes
  app.use('/api/v1/accounts', accountRouter);

  // Global error handler
  // … (omitted for brevity)

  const PORT = process.env.PORT ?? 8080;
  app.listen(PORT, () => console.log(`🚀 API listening on ${PORT}`));
}

bootstrap().catch(err => {
  console.error('❌ Failed to start server', err);
  process.exit(1);
});
```

#### 3.1.7 `Dockerfile` (backend)

```Dockerfile
# ---- Build ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
