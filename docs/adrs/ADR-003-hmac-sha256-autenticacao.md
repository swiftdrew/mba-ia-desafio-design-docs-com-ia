# ADR-003: Autenticação HMAC-SHA256 com Secret por Endpoint

**Status:** Decided
**Data:** 2025-11-01
**Decisor:** Sofia (Security Engineer), Larissa (Tech Lead)
**Participantes:** Diego (Senior Engineer, Platform), Bruno (Senior Engineer, Orders)

---

## Contexto

O webhook dispara eventos com dados sensíveis de pedidos para URLs fora da nossa infraestrutura (clientes B2B). É crítico que o cliente possa:
1. **Verificar autenticidade:** garantir que o request veio realmente de nós (não de um atacante)
2. **Detectar tampering:** garantir que o payload não foi alterado em trânsito
3. **Gerenciar secrets com segurança:** rotacionar credentials sem quebrar integrações

Padrão de mercado: Stripe, GitHub, Twilio, AWS usam HMAC para webhooks.

**Ameaças a mitigar:**
- Attacker que intercepta request e modifica payload
- Attacker que fake request de nossa API pra cliente
- Vazamento de secret, necessidade de rotação sem downtime

## Decisão

Implementar **HMAC-SHA256 com secret único por endpoint e rotação com grace period de 24h**:

**Assinatura:**
1. Client registra webhook com nossa API
2. Nós geramos e armazenamos `webhook_secret` (string aleatória, 32+ caracteres)
3. Quando enviamos webhook, calculamos: `signature = HMAC-SHA256(webhook_secret, request_body)`
4. Mandamos signature no header `X-Signature`

**Cliente valida:**
```typescript
// Cliente recebe request
const signature = request.headers['x-signature'];
const body = request.rawBody; // Exato como recebido
const calculated = HMAC-SHA256(webhook_secret, body);
if (signature !== calculated) {
  reject("Webhook não autenticado");
}
```

**Tabela webhook:**
```sql
CREATE TABLE webhooks (
  id CHAR(36) PRIMARY KEY,
  customer_id CHAR(36) NOT NULL,
  url VARCHAR(2048) NOT NULL,  -- Must be HTTPS
  current_secret VARCHAR(255) NOT NULL,
  previous_secret VARCHAR(255) NULL,  -- Para grace period
  secret_rotated_at TIMESTAMP NULL,
  events JSON NOT NULL,  -- Array de OrderStatus que ele quer
  enabled BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  UNIQUE KEY uk_customer_url (customer_id, url),
  INDEX idx_customer (customer_id),
  FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

**Rotação de secret:**
- Endpoint: `POST /webhooks/:id/rotate-secret` (autenticado, role qualquer inicialmente)
- Retorna novo secret ao cliente
- `current_secret` vira `previous_secret`
- Nova geração atribuída a `current_secret`
- `previous_secret` é válido por 24 horas
- Após 24h, `previous_secret` é deletado

**Implementação:**

Arquivo: `src/modules/webhooks/webhook.security.ts`
```typescript
import crypto from 'crypto';

export class WebhookSecurityService {
  /**
   * Gera novo secret aleatório
   */
  generateSecret(): string {
    return crypto.randomBytes(32).toString('hex');
  }

  /**
   * Calcula assinatura HMAC-SHA256 do payload
   */
  sign(secret: string, payload: string): string {
    return crypto
      .createHmac('sha256', secret)
      .update(payload)
      .digest('hex');
  }

  /**
   * Valida signature com current ou previous secret (grace period)
   */
  isValidSignature(
    payload: string,
    signature: string,
    currentSecret: string,
    previousSecret?: string,
  ): boolean {
    const currentValid = this.sign(currentSecret, payload) === signature;
    const previousValid = previousSecret
      ? this.sign(previousSecret, payload) === signature
      : false;
    return currentValid || previousValid;
  }

  /**
   * Rotaciona secret com grace period de 24h
   */
  async rotateSecret(webhookId: string): Promise<{ newSecret: string }> {
    const webhook = await this.repo.findById(webhookId);
    const newSecret = this.generateSecret();

    await this.repo.update(webhookId, {
      previous_secret: webhook.current_secret,
      current_secret: newSecret,
      secret_rotated_at: new Date(),
    });

    // Schedule cleanup of previous secret após 24h
    setTimeout(() => {
      this.repo.update(webhookId, { previous_secret: null });
    }, 24 * 60 * 60 * 1000);

    return { newSecret };
  }
}
```

**Headers enviados:**
```
X-Signature: 2d7e4c91af3f9d8c2b1e6a5f4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c
X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
X-Timestamp: 2025-11-01T14:30:00Z
X-Webhook-Id: 660f8500-e39c-41d4-a816-446655440001
Content-Type: application/json
```

**Validação de URL (TLS obrigatório):**
```typescript
const webhookUrlSchema = z.string().url().refine(
  (url) => url.startsWith('https://'),
  'Webhook URL must use HTTPS'
);
```

---

## Alternativas Consideradas

### 1. Usar OAuth 2.0 client credentials
**Vantagem:** Padrão industry para API-to-API, mais seguro (separação de concerns)
**Desvantagem:**
- Overkill para webhook outbound
- Exigiria cliente ser OAuth provider também
- Complexidade extra pra cliente integrar
- Maior overhead (token exchange a cada requisição ou com refresh)

**Trade-off:** HMAC é mais simples, padrão para webhooks em específico.

### 2. Bearer token em header Authorization
**Vantagem:** Simples, padrão HTTP
**Desvantagem:**
- Vulnerável a "replay attack": mesmo token em múltiplos requests
- Requer encriptação em trânsito (HTTPS), mais complexo de validar
- Sem timestamp, cliente não consegue detectar réplica velha

**Trade-off:** HMAC com X-Timestamp é mais seguro contra replay.

### 3. TLS client certificates (mTLS)
**Vantagem:** Muito seguro, assimétrico
**Desvantagem:**
- Complexidade operacional para cliente
- Gerenciamento de certificados é overhead
- Cliente precisa de PKI infrastructure

**Trade-off:** Muito pesado para webhooks B2B simples.

## Consequências

**Positivas:**
- ✅ Autenticidade garantida: cliente consegue verificar que request veio de nós
- ✅ Integridade: payload não pode ser alterado em trânsito (HMAC protege)
- ✅ Padrão de mercado: Stripe, GitHub, AWS usam igual
- ✅ Rotação segura: grace period de 24h permite transição sem downtime
- ✅ TLS obrigatório: força HTTPS, criptografia em trânsito
- ✅ Sem estado adicional no cliente: HMAC é função pura do payload

**Negativas:**
- ❌ Responsabilidade no cliente: precisa implementar validação (não é automática)
- ❌ Secret vazamento: cliente precisa guardar secret com segurança, se vaza precisa rotacionar
- ❌ Reprocessamento: se cliente perde secret, precisa rotacionar (não é recuperável)
- ❌ Graceful degradation: se previous_secret expirou, cliente em transição quebra

**Trade-off principal:** Segurança via criptografia balanceada contra complexidade operacional reduzida (HMAC é mais simples que mTLS).

---

## Ligações

- Relacionado: ADR-004 (Idempotência com X-Event-Id)
- Implementação: docs/FDD.md - Seção "Contratos Públicos" (Headers)
- Referência de código: `src/modules/webhooks/webhook.security.ts`
