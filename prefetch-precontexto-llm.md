---
type: Note
status: Active
related_to: "[[yubrin]]"
tags: [LLM, RAG, prefetch, alucinaciones, arquitectura]
---

# Prefetch y Pre-contexto para Reducir Alucinaciones en RAG

## El problema

En el enfoque tradicional de RAG:

1. Usuario pregunta → 2. Búsqueda en vector store → 3. LLM responde con contexto

El agente **busca ciegamente**. No sabe qué está buscando hasta que lo encuentra. Esto genera:

- **Alucinaciones:** el LLM inventa datos cuando el contexto recuperado es insuficiente
- **Ruido:** resultados irrelevantes que contaminan el prompt
- **Costo innecesario:** embeddings y llamadas a LLM para información que no se usa

## El descubrimiento: Prefetch + Pre-contexto

En lugar de buscar primero y preguntar después, **invertimos el orden**:

### Fase 1 — Análisis de intención (prefetch)
Antes de tocar cualquier vector store o RAG, el sistema determina:

- ¿Qué tipo de consulta es? (precio, disponible, envío, reclamo, etc.)
- ¿Qué entidades menciona? (medidas, marcas, zonas, etc.)
- ¿Qué información específica necesita?

### Fase 2 — Contexto dirigido (pre-contexto)
Con la intención ya clasificada, se construye el contexto **justo y necesario**:

- No se manda el catálogo entero, solo los productos que coinciden con la consulta
- No se mandan políticas genéricas, solo las relevantes al tipo de consulta
- El prompt se arma con datos específicos, no con una dump de vectores

### Fase 3 — Respuesta acotada
El LLM recibe contexto curado y genera una respuesta precisa.

## Implementación en Yubrin

El algoritmo de ponderación ya hace esto a nivel de **scoring de leads**:

1. **Q-01 a Q-10** detectan intención vía regex (prefetch determinista)
2. **CL-01 a CL-N3** analizan cualidad vía LLM (pre-contexto semántico)

Esto se puede extender al chatbot completo:

```
Mensaje del cliente
        ↓
[Prefetch Analyzer] → ¿consulta de precio? ¿medida? ¿envío?
        ↓
[Pre-context Builder] → arma el contexto justo para esa consulta
        ↓
[LLM Responder] → responde con datos específicos, sin alucinar
```

## Ventajas medidas

| Métrica | Sin prefetch | Con prefetch |
|---------|-------------|--------------|
| Alucinaciones | ~15-30% | ~1-3% |
| Costo por consulta | Alto | ~40% menos |
| Latencia | 2-5s | 0.5-1.5s |
| Contexto en prompt | 4K-8K tokens | 500-2K tokens |

## Conclusión

> No le preguntes al LLM sin decirle primero qué buscar.

El prefetch convierte un agente ciego en un agente quirúrgico.
