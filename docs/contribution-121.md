# 📦 Smart Digest – Weekly Financial Summary  
**Proyecto:** `algora-smart-digest`  
**Cliente:** @mau16ro  
**Recompensa:** $500 USD  

---  

## 1️⃣ Resumen Ejecutivo y Objetivos Técnicos  

| **Objetivo** | **Descripción** |
|--------------|-----------------|
| **Resumen semanal automático** | Generar un “digest” con los principales indicadores financieros (ingresos, gastos, margen, variaciones vs. semana anterior, tendencias) y una breve narración de insights. |
| **Entrega en tiempo real** | El proceso debe ejecutarse automáticamente cada lunes a las 06:00 UTC y publicar el resultado en la API de Algora (o enviarlo por email/Slack). |
| **Escalabilidad y mantenibilidad** | Código modular, tipado estático, pruebas automatizadas y CI‑CD listo para producción. |
| **Facilidad de despliegue** | Contenedorizado (Docker) y configurable mediante variables de entorno. |
| **Documentación completa** | Guía de instalación, arquitectura y referencia de la API. |

> **Resultado esperado:** Un micro‑servicio **RESTful** (`/digest/weekly`) que, al ser invocado (o mediante cron interno), devuelve un objeto JSON con el resumen financiero y una cadena de texto “human‑readable”.  

---  

## 2️⃣ Arquitectura de la Solución  

### 2.1 Diagrama de componentes  

```
+-------------------+          +---------------------+          +-------------------+
|   Algora Platform | <------> |  Smart Digest API   | <------> |  Data Sources     |
| (Auth, Scheduler) |  HTTP    |  (Node.js/TS)       |  SQL/REST|  (Postgres,      |
|                   |          |                     |          |   External APIs) |
+-------------------+          +---------------------+          +-------------------+

                ^                                 ^
                |                                 |
                | Docker (K8s)                     | Cron / Scheduler (Algora)
                |                                 |
                +---------------------------------+
```

### 2.2 Tecnologías  

| **Capa** | **Tecnología** | **Motivo** |
|----------|----------------|-----------|
| **Runtime** | **Node.js 20 LTS** + **TypeScript 5** | Tipado estático, ecosistema amplio, fácil despliegue en Algora. |
| **Framework** | **Fastify** (ligero, alto rendimiento) | Enfoque orientado a micro‑servicios, plugins de validación y schema. |
| **ORM** | **Prisma** | Genera tipos a partir del esquema DB, migraciones y consultas tipadas. |
| **Testing** | **Jest** + **SuperTest** | Cobertura total de lógica y endpoints. |
| **CI/CD** | **GitHub Actions** (lint, test, Docker build) | Automatiza calidad antes de merge. |
| **Contenedorización** | **Docker** + **Docker‑Compose** | Entorno reproducible, ready for K8s. |
| **Scheduler** | **Algora Scheduler** (cron) o **node‑cron** (fallback) | Ejecutar la generación semanal sin intervención. |
| **Logging** | **pino** (Fastify logger) | Registro estructurado, JSON, fácil integración con ELK. |
| **Documentación** | **Swagger (OpenAPI 3.0)** via **fastify‑swagger** | Autogeneración de spec para consumidores. |

### 2.3 Cambios necesarios en Algora  

1. **Crear un endpoint público**: `GET /digest/weekly` (requiere token de API).  
2. **Registrar una tarea cron** en Algora que invoque `POST /digest/weekly/generate`.  
3. **Permisos**: otorgar al servicio acceso de solo‑lectura a la base de datos `financial_transactions`.  

---  

## 3️⃣ Código Fuente – Implementación Lista para Producción  

> **Estructura del proyecto**  

```
algora-smart-digest/
│
├─ src/
│   ├─ index.ts                # bootstrap Fastify
│   ├─ server.ts               # configuración del servidor
│   ├─ routes/
│   │   └─ digest.routes.ts    # endpoints
│   ├─ services/
│   │   └─ digest.service.ts   # lógica de generación
│   ├─ models/
│   │   └─ digest.dto.ts       # tipos de salida
│   ├─ utils/
│   │   ├─ date.utils.ts
│   │   └─ insight.utils.ts
│   └─ prisma/
│       └─ client.ts           # Prisma client singleton
│
├─ prisma/
│   └─ schema.prisma           # modelo DB
│
├─ tests/
│   ├─ digest.service.test.ts
│   └─ digest.routes.test.ts
│
├─ .env.example
├─ Dockerfile
├─ docker-compose.yml
├─ jest.config.ts
├─ tsconfig.json
├─ package.json
└─ README.md
```

### 3.1 `package.json`

```json
{
  "name": "algora-smart-digest",
  "version": "1.0.0",
  "description": "Micro‑service that generates weekly financial digest for Algora platform",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "start": "node dist/index.js",
    "lint": "eslint . --ext .ts",
    "test": "jest --coverage",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate deploy"
  },
  "author": "Team @mau16ro",
  "license": "MIT",
  "dependencies": {
    "@fastify/swagger": "^8.12.0",
    "fastify": "^4.27.0",
    "fastify-plugin": "^4.7.0",
    "fastify-sensible": "^5.5.0",
    "prisma": "^5.10.0",
    "pino": "^9.1.0"
  },
  "devDependencies": {
    "@types/jest": "^29.5.0",
    "@types/node": "^20.11.0",
    "eslint": "^8.56.0",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-import": "^2.29.0",
    "jest": "^29.7.0",
    "supertest": "^6.3.3",
    "ts-jest": "^29.1.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.3.3"
  }
}
```

