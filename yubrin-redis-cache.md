---
type: Note
status: Active
related_to: "[[yubrin]]"
tags: [arquitectura, redis, cache, escalabilidad, hot-sale]
---

# Yubrin Scoring — Arquitectura con Redis Cache

## Problema

En Hot Sale, el scoring recibe miles de requests/minuto. Cada una llama a Deepseek API, que tarda 1-3s. Con 4 workers de Gunicorn, el throughput máximo es ~240 req/min.

## Solución: Redis Cache

### Estrategia

```
POST /scoring
    ↓
[Q: regex] → siempre se corre (1-5ms)
    ↓
[Redis: SHA256(conversación + eventos)]
    ↓ hit (ya vimos esta conversación antes)
Devuelve score cualitativo cacheados → 2ms
    ↓ miss
[Deepseek LLM] → genera CL-01 a CL-N3 → 1-3s
[Redis SET con TTL] → guarda resultado → 1ms
Devuelve scoring completo
```

### Configuración propuesta

```yaml
REDIS_URL: "redis://redis:6379"
REDIS_CACHE_TTL: 3600          # 1 hora
REDIS_CACHE_PREFIX: "yubrin:score:"
```

### Workers para Hot Sale

| Componente | Normal | Hot Sale |
|------------|--------|----------|
| Gunicorn workers | 4 | **16** |
| MAX_CONCURRENT_LLM | 20 | **60** |
| LLM_TIMEOUT | 15s | **10s** |
| Redis cache | ❌ | ✅ |

### Conexión con n8n

n8n pega a `POST https://yubrin.matiasprieto.dev/scoring`
con headers:
```json
{
  "X-API-Key": "{{ yubrin_api_key }}",
  "Content-Type": "application/json"
}
```

Body:
```json
{
  "conversation": [
    {"role": "user", "content": "Hola, quiero un colchón de 80x190", "timestamp": "..."}
  ],
  "eventos_bot": ["venta", "tarjeta_bancarizada"]
}
```

Respuesta:
```json
{
  "puntaje_total": 85,
  "etiqueta_intencion": "caliente",
  "score_q": 65,
  "score_cl": 20,
  "cuantitativo": {...},
  "cualitativo": {...}
}
```
