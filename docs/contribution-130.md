# **Entrega Técnica – Resilient Background Job Retry & Monitoring**  
**Proyecto:** Algora – *Resilient background job retry & monitoring*  
**Cliente:** @mau16ro  
**Valor:** $250 USD  

---  

## 1️⃣ Resumen Ejecutivo y Objetivos Técnicos  

| **Objetivo** | **Descripción** |
|--------------|-----------------|
| **Confiabilidad** | Implementar un motor de jobs que garantice la ejecución eventual de tareas asíncronas, con políticas de re‑intento configurables y “dead‑letter” para errores irreparables. |
| **Observabilidad** | Exponer métricas (intentos, éxitos, fracasos, latencia) a Prometheus y crear dashboards básicos en Grafana. |
| **Escalabilidad** | Utilizar una cola basada en Redis que permita múltiples workers en paralelo sin colisión. |
| **Seguridad y Buenas Prácticas** | Tipado estricto (TypeScript), validación de datos, manejo centralizado de errores, pruebas unitarias y de integración, documentación actualizada. |
| **Producción Ready** | Configuración de graceful shutdown, health‑checks, y despliegue vía Docker‑Compose/Kubernetes. |

**Resultado esperado:**  
Un paquete npm (`@algora/background-jobs`) listo para ser importado por cualquier micro‑servicio de Algora, que brinda una API sencilla para registrar, ejecutar y monitorizar jobs de forma resiliente.

---  

## 2️⃣ Arquitectura de la Solución  

### 2.1 Componentes Principales  

| **Componente** | **Tecnología** | **Responsabilidad** |
|----------------|----------------|---------------------|
| **Job Queue** | **BullMQ** (Redis‑backed) | Encolado, reintentos, DLQ, prioridades. |
| **Workers** | Node.js + TypeScript | Consumen jobs, ejecutan lógica de negocio. |
| **Metrics Exporter** | `prom-client` | Exporta métricas en `/metrics`. |
| **Dashboard** | Grafana (opcional) | Visualiza métricas en tiempo real. |
| **Health‑Check** | Express endpoint `/healthz` | Estado del worker y de la conexión a Redis. |
| **Orquestación** | Docker‑Compose (dev) / Helm (prod) | Despliegue reproducible. |

### 2.2 Flujo de Ejecución  

1. **Productor** llama a `JobScheduler.schedule(...)` → crea un job en la cola BullMQ.  
2. **Worker** suscrito a la cola procesa el job:  
   - Si **éxito** → se registra métrica `job_success_total`.  
   - Si **error recuperable** → BullMQ re‑intenta según la política configurada; métrica `job_retry_total`.  
   - Si **error irreparable** (después de `maxAttempts`) → el job se mueve a la **Dead‑Letter Queue (DLQ)** y se dispara una alerta (Slack/Webhook).  
3. **Prometheus** raspa `/metrics` → almacena series temporales.  
4. **Grafana** muestra dashboards con tasas de éxito, tiempo medio de procesamiento, número de jobs en DLQ, etc.  

### 2.3 Diagrama (texto)  

```
+-------------------+          +-------------------+          +-------------------+
|   Application A   |  HTTP    |   JobScheduler    |  Redis   |   BullMQ Queue    |
| (producer code)   |--------->| (scheduleJob)    |<-------->| (jobs + DLQ)      |
+-------------------+          +-------------------+          +-------------------+
                                   |
                                   |  Worker (Node.js + TS)
                                   v
                         +-------------------+
                         |   JobProcessor    |
                         | (execute logic)   |
                         +-------------------+
                                   |
                 +-----------------+-----------------+
                 |                                   |
                 v                                   v
        +-------------------+               +-------------------+
        |  Prometheus Export|               |  DLQ Alert Hook   |
        |   (/metrics)      |               | (Slack/Webhook)   |
        +-------------------+               +-------------------+
```

---  

## 3️⃣ Código Fuente – Implementación Lista para Producción  