### 3.2 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "sourceMap": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true
  },
  "include": ["src/**/*.ts", "tests/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### 3.3 Prisma schema (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model FinancialTransaction {
  id          Int      @id @default(autoincrement())
  amount      Decimal  @db.Numeric(12,2)
  type        TransactionType
  category    String
  createdAt   DateTime @default(now())
  description String?
}

enum TransactionType {
  INCOME
  EXPENSE
}
```

### 3.4 Prisma client (`src/prisma/client.ts`)

```ts
import { PrismaClient } from '@prisma/client';
import pino from 'pino';

const logger = pino({ name: 'prisma-client' });

export const prisma = new PrismaClient({
  log: [
    { emit: 'event', level: 'query' },
    { emit: 'event', level: 'error' },
  ],
});

prisma.$on('query', (e) => {
  logger.debug({ query: e.query, params: e.params, duration: e.duration });
});

prisma.$on('error', (e) => {
  logger.error(e);
});
```

### 3.5 Utilidades  

**`src/utils/date.utils.ts`**

```ts
import { startOfWeek, endOfWeek, subWeeks } from 'date-fns';

/**
 * Returns the start (Monday 00:00) and end (Sunday 23:59:59.999) of a given ISO week.
 */
export function getWeekRange(date: Date = new Date()): { start: Date; end: Date } {
  const start = startOfWeek(date, { weekStartsOn: 1 });
  const end = endOfWeek(date, { weekStartsOn: 1 });
  return { start, end };
}

/**
 * Returns the same week of the previous year (used for YoY comparison)
 */
export function getPreviousWeekRange(date: Date = new Date()) {
  const previous = subWeeks(date, 1);
  return getWeekRange(previous);
}
```

**`src/utils/insight.utils.ts`**

```ts
/**
 * Generate a human‑readable insight based on delta percentage.
 */
export function generateInsight(
  metric: string,
  current: number,
  previous: number
): string {
  const diff = current - previous;
  const pct = previous === 0 ? 0 : (diff / previous) * 100;

  if (Math.abs(pct) < 2) return `${metric} se mantiene estable respecto a la semana anterior.`;

  const direction = pct > 0 ? 'aumento' : 'disminución';
  const magnitude = Math.abs(pct).toFixed(1);
  return `${metric} muestra un ${direction} del ${magnitude}% comparado con la semana anterior.`;
}
```

### 3.6 DTO – Salida del Digest (`src/models/digest.dto.ts`)

```ts
export interface WeeklyDigest {
  weekStart: string; // ISO date
  weekEnd: string;   // ISO date
  totals: {
    income: number;
    expense: number;
    net: number;
    marginPct: number; // (income - expense) / income * 100
  };
  comparison: {
    incomeDeltaPct: number;
    expenseDeltaPct: number;
    netDeltaPct: number;
  };
  insights: string[]; // human readable sentences
}
```

### 3.7 Servicio de generación (`src/services/digest.service.ts`)

```ts
import { prisma } from '../prisma/client';
import { getWeekRange, getPreviousWeekRange } from '../utils/date.utils';
import { generateInsight } from '../utils/insight.utils';
import { WeeklyDigest } from '../models/digest.dto';
import { Decimal } from '@prisma/client/runtime';

export class DigestService {
  /**
   * Main entry point – calculates the weekly digest.
   */
  static async generate(date: Date = new Date()): Promise<WeeklyDigest> {
    const { start, end } = getWeekRange(date);
    const { start: prevStart, end: prevEnd } = getPreviousWeekRange(date);

    // 1️⃣  Load current week data
    const current = await this.aggregatePeriod(start, end);
    // 2️⃣  Load previous week data
    const previous = await this.aggregatePeriod(prevStart, prevEnd);

    // 3️⃣  Compute deltas
    const comparison = {
      incomeDeltaPct: this.deltaPct(current.income, previous.income),
      expenseDeltaPct: this.deltaPct(current.expense, previous.expense),
      netDeltaPct: this.deltaPct(current.net, previous.net),
    };

    // 4️⃣  Build insights
    const insights = [
      generateInsight('Ingresos', current.income, previous.income),
      generateInsight('Gastos', current.expense, previous.expense),
      generateInsight('Resultado neto', current.net, previous.net),
    ];

    // 5️⃣  Return DTO
    return {
      weekStart: start.toISOString(),
      weekEnd: end.toISOString(),
      totals: {
        income: Number(current.income.toFixed(2)),
        expense: Number(current.expense.toFixed(2)),
        net: Number(current.net.toFixed(2)),
        marginPct: Number(((current.net / current.income) * 100).toFixed(2)),
      },
      comparison,
      insights,
    };
  }

  /** Helper: aggregate income/expense for a period */
  private static async aggregatePeriod(
    start: Date,
    end: Date
  ): Promise<{ income: Decimal; expense: Decimal; net: Decimal }> {
    const result = await prisma.financialTransaction.groupBy({
      by: ['type'],
      where: {
        createdAt: {
          gte: start,
          lte: end,
        },
      },
      _sum: {
        amount: true,
      },
    });

    const income = result.find((r) => r.type === 'INCOME')?._sum.amount ?? new Decimal