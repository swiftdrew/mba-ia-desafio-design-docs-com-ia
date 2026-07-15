# ADR-006: Reuso Máximo de Padrões e Infraestrutura do Projeto

**Status:** Decided
**Data:** 2025-11-01
**Decisor:** Larissa (Tech Lead), Bruno (Senior Engineer, Orders)
**Participantes:** Diego (Senior Engineer, Platform)

---

## Contexto

O projeto existente (Order Management System) tem padrões bem estabelecidos:
- Estrutura modular com controller → service → repository
- Sistema de erros com `AppError` base e subclasses específicas
- Logger centralizado com Pino
- Middleware de validação com Zod
- Padrão de schemas Zod por módulo
- Middleware de autenticação com JWT
- Middleware de autorização com roles
- Pool de conexão Prisma

Decisão arquitetural: o novo módulo de webhooks deve **reaproveitar todos esses padrões** em vez de criar abstrações novas.

## Decisão

O módulo `src/modules/webhooks` deve seguir **exatamente** os padrões do projeto existente:

### 1. Estrutura de Arquivos (como `src/modules/orders/`)
```
src/modules/webhooks/
├── webhook.controller.ts       # HTTP handlers
├── webhook.service.ts          # Business logic
├── webhook.repository.ts       # Database access
├── webhook.schemas.ts          # Zod validation
├── webhook.routes.ts           # Route definitions
├── webhook.security.ts         # HMAC signing (new, specific to domain)
└── webhook.publisher.ts        # Event publishing to outbox (new, specific to domain)
```

Não criar abstrações novas como `WebhookManager`, `WebhookHandler`, etc. Apenas as 5 padrões existentes + 2 específicas do domínio (segurança + publisher).

### 2. Reuso de AppError

**Padrão existente:**
```typescript
// src/shared/errors/app-error.ts
export class AppError extends Error {
  constructor(
    message: string,
    statusCode: number,
    errorCode: string,
    details?: ErrorDetails
  ) { ... }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, `${resource.toUpperCase()}_NOT_FOUND`);
  }
}
```

**Aplicação para webhooks:**
```typescript
// src/modules/webhooks/webhook.errors.ts (opcional, pode ficar em outro arquivo)

export class WebhookNotFoundError extends AppError {
  constructor() {
    super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND');
  }
}

export class WebhookInvalidUrlError extends AppError {
  constructor(reason: string) {
    super(
      `Invalid webhook URL: ${reason}`,
      400,
      'WEBHOOK_INVALID_URL',
      { reason }
    );
  }
}

export class WebhookSecretRequiredError extends AppError {
  constructor() {
    super('Webhook secret is required', 400, 'WEBHOOK_SECRET_REQUIRED');
  }
}

// ... mais códigos de erro com prefixo WEBHOOK_
```

**Integração no error middleware:**
O middleware existente em `src/middlewares/error.middleware.ts` já trata `AppError` automaticamente. Não fazer nada especial, apenas estender `AppError`.

### 3. Reuso de Logger (Pino)

**Padrão existente:**
```typescript
import { logger } from '@/shared/logger';

logger.info({ userId, orderId }, 'Order status changed');
logger.error({ error }, 'Failed to process order');
```

**Aplicação:**
```typescript
// src/modules/webhooks/webhook.worker.ts
import { logger } from '@/shared/logger';

logger.info(
  { eventId: event.event_id, webhookId: webhook.id },
  'Webhook delivered successfully'
);
```

Não criar logger separado, usar o centralizado. Redaction de secrets já é configurada globalmente.

### 4. Reuso de Middleware (authenticate, requireRole, validate)

**Padrão existente:**
```typescript
router.patch(
  '/:id/status',
  validate({ params: orderIdParamSchema, body: updateOrderStatusSchema }),
  controller.changeStatus
);
```

**Aplicação para webhooks:**
```typescript
// src/modules/webhooks/webhook.routes.ts
import { authenticate } from '@/middlewares/auth.middleware';
import { requireRole } from '@/middlewares/role.middleware';
import { validate } from '@/middlewares/validate.middleware';

export function buildWebhookRouter(controller: WebhookController): Router {
  const router = Router();

  // CRUD endpoints: autenticado, sem role específica
  router.use(authenticate);

  router.post('/', validate({ body: createWebhookSchema }), controller.create);
  router.patch('/:id', validate({ params: webhookIdParamSchema, body: updateWebhookSchema }), controller.update);
  router.get('/', controller.list);
  router.get('/:id', validate({ params: webhookIdParamSchema }), controller.get);
  router.delete('/:id', validate({ params: webhookIdParamSchema }), controller.delete);

  // Secret rotation
  router.post(
    '/:id/rotate-secret',
    validate({ params: webhookIdParamSchema }),
    controller.rotateSecret
  );

  // Admin endpoints
  router.post(
    '/admin/dead-letter/:id/replay',
    requireRole('ADMIN'),
    validate({ params: deadLetterIdParamSchema }),
    controller.replayDeadLetter
  );

  return router;
}
```