> **Estructura del proyecto**  
> ```
> background-jobs/
> ├─ src/
> │  ├─ index.ts                # Export público
> │  ├─ scheduler.ts            # API de encolado
│  │  ├─ worker.ts               # Worker + procesamiento
│  │  ├─ types.ts                # Tipos y interfaces
│  │  ├─ metrics.ts              # Prometheus exporter
│  │  └─ logger.ts               # Winston logger
> ├─ test/
> │  ├─ scheduler.test.ts
│  │  └─ worker.test.ts
> ├─ Dockerfile
> ├─ docker-compose.yml
> ├─ package.json
> └─ README.md
> ```

### 3.1 `src/types.ts`  

```ts
// src/types.ts
import { JobOptions } from 'bullmq';

/**
 * Tipo genérico para la carga útil de un job.
 * Cada job debe implementar esta interface para asegurar tipado.
 */
export interface JobPayload {
  /** Identificador único del recurso relacionado (ej. userId, orderId) */
  id: string;
  /** Datos arbitrarios del job */
  [key: string]: unknown;
}

/**
 * Configuración de re‑intentos y comportamiento del job.
 */
export interface JobRetryPolicy {
  /** Número máximo de intentos (incluyendo el primero) */
  maxAttempts: number;
  /** Retardo base en milisegundos (exponencial) */
  backoffDelayMs?: number;
  /** Tipo de backoff (exponential | fixed) */
  backoffType?: 'exponential' | 'fixed';
}

/**
 * Opciones de programación de un job. Extiende BullMQ JobOptions.
 */
export interface ScheduleOptions extends JobOptions {
  /** Política de re‑intento personalizada */
  retryPolicy?: JobRetryPolicy;
}
```

### 3.2 `src/logger.ts`  

```ts
// src/logger.ts
import winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL ?? 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
  ),
  transports: [new winston.transports.Console()],
});
```

### 3.3 `src/metrics.ts`  

```ts
// src/metrics.ts
import client from 'prom-client';

export const jobSuccessCounter = new client.Counter({
  name: 'job_success_total',
  help: 'Número total de jobs completados con éxito',
  labelNames: ['queue', 'job_name'],
});

export const jobFailureCounter = new client.Counter({
  name: 'job_failure_total',
  help: 'Número total de jobs fallidos (después de reintentos)',
  labelNames: ['queue', 'job_name'],
});

export const jobRetryCounter = new client.Counter({
  name: 'job_retry_total',
  help: 'Número total de reintentos de jobs',
  labelNames: ['queue', 'job_name'],
});

export const jobDurationHistogram = new client.Histogram({
  name: 'job_processing_duration_seconds',
  help: 'Duración del procesamiento de jobs',
  labelNames: ['queue', 'job_name'],
  buckets: [0.1, 0.5, 1, 2, 5, 10],
});

/** Exponer métricas en formato Prometheus */
export const metricsMiddleware = (req: any, res: any, next: any) => {
  if (req.path === '/metrics') {
    res.set('Content-Type', client.register.contentType);
    res.end(client.register.metrics());
  } else {
    next();
  }
};
```

### 3.4 `src/scheduler.ts`  

```ts
// src/scheduler.ts
import { Queue, QueueOptions, JobOptions } from 'bullmq';
import IORedis from 'ioredis';
import { JobPayload, ScheduleOptions } from './types';
import { logger } from './logger';

export class JobScheduler {
  private static instance: JobScheduler;
  private queue: Queue;

  private constructor(queueName: string, redisOpts: IORedis.RedisOptions, opts?: QueueOptions) {
    this.queue = new Queue(queueName, {
      connection: new IORedis(redisOpts),
      ...(opts ?? {}),
    });
  }

  /** Singleton – garantiza una única conexión por proceso */
  public static getInstance(
    queueName = 'algora-default',
    redisOpts: IORedis.RedisOptions = { host: 'localhost', port: 6379 },
    opts?: QueueOptions,
  ): JobScheduler {
    if (!JobScheduler.instance) {
      JobScheduler.instance = new JobScheduler(queueName, redisOpts, opts);
    }
    return JobScheduler.instance;
  }

  /**
   * Programa un job en la cola.
   * @param name    Nombre lógico del job (usado para métricas)
   * @param payload Datos de entrada del job
   * @param opts    Opciones de BullMQ + política de retry personalizada
   */
  async schedule<T extends JobPayload>(
    name: string,
    payload: T,
    opts?: ScheduleOptions,
  ): Promise<void> {
    const jobOpts: JobOptions = {
      attempts: opts?.retryPolicy?.maxAttempts ?? 3,
      backoff: {
        type: opts?.retryPolicy?.backoffType ?? 'exponential',
        delay: opts?.retryPolicy?.backoffDelayMs ?? 5000,
      },
      removeOnComplete: true,
      removeOnFail: false, // keep in DLQ for later analysis
      ...(opts ?? {}),
    };

    await this.queue.add(name, payload, jobOpts);
    logger.info('Job scheduled', { queue: this.queue.name, name, payload });
  }

  /** Exponer la cola para pruebas o extensiones */
  getQueue(): Queue {
    return this.queue;
  }
}
```

