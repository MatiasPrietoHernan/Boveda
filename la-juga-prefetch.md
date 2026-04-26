---
type: Note
status: Active
tags: [prefetch, precontexto, arquitectura, estandar, receta]
---

# 🧉 La Juga — Prefetch, Filter Tree y Pre-contexto

*"Como el mate: si no preparás el agua, la yerba y el termo antes, cagaste."*

---

## ¿Qué es La Juga?

Es el **ritual de preparación** que todo agente de IA debería hacer antes de contestar. Así como no tomás mate sin cebar primero, no dejás que un agente busque sin preparar el terreno.

Son **3 pasos** que van antes (prefetch), durante (filter tree) y después (pre-contexto) de la búsqueda real.

---

## Los 3 pasos de La Juga

### 🥇 Paso 1: El Cebador (Prefetch)

Antes de buscar, **entendé qué carajo quiere el usuario**.

**Qué hace:** Toma el mensaje crudo y extrae intención + entidades.

**Ejemplo con propiedades:**
```
Usuario: "hola busco un depto de 2 ambientes en capital hasta 450k quiero ver antes del finde"

Prefetch Output:
{
  intent: "compra",
  property_type: "departamento",
  operation: "venta",
  location: { city: "San Miguel de Tucumán" },
  price_range: { max: 450000, currency: "USD" },
  bedrooms: 2,
  urgency: "alta",
  enriched_query: "departamento dos ambientes venta capital tucumán",
  confidence: 0.92
}
```

**Ejemplo con colchones (Yubrin):**
```
Usuario: "hola quiero un colchón de 80x190 queen, peso 85kg, vivo en smt"

Prefetch Output:
{
  intent: "compra",
  measure: "80x190",
  size_type: "queen",
  weight: 85,
  zone: "SMT",
  urgency: "media",
  enriched_query: "colchón 80x190 queen para persona de 85kg san miguel",
  confidence: 0.95
}
```

**Ejemplo con reclamo técnico:**
```
Usuario: "no me funciona el sistema de tracking, me aparece error 500"

Prefetch Output:
{
  intent: "soporte",
  issue_type: "error_500",
  system: "tracking",
  urgency: "alta",
  enriched_query: "tracking error 500 solución",
  confidence: 0.88
}
```

---

### 🥈 Paso 2: El Filtro (Filter Tree)

Con la intención clara, **aplicá los filtros justos** al stock disponible.

**No es buscar a ciegas.** Es saber exactamente qué pedirle a la base vectorial.

```
Prefetch dice: { type: depto, beds: 2, city: capital, max_price: 450k }

Filter Tree construye:
├── tenant_id: [aislado automático]  
├── operation: = venta
├── property_type: = departamento
├── city: = San Miguel de Tucumán
├── bedrooms: ≥ 2
├── price: ≤ 450000
└── semantic: "departamento dos ambientes capital"

→ Qdrant busca con TODOS esos filtros activos
→ Resultados: precisos, no hay que relajar nada
```

**La magia:** Si el prefetch está bien hecho, el filter tree **nunca necesita el Stage 2-3-4 de relajación**. Buscás una sola vez y listo.

---

### 🥉 Paso 3: El Termo (Pre-contexto)

Ya tenés los resultados. Ahora **armá el prompt justo** para que el LLM no alucine.

**Mal (sin pre-contexto):**
```
Prompt: "El usuario quiere un depto. Estos son los resultados. Respondé."
→ El LLM se inventa características, mezcla precios, alucina direcciones.
```

**Bien (con pre-contexto):**
```
Prompt: "El usuario busca comprar un departamento de 2 ambientes en capital 
hasta USD 450,000 con alta urgencia.

Resultados exactos que coinciden:
1. Av. Mitre 850 - 2 dorm, 55m², USD 420,000 - 🏷️ OFERTA
2. San Martín 320 - 2 dorm, 60m², USD 445,000 - 💎 DESTACADO
3. 9 de Julio 150 - 2 dorm, 48m², USD 398,000 - 🔥 OPORTUNIDAD

Políticas relevantes para esta consulta:
- Visitas: se coordinan 24hs antes
- Reserva: 30% para apartar
- Escritura: 15 días hábiles

Instrucción: Respondé AL CLIENTE con estas 3 opciones ordenadas por 
precio. No inventes propiedades que no están en la lista."
```

**Resultado:** El LLM responde con datos reales, cero alucinación.

---

## La Receta Completa (Diagrama)

```
🧉 LA JUGA — Pipeline completo para waiprop

Usuario escribe
    │
    ▼
┌──────────────────────┐
│ 1. 🧉 CEBADOR        │  ← Groq rápido (llama-3.1-8b)  ~200ms
│    Prefetch Analyzer  │
│    → StructuredQuery  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 2. 🏗️ FILTRO         │  ← build_properties_filter()   ~5ms
│    Filter Tree        │
│    → Qdrant call      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 3. 🔥 TERMO          │  ← build_system_prompt()       ~5ms
│    Pre-contexto       │
│    → Prompt armado    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 4. 🤖 RESPUESTA       │
│    LLM con contexto   │  ← Groq + 0 alucinación
│    → Reply al user    │
└──────────────────────┘
```

---

## Casos de uso fuera de waiprop

### 🛒 E-commerce (Yubrin - colchones)
```
Prefetch: medida=80x190, peso=85kg, zona=SMT, tarjeta=visa
Filter Tree: productos con esa medida, envío a SMT, cuotas con visa
Pre-contexto: "Estos 3 colchones coinciden, con cuotas sin interés"
```

### 💬 Chatbot de soporte técnico
```
Prefetch: intent=soporte, sistema=tracking, error=500, urgencia=alta
Filter Tree: knowledge base de errores 500 en tracking
Pre-contexto: "Estos son los pasos para resolver el error 500..."
```

### 📊 Dashboard de métricas
```
Prefetch: intent=reporte, métrica=ventas, periodo=Q1, segmento=commerce
Filter Tree: filtros de fecha + segmento en DB analítica
Pre-contexto: "Ventas Q1: $2.1M, +15% vs Q4, top producto: colchón queen"
```

### 🤝 CRM / Lead scoring
```
Prefetch: intent=calificación, lead=fulanito, interacción=whatsapp, duración=3d
Filter Tree: historial del lead, interacciones, stage actual
Pre-contexto: "Lead caliente, 3 interacciones, preguntó precio 2 veces"
```

---

## Regla de oro

> **No dejes que un LLM busque sin tener contexto. Pero tampoco le des contexto sin haber entendido qué busca.**

📌 Prefetch sin Filter Tree → sabés qué buscar pero buscás mal
📌 Filter Tree sin Prefetch → buscás bien pero no sabés QUÉ
📌 Pre-contexto sin los otros dos → armás un prompt perfecto con datos incorrectos

**Los 3 pasos de La Juga son obligatorios. Siempre.** 🧉

---

## Checklist para implementar

- [ ] Prefetch Analyzer (extraer StructuredQuery del mensaje)
- [ ] El Filter Tree acepta el StructuredQuery como entrada estructurada
- [ ] El Pre-contexto arma el prompt con los resultados formateados
- [ ] El LLM responde SIN alucinar porque el contexto es exacto
- [ ] Logging de cada paso para mejora continua
- [ ] Cache de prefetch (consultas repetitivas ni gastan LLM)