### 5. Reuso de Padrão Zod Schemas

**Padrão existente:**
```typescript
// src/modules/orders/order.schemas.ts
export const createOrderSchema = z.object({
  customerId: z.string().uuid(),
  items: z.array(orderItemInputSchema).min(1),
  discountCents: z.number().int().nonnegative().default(0),
});

export type CreateOrderInput = z.infer<typeof createOrderSchema>;
```

**Aplicação:**
```typescript
// src/modules/webhooks/webhook.schemas.ts
export const createWebhookSchema = z.object({
  customer_id: z.string().uuid(),
  url: z.string().url().refine((url) => url.startsWith('https://'), {
    message: 'Webhook URL must use HTTPS',
  }),
  events: z.array(z.nativeEnum(OrderStatus)).min(1),
});

export type CreateWebhookInput = z.infer<typeof createWebhookSchema>;

export const webhookIdParamSchema = z.object({
  id: z.string().uuid(),
});

export type WebhookIdParam = z.infer<typeof webhookIdParamSchema>;
```

**Type safety:** Sempre usar `z.infer<>` pra derivar types de schemas.

### 6. Reuso de Padrão Controller/Service/Repository

**Controller (thin, delegates to service):**
```typescript
// src/modules/webhooks/webhook.controller.ts
export class WebhookController {
  constructor(private readonly webhooks: WebhookService) {}

  create: RequestHandler = async (req, res, next) => {
    try {
      const input = req.body as CreateWebhookInput;
      const result = await this.webhooks.create(input, req.user!.id);
      res.status(201).json(result);
    } catch (err) {
      next(err);
    }
  };

  list: RequestHandler = async (req, res, next) => {
    try {
      const customerId = req.body.customer_id; // from query or body
      const result = await this.webhooks.list(customerId);
      res.status(200).json(result);
    } catch (err) {
      next(err);
    }
  };
}
```

**Service (business logic, orchestrates repository + Prisma):**
```typescript
// src/modules/webhooks/webhook.service.ts
export class WebhookService {
  constructor(
    private readonly repository: WebhookRepository,
    private readonly securityService: WebhookSecurityService,
    private readonly prisma: PrismaClient,
  ) {}

  async create(input: CreateWebhookInput, userId: string): Promise<PublicWebhook> {
    // Validate webhook URL is reachable (optional, can be async)
    // Generate secret
    const secret = this.securityService.generateSecret();

    // Create via repository
    const webhook = await this.repository.create({
      customer_id: input.customer_id,
      url: input.url,
      current_secret: secret,
      events: input.events,
      enabled: true,
    });

    // Return public response (without secret logged)
    return this.toPublic(webhook, secret); // Secret returned ONCE
  }

  async rotateSecret(webhookId: string): Promise<{ newSecret: string }> {
    const webhook = await this.repository.findById(webhookId);
    if (!webhook) throw new WebhookNotFoundError();

    const newSecret = this.securityService.generateSecret();
    await this.repository.update(webhookId, {
      previous_secret: webhook.current_secret,
      current_secret: newSecret,
      secret_rotated_at: new Date(),
    });

    // Schedule cleanup of previous_secret after 24h
    setTimeout(async () => {
      await this.repository.update(webhookId, { previous_secret: null });
    }, 24 * 60 * 60 * 1000);

    return { newSecret };
  }

  private toPublic(webhook: Webhook, secret?: string): PublicWebhook {
    return {
      id: webhook.id,
      customer_id: webhook.customer_id,
      url: webhook.url,
      events: webhook.events,
      enabled: webhook.enabled,
      created_at: webhook.created_at,
      secret: secret, // Only returned on creation, never in list/get
    };
  }
}
```

**Repository (data access layer):**
```typescript
// src/modules/webhooks/webhook.repository.ts
export class WebhookRepository {
  constructor(private prisma: PrismaClient) {}

  async create(data: Prisma.WebhookUncheckedCreateInput): Promise<Webhook> {
    return this.prisma.webhook.create({ data });
  }

  async findById(id: string): Promise<Webhook | null> {
    return this.prisma.webhook.findUnique({ where: { id } });
  }

  async list(customerId: string): Promise<Webhook[]> {
    return this.prisma.webhook.findMany({
      where: { customer_id: customerId },
      orderBy: { created_at: 'desc' },
    });
  }

  async update(id: string, data: Prisma.WebhookUncheckedUpdateInput): Promise<Webhook> {
    return this.prisma.webhook.update({ where: { id }, data });
  }

  async delete(id: string): Promise<void> {
    await this.prisma.webhook.delete({ where: { id } });
  }
}
```

### 7. Integração no OrderService

**Ponto de integração:** `src/modules/orders/order.service.ts`

Método existente `changeStatus()` faz:
```typescript
return this.prisma.$transaction(async (tx) => {
  // ... validate, debit stock, update order ...

  // NEW: Publish webhook event
  await publishWebhookEvent(tx, order, from, to);

  // ... return updated order ...
});
```

