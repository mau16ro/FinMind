# 📦 Goal‑Based Savings Tracking & Milestones  
**Proyecto:** Algora – Reward $250 USD  
**Desarrollador:** @mau16ro (Senior Engineer)  

---  

## 1️⃣ Resumen Ejecutivo  

| **Objetivo** | Permitir a los usuarios crear, seguir y completar metas de ahorro, con hitos (milestones) que se marcan automáticamente al alcanzar el objetivo parcial. |
|--------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| **Valor de negocio** | Incrementa la retención y el engagement de la plataforma Algora al ofrecer una funcionalidad de finanzas personales que genera hábitos de ahorro. |
| **Entregables** | - API RESTful **production ready**. <br> - Persistencia en PostgreSQL usando Prisma. <br> - Validación, logging y manejo de errores centralizado. <br> - Tests unitarios y de integración con **Jest** + **SuperTest**. <br> - Documentación Swagger/OpenAPI y README de instalación. |
| **Tecnologías** | Node.js 16+, TypeScript, Express, Prisma, PostgreSQL, Jest, SuperTest, Swagger UI, Docker (opcional). |
| **Criterios de aceptación** | - Cumple con la especificación de endpoints. <br> - Cobertura de tests ≥ 80 %. <br> - Código tipado y con comentarios claros. <br> - README e‑docs actualizados. |

---  

## 2️⃣ Arquitectura y Cambios Necesarios  

### 2.1 Diagrama de alto nivel  

```
+-------------------+        +-------------------+        +-------------------+
|   Cliente (SPA)   | <--->  |  API Gateway (NGINX) | <---> |  Savings Service  |
+-------------------+        +-------------------+        +-------------------+
                                               |
                                           PostgreSQL
```

- **Savings Service**: micro‑servicio que expone los endpoints para metas y hitos.  
- **API Gateway**: (ya existente en Algora) enruta `/api/v1/goals/*` al servicio.  
- **Base de datos**: tabla `User`, `SavingsGoal`, `Milestone`.  

### 2.2 Modelo de datos (Prisma)

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
  previewFeatures = ["clientExtensions"]
}

/**
 * Un usuario registrado en Algora.
 */
model User {
  id          String        @id @default(uuid())
  email       String        @unique
  name        String?
  goals       SavingsGoal[]
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt
}

/**
 * Meta de ahorro creada por un usuario.
 */
model SavingsGoal {
  id           String      @id @default(uuid())
  user         User        @relation(fields: [userId], references: [id])
  userId       String
  title        String
  targetAmount Decimal     // Monto total deseado
  savedAmount  Decimal     @default(0) // Monto acumulado
  deadline     DateTime?   // Opcional
  milestones   Milestone[]
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt

  @@check(savedAmount <= targetAmount) // regla de integridad
}

/**
 * Hito que forma parte de una meta.
 */
