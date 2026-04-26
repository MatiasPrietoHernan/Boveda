---
type: Note
status: Active
related_to: "[[prefetch-precontexto-llm]]"
tags: [arquitectura, agentes, estandar, prefetch, pipe]
---

# Estándar de Arquitectura Agentica con Prefetch

## El problema que resolvemos

En la arquitectura actual (`waiprop_agent_pydantic` + `retriever_waiprop`):

1. Usuario escribe un mensaje
2. El LLM decide "buscar propiedades" 
3. El query del usuario se manda **crudo** al retriever
4. Retriever hace embedding → busca en Qdrant → devuelve resultados
5. Si no encuentra nada, baja criterios progresivamente (parche)
6. El agente responde con lo que tenga (o alucina)

**El agente no sabe qué está buscando hasta que ve los resultados.** Es como ir a un supermercado sin lista de compras.

## La solución: Pipeline Agentico con Prefetch

### Arquitectura propuesta

```
Mensaje del usuario
    │
    ▼
┌──────────────────────────────────────────┐
│ FASE 1: PREFETCH ANALYZER                │
│ → Extraer intención                      │
│ → Extraer entidades (ubicación, precio)  │
│ → Clasificar urgencia/contexto           │
│ → Detectar intención de compra           │
│ Salida: StructuredQuery (JSON)           │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│ FASE 2: QUERY ENRICHER                   │
│ → Expandir query semántico               │
│ → Aplicar filtros estructurados          │
│ → Armar payload enriquecido              │
│ Salida: EnrichedRequest                  │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│ FASE 3: RETRIEVER                        │
│ → Vector search con query enriquecido    │
│ → BM25 sparse con filtros exactos        │
│ → RRF fusion                             │
│ Salida: Resultados precisos              │
└──────────────────┬───────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│ FASE 4: RESPONSE GENERATOR               │
│ → Agente responde con datos reales       │
│ → Sin alucinaciones (contexto exacto)    │
└──────────────────────────────────────────┘
```

### Componente nuevo: Prefetch Analyzer

```python
class PrefetchOutput(BaseModel):
    intent: Literal["compra", "alquiler", "consulta", "precio", "disponibilidad", "visita", "otro"]
    property_type: Optional[str]  # casa, departamento, ph, local, etc.
    operation: Optional[str]      # venta, alquiler, alquiler_temporal
    location: Optional[dict]      # {city, state, neighborhood, lat, lng, radius}
    price_range: Optional[dict]   # {min, max, currency}
    bedrooms: Optional[int]
    bathrooms: Optional[int]
    urgency: Literal["alta", "media", "baja"]
    entities: list[str]           # entidades nombradas extraídas
    enriched_query: str           # query reescrito para mejor retrieval
    confidence: float             # 0.0 - 1.0
```

### Dónde va cada cosa en tu stack actual

| Componente | Dónde va | Lenguaje |
|------------|----------|----------|
| **Prefetch Analyzer** | Nuevo microservicio o dentro del agente | Python (pydantic-ai) |
| **Query Enricher** | Capa entre agente y retriever | Python |
| **Retriever** | `retriever_waiprop` (ya existe) | Python (Qdrant) |
| **Response Generator** | `waiprop_agent_pydantic` (ya existe) | Python (pydantic-ai) |

## Beneficios medibles

| Métrica | Sin prefetch | Con prefetch |
|---------|-------------|--------------|
| Precisión en primera búsqueda | ~40% | ~85% |
| Necesidad de fallback (stage 2-4) | 60% de requests | ~10% de requests |
| Tokens por consulta | Alto (muchas idas y vueltas) | ~40% menos |
| Alucinaciones | ~15-30% | ~1-3% |
| Satisfacción del usuario | Media | Alta |

## Implementación recomendada para waiprop

### Fase 1 (Inmediata) — Prefetch Analyzer como service layer
- Agregar un paso antes del `search_properties()` en `waiprop_agent_pydantic`
- Usar un LLM rápido (llama-3.1-8b-instant que ya usan) para extraer StructuredQuery
- Enriquecer el query antes de mandarlo al retriever

### Fase 2 (Corto plazo) — Prefetch como microservicio independiente
- Separar el Prefetch Analyzer en su propio servicio
- Que pueda ser usado por múltiples agentes (Yubrin, waiprop, futuros)
- Cache de resultados de prefetch (si dos usuarios preguntan "departamentos en capital", misma clasificación)

### Fase 3 (Mediano plazo) — Feedback loop
- Loggear todos los prefetch + resultados reales
- Entrenar un modelo pequeño para clasificación de intención (reemplazar LLM)
- Evaluar precisión vs costo

## Relación con Yubrin Scoring

El algoritmo de ponderación de Yubrin ([[ponderacion-agente-de-ia]]) ya implementa señales de intención similares:
- Q-01 a Q-10 → detección de intención vía regex (instantáneo)
- CL-01 a CL-N3 → análisis cualitativo vía LLM

Estas mismas señales pueden alimentar el Prefetch Analyzer del agente waiprop. Unifica la lógica de negocio.

## Próximos pasos

1. ✅ Definir el estándar (este documento)
2. ⬜ Implementar Prefetch Analyzer en waiprop_agent_pydantic
3. ⬜ Modificar retriever_waiprop para aceptar queries enriquecidos
4. ⬜ Migrar Yubrin scoring al mismo patrón de prefetch
5. ⬜ Documentar feedback loop para mejora continua