**Implementação:**
```typescript
// No topo do arquivo order.service.ts
import { publishWebhookEvent } from '@/modules/webhooks/webhook.publisher';

// No método changeStatus
async changeStatus(
  id: string,
  input: UpdateOrderStatusInput,
  userId: string,
): Promise<OrderWithRelations> {
  return this.prisma.$transaction(async (tx) => {
    const order = await tx.order.findUnique({ where: { id }, include: { items: true } });
    if (!order) throw new NotFoundError('Order');

    const { from, to } = this.validateTransition(order.status, input.toStatus);

    // Stock management
    if (shouldDebitStock(from, to)) await this.debitStock(tx, order.items);
    if (shouldReplenishStock(from, to)) await this.replenishStock(tx, order.items);

    // Update order
    await tx.order.update({ where: { id }, data: { status: to } });
    await tx.orderStatusHistory.create({
      data: { orderId: id, fromStatus: from, toStatus: to, changedById: userId },
    });

    // NEW: Publish webhook (atomic with above)
    await publishWebhookEvent(tx, order, from, to);

    // Return refreshed data
    return await tx.order.findUnique({
      where: { id },
      include: { items: { include: { product: { select: { id: true, sku: true, name: true } } } } },
      history: { orderBy: { changedAt: 'asc' } },
      customer: { select: { id: true, name: true, email: true } },
    });
  });
}
```

### 8. Integração na App Setup

**Arquivo:** `src/app.ts`

```typescript
import { WebhookController } from '@/modules/webhooks/webhook.controller';
import { WebhookService } from '@/modules/webhooks/webhook.service';
import { WebhookRepository } from '@/modules/webhooks/webhook.repository';
import { WebhookSecurityService } from '@/modules/webhooks/webhook.security';
import { buildWebhookRouter } from '@/modules/webhooks/webhook.routes';

export function buildControllers(prisma: PrismaClient): Controllers {
  // ... existing controllers ...

  // Webhooks
  const webhookSecurityService = new WebhookSecurityService();
  const webhookRepository = new WebhookRepository(prisma);
  const webhookService = new WebhookService(
    webhookRepository,
    webhookSecurityService,
    prisma
  );
  const webhookController = new WebhookController(webhookService);

  return {
    // ... existing ...
    webhooks: webhookController,
  };
}

export function buildApp(deps: AppDependencies): Express {
  const app = express();
  const controllers = buildControllers(deps.prisma);

  // ... existing middlewares ...

  app.use('/api/v1/webhooks', buildWebhookRouter(controllers.webhooks));
  app.use('/api/v1/admin/webhooks', buildWebhookRouter(controllers.webhooks)); // ou rota separada

  // ... 404 + error middleware ...

  return app;
}
```

---

## Alternativas Consideradas

### 1. Criar nova abstração (WebhookManager, WebhookProcessor)
**Vantagem:** Pode parecer mais "moderno"
**Desvantagem:**
- Quebra padrão do projeto
- Equipe precisa aprender nova abstração
- Futuros novos módulos vão copiar o antigo padrão, inconsistência
- Mais dificuldade pra testes

**Trade-off:** Consistência é mais importante que variedade de abstrações.

### 2. Usar framework/library específica para webhooks (svix, webhook.cool)
**Vantagem:** Muito robusto, features prontas
**Desvantagem:**
- Vendor lock-in
- Custos em produção
- Perda de controle sobre implementação
- Não rola bem com MySQL outbox (integração complicada)

**Trade-off:** Build in-house com padrões locais é melhor pra current scale.

### 3. Manter estrutura existente mas adicionar classes "Manager"
**Vantagem:** Não quebra padrão, adiciona camada
**Desvantagem:**
- Overcomplexity, bloat de código
- Ninguém sabe pra que serve cada classe
- Testes ficam mais chatos (mais mocks)

**Trade-off:** Melhor ficar com padrão puro.

## Consequências

**Positivas:**
- ✅ Consistência: novo módulo segue exatamente padrão do projeto
- ✅ Legibilidade: desenvolvedores já sabem onde procurar (controller/service/repository)
- ✅ Onboarding: novo dev vê padrão uma vez, entende tudo
- ✅ Testes: padrão de testes já existente funciona igual
- ✅ Manutenção: mudanças no padrão afetam webhook igual aos outros módulos
- ✅ Sem vendor lock-in: totalmente em casa

**Negativas:**
- ❌ Responsabilidade: nós implementamos tudo (não há "magic")
- ❌ Sem features prontas: se precisar de feature X no futuro, implementamos
- ❌ Replicação de código: algumas abstrações do padrão são boilerplate

**Trade-off principal:** Consistência + controle em troca de responsabilidade integral.

---

## Ligações

- Referência de código:
  - `src/modules/orders/` (padrão existente)
  - `src/modules/webhooks/` (novo módulo seguindo padrão)
  - `src/shared/errors/` (AppError base)
  - `src/shared/logger/` (logger Pino)
  - `src/middlewares/` (auth, validate, error)
- Implementação: docs/FDD.md - Seção "Integração com o Sistema Existente"
