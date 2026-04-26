---
type: Note
status: Active
related_to: "[[yubrin-redis-cache]]"
tags: [yubrin, ponderacion, scoring, agente]
---

# Ponderación — Agente de IA de Yubrin

## Pipeline

```
Mensaje → conversation_id → Extraer conversación → Formatear → 
  POST /scoring → [Q: regex + CL: Deepseek] → Score + Etiqueta
```

## Microservicio de Ponderación

- **Repo:** `yubrin_algoritmo_ponderacion` (privado)
- **Stack:** Python 3.13 + FastAPI + pydantic-ai (Deepseek)
- **Dominio:** `yubrin.matiasprieto.dev`
- **Infra:** Docker + Traefik en servidor personal (207.180.208.88)

## Scoring

```
Score Final = Q × 0.6 + CL × 0.4
Clasificación: ≤29 frío · 30-59 tibio · ≥60 caliente
```

### Cuantitativo (Q) — 14 señales vía regex
| Señal | Pts | Descripción |
|-------|-----|-------------|
| Q-01 | +15 | Medida específica en primer mensaje |
| Q-02 | +15 | Peso corporal declarado |
| Q-03 | +20 | Tarjeta específica mencionada |
| Q-04 | +25 | Dirección de envío |
| Q-05 | +20 | Link de pago solicitado |
| Q-06 | +8 | Preguntó horario/sucursal |
| Q-07 | +10 | URL de producto |
| Q-08 | +7 | Cuotas/financiación |
| Q-09 | +5 | Marca preferida |
| Q-10 | +5 | Zona SMT |
| Q-N1 | -5 | Solo débito |
| Q-N2 | -5 | Interior >100km |
| Q-N3 | -15 | "Está caro" |
| Q-N4 | -20 | Medida fuera de catálogo |

### Cualitativo (CL) — 13 señales vía LLM + 2 temporales
| Señal | Pts | Descripción |
|-------|-----|-------------|
| CL-01 | +30 | 3+ datos en primer mensaje |
| CL-02 | +25 | Contexto familiar emocional |
| CL-03 | +20 | Urgencia personal |
| CL-04 | +20 | 2 datos en primer mensaje |
| CL-05 | +15 | Confianza digital |
| CL-06 | +15 | Tono decisivo |
| CL-07 | +15 | Contexto familiar breve |
| CL-08 | +15 | Peso con humor |
| CL-09 | +10 | Respuesta rápida (<5min) |
| CL-10 | +5 | Expresiones de satisfacción |
| CL-11 | +5 | Apertura a complementos |
| CL-12 | +5 | Tono exploratorio |
| CL-N1 | -10 | Desconfianza digital |
| CL-N2 | -5 | Respuesta lenta (>12h) |
| CL-N3 | -20 | Objeción de precio dura |

## Integración con n8n

n8n llama al microservicio vía HTTP Request node:
- **URL:** `https://yubrin.matiasprieto.dev/scoring`
- **Method:** POST
- **Headers:** `X-API-Key: {{ api_key }}`
- **Body:** conversación formateada + eventos_bot

## Hot Sale — Scaling

Ver [[yubrin-redis-cache]] para la estrategia de escalabilidad con Redis.

## Descubrimiento relacionado

Ver [[prefetch-precontexto-llm]] — técnica de pre-contexto para reducir alucinaciones.