model Milestone {
  id            String      @id @default(uuid())
  goal          SavingsGoal @relation(fields: [goalId], references: [id])
  goalId        String
  description   String
  targetPercent Decimal      // % del targetAmount (ej. 0.25 = 25%)
  achievedAt    DateTime?   // null si no está alcanzado
  createdAt     DateTime    @default(now())
}
```

### 2.3 Componentes de código  

| **Componente** | **Responsabilidad** |
|----------------|----------------------|
| `src/index.ts` | Bootstrap de la aplicación (Express, middlewares, routes). |
| `src/server.ts` | Exporta el `http.Server` para pruebas. |
| `src/routes/goals.ts` | Declaración de rutas REST. |
| `src/controllers/goalController.ts` | Orquesta la lógica de la capa de presentación. |
| `src/services/goalService.ts` | Reglas de negocio: crear metas, actualizar ahorro, validar y marcar hitos. |
| `src/middlewares/auth.ts` | Mock de autenticación (en prod se reemplaza por JWT/SSO). |
| `src/middlewares/errorHandler.ts` | Manejo centralizado de errores y respuesta JSON. |
| `src/utils/validator.ts` | Schemas Zod para validar payloads. |
| `tests/*.test.ts` | Tests unitarios e integración (Jest + SuperTest). |
| `swagger.yaml` | Definición OpenAPI 3.0 (documentada en README). |

### 2.4 Flujo de negocio clave  

1. **Creación de meta** → `POST /goals`  
   - Valida payload (`title`, `targetAmount`, opcional `deadline`).  
   - Crea registro `SavingsGoal` con `savedAmount = 0`.  

2. **Agregar hito** → `POST /goals/:id/milestones`  
   - `targetPercent` debe ser > 0 y ≤ 1 y único dentro de la meta.  

3. **Registrar aporte** → `PATCH /goals/:id` (campo `savedAmount`)  
   - Incrementa `savedAmount` (no puede superar `targetAmount`).  
   - Después de actualizar, el **service** revisa todos los hitos: si `savedAmount >= targetAmount * targetPercent` y `!achievedAt`, marca `achievedAt = now()`.  

4. **Consulta** → `GET /goals/:id` devuelve meta + progreso + hitos con estado.  

---  

## 3️⃣ Código Fuente (Production Ready)  

> **Estructura del proyecto**  

```
algora-savings/
├─ src/
│  ├─ index.ts
│  ├─ server.ts
│  ├─ routes/
│  │   └─ goals.ts
│  ├─ controllers/
│  │   └─ goalController.ts
│  ├─ services/
│  │   └─ goalService.ts
│  ├─ middlewares/
│  │   ├─ auth.ts
│  │   └─ errorHandler.ts
│  ├─ utils/
│  │   └─ validator.ts
│  └─ logger.ts
├─ prisma/
│  └─ schema.prisma
├─ tests/
│  └─ goal.test.ts
├─ swagger.yaml
├─ package.json
├─ tsconfig.json
└─ README.md
```

### 3.1 `package.json`

```json
{
  "name": "algora-savings",
  "version": "1.0.0",
  "description": "Goal‑based savings tracking & milestones service for Algora",
  "main": "dist/index.js",
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "test": "jest --coverage",
    "lint": "eslint . --ext .ts",
    "prestart": "npm run build"
  },
  "author": "mau16ro",
  "license": "MIT",
  "dependencies": {
    "@prisma/client": "^5.10.0",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "helmet": "^7.0.0",
    "morgan": "^1.10.0",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "@types/express": "^4.17.17",
    "@types/jest": "^29.5.5",
    "@types/morgan": "^1.9.4",
    "@types/node": "^20.8.0",
    "@types/supertest": "^2.0.12",
    "eslint": "^8.53.0",
    "eslint-config-prettier": "^9.1.0",
    "jest": "^29.7.0",
    "prisma": "^5.10.0",
    "supertest": "^6.3.3",
    "ts-jest": "^29.1.1",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.2.2"
  }
}
```

### 3.2 `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "sourceMap": true,
    "noImplicitAny": true,
    "resolveJsonModule": true,
    "experimentalDecorators": true
  },
  "include": ["src/**/*.ts", "tests/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 3.3 `src/logger.ts`

```ts
import { createLogger, format, transports } from 'winston';

const logger = createLogger({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  format: format.combine(
    format.timestamp(),
    format.errors({ stack: true }),
    format.splat(),
    format.json()
  ),
  transports: [
    new transports.Console({
      format: format.combine(format.colorize(), format.simple()),
    }),
  ],
});

export default logger;
```

### 3.4 `src/middlewares/auth.ts` (Mock)

```ts
import { Request, Response, NextFunction } from 'express';

/**
 * En desarrollo usamos un mock sencillo: se asume que
 * el encabezado `x-user-id` contiene el UUID del usuario.
 * En producción será reemplazado por JWT / SSO.
 */
export function mockAuth(
  req: Request,
  _res: Response,
  next: NextFunction
) {
  const userId = req.header('x-user-id');
  if (!userId) {
    return res.status(401).json({ error: 'Unauthenticated' });
  }
  // Attach user info to request
  (req as any).user = { id: userId };
  next();
}
```

### 3.5 `src/middlewares/errorHandler.ts`

```ts
import { Request, Response, NextFunction } from 'express';
import logger from '../logger';

/**
 * Manejo centralizado de errores.
 * Si el error tiene `status`, se usa; de lo contrario, 500.
 */
export function errorHandler(
  err: any,
  _req: Request,
  res: Response,
  _next: NextFunction
) {
  const status = err.status || 500;
  const message = err.message || 'Internal Server Error';
  logger.error('Error %d: %s', status, message, { stack: err.stack });
  res.status(status).json({ error: message });
}
```

### 3.6 `src/utils/validator.ts`

```ts
import { z } from 'zod';

export const createGoalSchema = z.object({
  title: z.string().min(1, 'Title is required'),
  targetAmount: z
    .string()
    .regex(/^\d+(\.\d{1,2})?$/)
    .transform((s) => parseFloat(s))
    .refine((n) => n > 