### 3.5 `src/worker.ts`  

```ts
// src/worker.ts
import {
  Worker,
  WorkerOptions,
  QueueEvents,
  Job,
} from 'bullmq';
import IORedis from 'ioredis';
import {
  jobSuccessCounter,
  jobFailureCounter,
  jobRetryCounter,
  jobDurationHistogram,
} from './metrics';
import { logger } from './logger';
import { JobPayload } from './types';

/**
 * Interfaz que deben implementar los handlers de jobs.
 */
export interface JobHandler<T extends JobPayload> {
  /** Nombre lógico del job (debe coincidir con el usado en el scheduler) */
  jobName: string;
  /** Lógica de negocio que se ejecuta por job */
  handle(payload: T): Promise<void>;
}

/**
 * Clase que encapsula la creación y gestión de workers.
 */
export class JobWorker {
  private worker: Worker;
  private queueEvents: QueueEvents;

  /**
   * @param queueName   Nombre de la cola en Redis
   * @param redisOpts   Configuración de conexión a Redis
   * @param handlers    Array de handlers que el worker escuchará
   * @param opts        Opciones de BullMQ Worker
   */
  constructor(
    queueName: string,
    redisOpts: IORedis.RedisOptions,
    handlers: JobHandler<JobPayload>[],
    opts?: WorkerOptions,
  ) {
    const connection = new IORedis(redisOpts);
    const handlerMap = new Map(handlers.map((h) => [h.jobName, h.handle]));

    this.worker = new Worker(
      queueName,
      async (job: Job) => {
        const start = Date.now();

        const handler = handlerMap.get(job.name);
        if (!handler) {
          logger.error('No handler found for job', { jobName: job.name });
          throw new Error(`Handler not found for job ${job.name}`);
        }

        try {
          await handler(job.data);
          const durationSec = (Date.now() - start) / 1000;
          jobSuccessCounter.inc({ queue: queueName, job_name: job.name });
          jobDurationHistogram.observe({ queue: queueName, job_name: job.name }, durationSec);
          logger.info('Job processed successfully', { jobId: job.id, jobName: job.name });
        } catch (err) {
          // Si todavía quedan intentos, BullMQ volverá a intentar automáticamente.
          if (job.attemptsMade < (job.opts.attempts ?? 1)) {
            jobRetryCounter.inc({ queue: queueName, job_name: job.name });
            logger.warn('Job failed, will retry', {
              jobId: job.id,
              jobName: job.name,
              attempt: job.attemptsMade,
              error: err instanceof Error ? err.message : err,
            });
            throw err; // re‑throw para que BullMQ haga el retry
          }

          // Si llegamos aquí, agotamos los intentos → enviamos a DLQ.
          jobFailureCounter.inc({ queue: queueName, job_name: job.name });
          logger.error('Job failed permanently, moving to DLQ', {
            jobId: job.id,
            jobName: job.name,
            error: err instanceof Error ? err.message : err,
          });
          // Aquí podríamos notificar a Slack/Webhook, pero lo delegamos a `queueEvents`.
          throw err;
        }
      },
      {
        connection,
        ...(opts ?? {}),
      },
    );

    //