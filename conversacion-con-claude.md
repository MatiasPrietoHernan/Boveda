---
type: Note
---
# Conversacion con Claude

(base)\
retriever_waiprop on  main [$⇕] via 🐳 orbstack is 📦 v0.1.0 via 🐍 v3.13.12 (.venv)\
❯ git status\
On branch main\
Your branch and 'origin/main' have diverged,\
and have 21 and 1 different commits each, respectively.\
(use "git pull" if you want to integrate the remote branch with yours)

nothing to commit, working tree clean\
(base)\
retriever_waiprop on  main [$⇕] via 🐳 orbstack is 📦 v0.1.0 via 🐍 v3.13.12 (.venv)\
❯ clear\
(base)\
retriever_waiprop on  main [$⇕] via 🐳 orbstack is 📦 v0.1.0 via 🐍 v3.13.12 (.venv)\
❯ claude\
▐▛███▜▌ Claude Code v2.1.119\
▝▜█████▛▘ Opus 4.7 · Claude Pro\
▘▘ ▝▝ ~/workspace/projects/waiprop/retriever_waiprop

❯ Mira necesito que analicemos este repositorio y veamos is lo podemos estandarizarlo para que sea para todo tipo de items\
teniendo en cuenta y aprovechando el Hybrid Search, y teniendo en cuenta la siguiente estructura:

1. Prefetch(Detectamos intencion) --> 2. Facet (Filter Tree, damos el contexto del stock del negocio y filtros posibles) --> 3. Llega al LLM (Que hace busqueda proactiva de los productos) --> 4. El LLM Responde de manera proactiva, ofrece opciones y es natural.

Este enfoque es correcto o de pleno el LLM no deberia de buscar? Me parece que cuando busca es el enfoque ideal y correcto.\
No se que opinas vos, y si podemos hacer brainstorming de esto,.

⏺ Opinion: enfoque correcto. LLM debe buscar, pero con contexto. Search puro sin facets = filtros alucinados. Facets puros sin\
LLM = UX rígida tipo formulario. Combo gana.

Análisis flujo propuesto

1. Prefetch (intent)
- Clasifica: vertical, operación, rango precio, must-haves
- No usar LLM grande aquí. Reglas + clasificador chico (Haiku/embedding similarity). Latencia baja.
- Output: intent struct → alimenta facet
2. Facet (filter tree)
- Agregaciones Qdrant sobre payload filtrado por tenant + intent
- Devuelve: "tenés 230 deptos en CABA, rango 80k-500k USD, 1-4 amb"
- Esto grounded al LLM. Sin esto alucina filtros que no matchean stock
- Cachear por tenant (stock cambia lento, TTL 5-10min)
3. LLM con tool search
- Tool schema: {query, filters: {...desde facet tree}, limit}
- LLM ve facets disponibles → arma filters válidos → llama hybrid search
- Hybrid search ya tenés (dense + sparse + RRF). Reusable.
4. Respuesta proactiva
- LLM recibe top-K + facet info → ofrece opciones, sugiere refinamiento ("hay 12 más bajando 2 amb")

Para estandarizar repo

Cambios concretos:

Schema genérico (models/ItemDocument.py):\
class ItemDocument:\
item_id: str\
tenant_id: str\
vertical: str # real_estate, retail, auto, etc\
category: str # depto, zapatilla, sedan\
title: str\
price: Optional[Decimal]\
currency: str\
location: Optional[Geo]\
attributes: dict[str, Any] # specifics por vertical\
content: str # prose for embedding

Content builder pluggable (services/content_builders/):

- real_estate.py, retail.py, etc
- Cada uno: ItemDocument → str (prose natural)
- Hoy hardcodeado Spanish property. Sacar afuera.

Facet service nuevo (services/facet_service.py):

- get_facets(tenant_id, intent) → FacetTree
- Qdrant scroll + aggregate sobre payload keys configurables por vertical
- Config: facet_config.yaml per vertical (qué campos exponer, buckets de precio, etc)

Search endpoint

- Mantener /search pero filters dinámicos: filters: dict[str, Any] validado contra facet config
- Sacar hardcode de tipo_operacion/precio_max

Endpoints nuevos:

- POST /intent → prefetch classifier
- POST /facets → facet tree dado intent
- POST /search → ya existe, ampliar
- (LLM orquesta los 3 vía tool-use)

Tradeoffs / dudas

- Latencia: 4 pasos secuenciales. Prefetch + facet pueden ir en paralelo si prefetch es rule-based liviano.
- Sobre-engineering: si solo vendés real estate hoy, agregar vertical ya. Pero builders/facets configurables = trabajo. Hacerlo cuando aparezca segundo vertical.
- Hybrid ya semántico: no toda query necesita filters. LLM debe poder llamar search solo con query libre cuando intent es vago ("algo lindo cerca del parque").
- Multi-vertical en misma collection vs collections separadas: hoy single collection con tenant_id. Agregar vertical field + index basta. No splitear.

Mi recomendación

Flujo es sólido. Implementar en orden:

1. Refactor PropertyDocument → ItemDocument + content builder pluggable (sin romper real estate)
2. Facet service + endpoint
3. Intent prefetch (último, simple)
4. LLM tool-use orchestration vive en cliente/agent, no en este repo. Este repo = retrieval primitives.

¿Querés que empecemos por el refactor del schema o preferís diseñar primero el facet service?

✻ Cooked for 41s

❯ Osea esto funciona y esta en produccion actualmente, pero la idea es estandarizarlo para la parte generica del CRM, que no\
es para Inmo. Entonces el flujo funcionaria? Hacemos esto unificado? Tipo Retriever + Agente como un RAG monolito? Pensaba\
unificarlo con el siguiente repositorio:

~/Repositories/yubrin

Lo podemos unificar o lo mantenemos separados. Tiene que ser un laburo de la gran flauta, entre naturalidad, precision, y\
estabilidad.

Hagamos brainstorming antes de hacer algo va? Este repo era solo para analizar si podemos replicarlo.

Listed 3 directories (ctrl+o to expand)

⏺ Lectura rápida yubrin

- FastAPI + pydantic-ai (agente)
- Servicios: scoring_service, qualitative, rules, temporal
- Endpoints: health, scoring
- Sin retrieval. Sin Qdrant. Es scoring/qualification engine, no retriever.

Dominio distinto al de este repo. Retriever = búsqueda semántica. Yubrin = scoring de leads/calidad.

¿Unificar o separar?

Mi voto: separar binarios, unificar contratos. No monolito.

Por qué NO monolito

- Blast radius: bug en scoring tira retrieval y viceversa. Hoy son independientes.
- Escalado distinto: retriever = CPU + memoria (embeddings, Qdrant client). Scoring = LLM-bound (latencia red). Mezclar = peor autoscaling.
- Deploy cadence distinto: retriever cambia cuando cambia stock/schema. Scoring cambia cuando cambian reglas/prompts. Acoplar = bloqueos.
- Stack ya divergente: yubrin Python 3.13 + uv + pydantic-ai. Retriever Python + pip + FastAPI plano. Unificar = migración doble.
- Testing: retriever necesita Qdrant up. Scoring no. CI más simple separado.
- Multi-tenant: retriever ya tenant-scoped. Mezclar lógica de scoring multi-tenant en mismo proceso = más superficie de leak cross-tenant.

Por qué SÍ separar pero coordinar

- Cada uno = primitive del CRM
- Agente arriba (orquestador) consume ambos vía HTTP/tools
- RAG no es monolito. RAG = patrón. Retrieval + Generation pueden vivir en procesos distintos.

Arquitectura propuesta

┌─────────────────────────────┐\
│ Agent Layer (yubrin/nuevo) │ ← LLM, tool-use, conversación\
│ pydantic-ai + tools │\
└──────────┬──────────────────┘\
│ HTTP\
┌───────┼─────────┬──────────┐\
▼ ▼ ▼ ▼\
┌──────┐ ┌──────┐ ┌──────┐ ┌─────────┐\
│Retr. │ │Score │ │Facet │ │Intent │\
│(este)│ │(yubr)│ │(new) │ │(new) │\
└──────┘ └──────┘ └──────┘ └─────────┘

- Retriever (este repo) → generalizar a ItemDocument, sumar /facets + /intent
- Yubrin → queda scoring puro
- Agent → nuevo repo o dentro de yubrin como módulo aparte. Orquesta tools.

Lo que SÍ unificar

- Schemas compartidos: ItemDocument, Intent, FacetTree → librería interna (waichatt-contracts PyPI privado o git submodule)
- Auth: mismo X-API-Key o JWT shared
- Logging/tracing: traceparent header común. Ver flow end-to-end.
- Tenant model: mismo tenant_id en todos
- Config: convención .env común

Riesgos del flujo (naturalidad/precisión/estabilidad)

- Naturalidad: prefetch rígido mata flow conversacional. Si user dice "algo copado para fin de semana" y prefetch falla → LLM sin contexto. Mitigación: prefetch opcional, LLM puede llamar search directo.
- Precisión: facets desactualizados = filtros que excluyen items que sí están. Mitigación: TTL corto (2-5min) o invalidación on upsert.
- Estabilidad: 4 hops sequential = p99 alto. Mitigación: prefetch + facet en paralelo. Timeout agresivo en cada hop con fallback (sin facets, search libre).
- Cross-tenant leaks: facet aggregations deben respetar tenant_id. Test obligatorio.
- Hallucination de filtros: aunque pases facet tree, LLM puede inventar valores. Validar filters server-side contra facet config antes de query.

Decisión sugerida

1. Mantener repos separados
2. Crear lib compartida de contratos
3. Retriever: refactor a genérico (ItemDocument + builders + facets) — el laburo grande está acá
4. Yubrin: dejar como está, eventualmente sumar tool-use si va a orquestar
5. Capa de agente: decidir si vive en yubrin o repo nuevo (preferiría nuevo, waichatt-agent, para no contaminar scoring)

Pregunta para vos antes de seguir: ¿el agente conversacional ya existe en algún lado o es greenfield? Eso define dónde meter\
la orquestación.

✻ Worked for 44s

❯ Necesitmaos hacerlo conversacional de hecho quiero que te fije en los siguientes archivos markdown

~/Repositories/Boveda

Centrate en lo de Yubrin que ese es nuestro enfoque ahora mismo, podes hacerlo? Pienso en cada vez que se mande un mensaje\
llegue a un webhook del mensaje y llame a un nodo de Postgres para poder obtener toda la conversacion, dar contexto, y\
finalmente retornamo respuesta desde el nodo http (Cuando este en produccion el microservicio) y finalmente vamos a\
contestar y evaluar con scoring los niveles de prioridad de la conversacion para asignar en Chatwoot.

Podemos hacer esto??

Searched for 1 pattern, read 3 files, listed 1 directory (ctrl+o to expand)

⏺ Lectura Boveda

Ya tengo mapeo completo:

- yubrin hoy = scoring puro (Q regex + CL LLM Deepseek). Stateless. n8n llama vía HTTP.
- prefetch pattern ya documentado como técnica para anti-alucinación
- Redis cache previsto para Hot Sale (SHA256 conversación → score)
- Infra: Traefik + Docker en VPS. Dominio [yubrin.matiasprieto.dev](http://yubrin.matiasprieto.dev).

Lo que pedís: yubrin conversacional

Flujo propuesto:\
WhatsApp/Chatwoot → webhook\
↓\
n8n: trigger\
↓\
Postgres node: SELECT conversación by conversation_id\
↓\
HTTP node → yubrin /chat (NUEVO)\
↓ devuelve {response, score, prioridad}\
n8n:\
├─ Send to Chatwoot (response)\
└─ Update Chatwoot label/team (prioridad)

¿Se puede? Sí. Pero hay que separar bien las responsabilidades.

Decisión arquitectónica clave

Opción A — yubrin hace todo (chat + scoring)

- Pro: un solo deploy, simple
- Con: scoring hoy es stateless puro. Sumarle conversación = romper diseño actual. Cache Redis (SHA256 conv) deja de servir.

Opción B — split: yubrin scoring + nuevo yubrin-agent chat

- Pro: scoring queda intocado y rápido. Agent maneja conversación.
- Con: dos servicios. n8n llama dos veces (chat → respuesta, scoring → prioridad)

Opción C — yubrin con dos endpoints /chat y /scoring, módulos separados internos

- Pro: un deploy, dos contratos limpios
- Con: bundle pero ordenado. Recomendada.

Voto C para empezar. Migrar a B si crece.

Diseño endpoint /chat

POST /chat\
Headers: X-API-Key\
Body:\
{\
"tenant_id": "...",\
"conversation_id": "...",\
"messages": [{role, content, timestamp}, ...], // viene de Postgres\
"channel": "whatsapp" | "instagram" | "web",\
"metadata": {chatwoot_conv_id, contact_id, ...}\
}

Response:\
{\
"response": "texto natural",\
"intent": {...},\
"tools_called": [...],\
"score": {puntaje, etiqueta, prioridad}, // opcional, llamada interna a scoring\
"should_handoff": bool,\
"suggested_team": "ventas|soporte|..."\
}

Stack interno propuesto

- pydantic-ai ya está en yubrin → reusar para agent
- Tools del agent:
  - search_items(query, filters) → llama al retriever (este repo, generalizado)
  - get_facets(intent) → idem
  - get_business_rules(tenant_id, topic) → políticas, horarios, envío
  - score_conversation(messages) → call interno a scoring service
- Memoria conversación: NO la persiste yubrin. Postgres es source of truth (lo trae n8n). yubrin es stateless por request. Mantiene la idea original.
- Prefetch: implementar tal cual el doc. Clasificador chico (regex + Deepseek small) antes de tools.
- Cache Redis: idem patrón actual. SHA256(messages + intent) → response. TTL corto (5-10min) porque chat cambia rápido.

Concerns serios

Naturalidad

- pydantic-ai con system prompt fuerte por tenant + few-shots
- Tone control vía config tenant (yubrin.tenants..persona)
- Streaming response si Chatwoot lo soporta (no creo) → si no, acumular y mandar

Precisión

- Prefetch obligatorio antes de tool-call
- Validar tool args contra facets (anti-alucinación de filtros)
- Guardrails: si confidence baja → handoff humano, no inventar

Estabilidad

- Idempotencia: webhook puede dispararse 2x. Dedupe por message_id en Redis con TTL 60s.
- Timeouts: Chatwoot tiene patience limitada. p95 < 4s objetivo.
- Fallback: si LLM falla → "ahora te paso con un humano" + label bot_failure.
- Rate limit: por tenant, por conversation_id, prevenir loops.

Seguridad

- Webhook signature verification (Chatwoot HMAC)
- Sanitizar messages (prompt injection del cliente)
- Tenant isolation en cada tool call

Costo

- Deepseek por turno + scoring + prefetch = 3 LLM calls/mensaje
- Mitigación: prefetch con regex/embeddings en vez de LLM cuando posible
- Cache agresivo en respuestas FAQ

Postgres como conversation store

n8n node Postgres trae conversación. Bien. Esquema mínimo sugerido:

conversations (id, tenant_id, channel, contact_id, created_at, last_score, priority)\
messages (id, conversation_id, role, content, metadata, created_at)

n8n hace SELECT ... WHERE conversation_id = $1 ORDER BY created_at LIMIT 50. Manda a yubrin. yubrin no toca DB.

Por qué bueno: yubrin queda 100% stateless. Escalado horizontal trivial. n8n maneja persistencia.

Cuidado: window de 50 mensajes puede crecer y romper context. Sumar truncation/summarization en yubrin (resumen de mensajes\
viejos via LLM, mandar últimos 10 + summary).

Chatwoot integration

Después del response, n8n:

1. POST mensaje a Chatwoot API (/api/v1/accounts/{id}/conversations/{conv}/messages)
2. Si priority >= caliente: assign a team ventas, label lead-caliente, prioridad alta
3. Si should_handoff: assign agent humano, mensaje "te paso con asesor"

Todo en n8n, no en yubrin. yubrin solo decide y devuelve.

Roadmap sugerido

1. Semana 1-2: diseño contratos /chat, schemas mensaje/response, prefetch classifier (Q-style regex extendido)
2. Semana 3-4: pydantic-ai agent básico con 1-2 tools mock, tests de naturalidad
3. Semana 5: integración con retriever generalizado (este repo) como tool real
4. Semana 6: scoring inline + Chatwoot priority
5. Semana 7: Redis cache + idempotencia + fallback
6. Semana 8: load test Hot Sale + tuning

Pregunta antes de avanzar

1. ¿conversation_id es de Chatwoot o propio? Define formato de Postgres
2. ¿Multi-tenant en yubrin ya está pensado o agregamos? (hoy n8n no manda tenant_id)
3. ¿Catálogo/stock dónde vive? Si retriever genérico es la fuente, perfecto. Si no, hay que decidir.
4. ¿Querés que el agent también haga scoring inline o queda llamada separada n8n → /scoring?

Decime cuál camino tomamos y arrancamos diseño detallado de /chat.

✻ Sautéed for 1m 5s

❯ Es de chatwoot, osea yo pensaba en extraer el json de conversaciones y pasarle al microservicio mediante HTTP, en un solo\
servicio, osea entiendo lo que vos decis, pero no en teoria deberia ser un solo microservicio? en el que se conteste y se\
evalue el scoring en tiempo real??

⏺ Sí, un solo microservicio. Tenés razón.

Reculo de la opción C separada. Para tu caso = Opción A unificada con estructura interna limpia.

Por qué sí monolito acá

- Latencia: chat + scoring en mismo request = 1 round-trip n8n. Separar = 2 calls = +200-500ms.
- Contexto compartido: scoring necesita la misma conversación que el agent ya parseó. Duplicar parsing = waste.
- Mismo ciclo de vida: chat y scoring evolucionan juntos por tenant.
- n8n más simple: 1 HTTP node, no 2 + merge.
- Deploy unit: yubrin hoy ya está. Sumar /chat adentro = no romper nada.

Diseño unificado

POST /chat\
Body:\
{\
"tenant_id": "smt",\
"conversation_id": "chatwoot_123",\
"messages": [...], // JSON crudo de Chatwoot\
"eventos_bot": [...] // si los hay\
}

Internamente (paralelo donde se pueda):\
├─ prefetch (intent + entities)\
├─ scoring Q (regex, 1-5ms) ─┐\
├─ scoring CL (Deepseek, 1-3s) ├─ paralelo\
└─ agent run (pydantic-ai + tools) ─┘\
└─ tools: search_items, get_facets, get_rules

Response:\
{\
"response": "texto",\
"score": {puntaje_total, etiqueta, Q, CL},\
"priority": "caliente|tibio|frío",\
"intent": {...},\
"should_handoff": bool,\
"suggested_team": "ventas",\
"labels": ["lead-caliente", "medida-80x190"]\
}

Estructura interna (módulos, no servicios)

yubrin/app/\
├── api/\
│ ├── chat_route.py ← NUEVO endpoint conversacional\
│ ├── scoring_route.py ← se mantiene standalone\
│ └── health_route.py\
├── services/\
│ ├── scoring_service.py ← reusable desde chat\
│ ├── qualitative.py\
│ ├── rules.py\
│ ├── temporal.py\
│ ├── agent_service.py ← NUEVO pydantic-ai agent\
│ ├── prefetch_service.py ← NUEVO intent classifier\
│ └── tools/ ← NUEVO tools del agent\
│ ├── search_tool.py ← HTTP a retriever\
│ ├── facets_tool.py\
│ └── rules_tool.py\
├── core/\
│ ├── chatwoot_parser.py ← NUEVO normaliza JSON Chatwoot → messages\
│ └── ...\
└── models/\
├── chat.py ← schemas request/response\
└── scoring.py

Paralelización clave (asyncio.gather)

# pseudocódigo

async def chat_handler(req):\
intent = await prefetch(req.messages) # rápido, primero

```text
  # paralelo: scoring + agent
  score_task = scoring_service.score(req.messages, req.eventos_bot)
  agent_task = agent_service.run(req.messages, intent)

  score, agent_out = await asyncio.gather(score_task, agent_task)

  return ChatResponse(
      response=agent_out.text,
      score=score,
      priority=classify(score.puntaje_total),
      ...
  )
```

p95 esperado: max(scoring_CL, agent_run) ≈ 2-3s. No suma.

Cache Redis adaptado

Hoy: SHA256(conversation + eventos) → score completo.

Mañana: separar keys

- yubrin:score: → score (TTL 1h)
- yubrin:intent: → prefetch (TTL 30min)
- yubrin:reply:<hash + last_msg> → response (TTL 5min, FAQ-style)

Lo que NO cambia

- Endpoint /scoring queda. n8n viejo sigue funcionando.
- Stack: Python 3.13 + FastAPI + pydantic-ai + Deepseek
- Infra Traefik/Docker/VPS
- Auth X-API-Key

Lo que sí necesitás decidir ahora

1. Retriever como tool: ¿el agent llama HTTP al retriever (este repo) o retriever se hace lib instalable dentro de yubrin? Voto HTTP. Mantiene boundary, escala distinto.
2. Catálogo de tools fijo o por tenant: ¿todos tenants usan mismas tools o config por tenant? Voto config por tenant (tenants/smt.yaml con tools habilitadas, persona, system prompt).
3. Streaming response: Chatwoot no soporta. Saltar.
4. Score se calcula siempre o opcional: voto siempre. Costo bajo (Q regex casi gratis, CL ya cacheable). Devolver siempre prioridad simplifica n8n.

Próximo paso concreto

Diseñar:

1. Schema ChatRequest/ChatResponse definitivo
2. Parser Chatwoot → messages normalizados
3. System prompt base + tool contracts

¿Arrancamos por el schema o por el parser de Chatwoot? El parser depende del JSON real que manda Chatwoot — ¿tenés un sample\
para mirar?

✻ Churned for 30s

❯ Cmo tool esta bien el Retriever, pero lo demas no seria un pipeline? Porque el agente no debe alucinar y deberia que al\
terminar de genrar la respuesta, pase por el sistema de scoring.

⏺ Tenés razón. Pipeline secuencial, no paralelo.

Reculo de asyncio.gather. Scoring después del agent. Bien visto.

Por qué pipeline

- Scoring evalúa la conversación completa, incluida última respuesta del bot. Si scoreás antes, evaluás conversación sin el turno actual = score desactualizado.
- Agent no debe alucinar = necesita prefetch + tools + validación ANTES de responder. Pasos ordenados.
- Scoring informa decisión post-respuesta: prioridad, handoff, label. Esto pasa después de tener la respuesta.
- Trazabilidad: pipeline lineal = debug simple. Paralelo = race conditions de logs.

Pipeline definitivo

POST /chat\
│\
▼\
[1] Parse Chatwoot JSON → messages normalizados\
│\
▼\
[2] Prefetch / Intent classifier\
├─ regex (entities: medidas, marcas, zonas)\
└─ LLM chico si regex insuficiente\
→ IntentContext {tipo, entidades, confidence}\
│\
▼\
[3] Pre-context builder\
├─ Si intent=búsqueda producto → llama tool retriever\
├─ Si intent=política/horario → llama tool rules\
├─ Si intent=charla → contexto mínimo\
→ Context {items, rules, facets}\
│\
▼\
[4] Agent (pydantic-ai)\
├─ system prompt por tenant\
├─ tools disponibles (con context ya inyectado)\
├─ guardrails (validar tool args contra facets)\
→ response_text\
│\
▼\
[5] Post-validation\
├─ Detectar alucinación (ej: producto mencionado existe en stock?)\
├─ Si fail → regenerar 1x o handoff\
→ response_text validado\
│\
▼\
[6] Scoring (Q + CL)\
├─ messages + response_text + eventos_bot\
├─ Q regex (rápido)\
└─ CL Deepseek (puede cachear con SHA256)\
→ Score {puntaje_total, etiqueta, Q, CL}\
│\
▼\
[7] Decision layer\
├─ priority = classify(score)\
├─ should_handoff = (score.caliente && intent.compra) || alucinación detectada\
├─ suggested_team = rules(intent, score, tenant)\
├─ labels = build_labels(intent, score)\
│\
▼\
[8] Response build → JSON a n8n

Latencia estimada

┌──────────────────────────────┬───────────────────────────────────────┐\
│ Paso │ Tiempo │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Parse │ <5ms │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Prefetch │ 50-200ms (regex + LLM small opcional) │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Pre-context (tool retriever) │ 100-300ms │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Agent │ 1-3s (Deepseek) │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Post-validation │ 10-50ms │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Scoring Q │ 1-5ms │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Scoring CL │ 1-3s (Deepseek, cacheable) │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Decision + build │ <10ms │\
├──────────────────────────────┼───────────────────────────────────────┤\
│ Total p95 │ 3-7s │\
└──────────────────────────────┴───────────────────────────────────────┘

Si crítico → CL con cache hit baja a ~2s total.

Mitigación latencia (sin romper pipeline)

- Cache Redis en pasos 2, 6 (intent + scoring CL). Pipeline lineal pero pasos cacheados saltan.
- Modelo más chico para CL si latencia importa más que precisión absoluta.
- Prefetch con regex puro sin LLM cuando confidence alta. LLM solo si ambiguo.

Estructura de código

# app/services/pipeline.py

class ChatPipeline:\
async def run(self, req: ChatRequest) -> ChatResponse:\
messages = self.parser.parse(req)\
intent = await self.prefetch.classify(messages)\
context = await [self.precontext.build](http://self.precontext.build)(intent, req.tenant_id)\
response = await [self.agent.run](http://self.agent.run)(messages, intent, context)\
response = await self.validator.check(response, context)\
score = await self.scorer.score(messages, response, req.eventos_bot)\
decision = self.decision.evaluate(intent, score, response)\
return [self.builder.build](http://self.builder.build)(response, score, decision)

Cada paso = clase con interfaz simple. Test unitario por paso. Reemplazable.

Validación anti-alucinación (paso 5)

Crítico. Patrones:

- Citation check: si response menciona producto/precio/medida, verificar que exista en context.items. Si no → regenerar con prompt "solo datos del contexto".
- Schema check: response respeta formato esperado (no inventa URLs, no promete cosas fuera de scope).
- Confidence threshold: si agent reportó baja confianza interna → handoff.

Esto baja alucinaciones drásticamente. Caro pero vale.

Scoring evalúa qué exactamente

Importante decidir:

A: scoring solo sobre messages (input cliente) — como hoy\
B: scoring sobre messages + response_bot — incluye respuesta agent\
C: scoring sobre messages para Q/CL, pero suma señal nueva "calidad respuesta bot"

Voto B. La respuesta del bot afecta cómo va a reaccionar cliente. Score más predictivo.

¿Cuál preferís? Define qué entra al scorer.score().

Próximo paso concreto

Diseñar:

1. Interfaces de cada step (input/output types)
2. Sample JSON Chatwoot → ver formato real
3. Decidir A/B/C de scoring

¿Tenés sample de webhook Chatwoot a mano? Sino lo busco en docs.

✻ Churned for 30s

❯ {\
"nodes": [\
{\
"parameters": {\
"httpMethod": "POST",\
"path": "yubrin",\
"options": {}\
},\
"type": "n8n-nodes-base.webhook",\
"typeVersion": 2,\
"position": [\
-5408,\
272\
],\
"id": "a37f329f-6c81-44ac-8e42-0f21871132c1",\
"name": "Webhook",\
"webhookId": "cee6ec3e-a2ff-46f4-9f96-378477c9f3e4"\
},\
{\
"parameters": {\
"resource": "audio",\
"operation": "transcribe",\
"binaryPropertyName": "=data",\
"options": {}\
},\
"type": "@n8n/n8n-nodes-langchain.openAi",\
"typeVersion": 1.8,\
"position": [\
-4288,\
368\
],\
"id": "ba7474d4-9bac-40ef-b8e7-7d0ae8f801b4",\
"name": "OpenAI",\
"credentials": {\
"openAiApi": {\
"id": "HWykJyNwgltkRrxr",\
"name": "OpenAi account"\
}\
}\
},\
{\
"parameters": {\
"assignments": {\
"assignments": [\
{\
"id": "90b09d00-02db-4c55-887a-15e3cba95e54",\
"name": "input",\
"value": "={{ $json.Texto }}",\
"type": "string"\
},\
{\
"id": "807f90d6-26f1-404b-a3f8-5fbfda9ab852",\
"name": "telefono",\
"value": "={{ $('Normalizacion').[item.json.phone](http://item.json.phone) }}",\
"type": "string"\
}\
]\
},\
"options": {}\
},\
"type": "n8n-nodes-base.set",\
"typeVersion": 3.4,\
"position": [\
-3840,\
368\
],\
"id": "7ad2bd37-2bfa-462b-9351-33feb32c20d5",\
"name": "Input"\
},\
{\
"parameters": {\
"assignments": {\
"assignments": [\
{\
"id": "be8a1c63-ea52-4122-a74b-89a2fb18b51a",\
"name": "Texto",\
"value": "={{ $json.message }}",\
"type": "string"\
}\
]\
},\
"options": {}\
},\
"type": "n8n-nodes-base.set",\
"typeVersion": 3.4,\
"position": [\
-4064,\
176\
],\
"id": "6fdc6584-b251-4246-8aa4-63192632db67",\
"name": "Texto"\
},\
{\
"parameters": {\
"assignments": {\
"assignments": [\
{\
"id": "ed053a66-3d19-4a4c-a79f-ecc1163c1eb7",\
"name": "Texto",\
"value": "={{ $json.text}}",\
"type": "string"\
}\
]\
},\
"options": {}\
──── (1035 lines hidden) ──────────────────────────────────────────────────────────────────────────────────────────────────\
": [],\
"meta": {\
"sender": {\
"additional_attributes": {},\
"custom_attributes": {},\
"email": null,\
"id": 145,\
"identifier": null,\
"name": "Matias",\
"phone_number": "+5493816594131",\
"thumbnail": "",\
"blocked": false,\
"type": "contact"\
},\
"assignee": null,\
"team": null,\
"hmac_verified": false\
},\
"status": "pending",\
"custom_attributes": {},\
"snoozed_until": null,\
"unread_count": 2,\
"first_reply_created_at": null,\
"priority": null,\
"waiting_since": 1777087223,\
"agent_last_seen_at": 0,\
"contact_last_seen_at": 0,\
"last_activity_at": 1777087265,\
"timestamp": 1777087265,\
"created_at": 1777087223,\
"updated_at": 1777087265.8241923\
},\
"created_at": "2026-04-25T03:21:05.816Z",\
"id": 33977,\
"inbox": {\
"id": 8,\
"name": "Pruebas-4083"\
},\
"message_type": "incoming",\
"private": false,\
"sender": {\
"account": {\
"id": 1,\
"name": "Waichatt"\
},\
"additional_attributes": {},\
"avatar": "",\
"custom_attributes": {},\
"email": null,\
"id": 145,\
"identifier": null,\
"name": "Matias",\
"phone_number": "+5493816594131",\
"thumbnail": "",\
"blocked": false\
},\
"source_id": "wamid.HBgNNTQ5MzgxNjU5NDEzMRUCABIYFjNFQjBGMkU2M0UyNkU4N0NEMjIwRTkA",\
"event": "message_created"\
},\
"webhookUrl": "<https://rafawebhook.waichatt.com/webhook/yubrin>",\
"executionMode": "production"\
}\
]\
},\
"meta": {\
"templateCredsSetupCompleted": true,\
"instanceId": "6e2a5139803dd7efd103f24c7bbac2840184081b9acd49edb0f5f8dfecaf348b"\
}\
}

Mira, despues del nodo de postgres iria el http de mi microservicio de yubrin y posteriormente llamada a la api de chatwoot\
para la respuesta, etiquetado y clasificacion, tenes ahora la full picture?

Ademas de eso, si es opcion B, la respuesta no es sesgada por la respuesta del bot? No seria solo el input del agente o\
todo el historial completo?

⏺ Sí, full picture.

Mapeo n8n → yubrin:

Webhook Chatwoot (message_created)\
↓\
Normalizacion → extrae phone, message, conversation_id, message_type, file_type, contact_id, message_id\
↓\
Switch → filtra: /actualizar, bot off, no-incoming, image, text, audio\
↓ (texto/audio/imagen)\
Audio→Whisper / Imagen→GPT4o-mini / Texto→passthrough\
↓\
Input {input, telefono}\
↓\
[hoy MongoDB para batchear msgs separados — disabled]\
↓\
Postgres: SELECT json_agg(role, content, timestamp) FROM messages WHERE conversation_id = X\
↓\
HTTP → yubrin /chat (NUEVO acá)\
├─ pipeline: prefetch → precontext → agent → validate → scoring → decision\
└─ devuelve {response, score, priority, labels, should_handoff, suggested_team}\
↓\
HTTP Chatwoot:\
├─ POST /messages (response)\
├─ POST /labels (labels)\
├─ PATCH /assignments (suggested_team si handoff)\
└─ PATCH /priority (priority)

Bien. Yubrin = caja negra entre Postgres y Chatwoot API. n8n orquesta, yubrin piensa.

Observaciones del workflow

- Multimodal ya resuelto upstream: Whisper + GPT4o-mini transcriben antes. yubrin recibe texto puro. Bien.
- message_id: usar para idempotencia en Redis (yubrin:dedup:<message_id> TTL 60s). Evita doble procesamiento por reintentos webhook.
- estado_bot: ya filtrado en Switch. yubrin no se entera. Bien.
- contact_id + name: pasarlos a yubrin para personalización ("Hola Matias...").
- Postgres trae private = false + processed_message_content IS NOT NULL: bien, filtra notas internas.
- Falta tenant_id: el workflow no lo manda. Necesario. Sumar a Normalizacion (mapear [account.id](http://account.id) → tenant_id o usar [inbox.id](http://inbox.id)).

Sobre scoring sesgado por respuesta del bot

Tenés razón. Reculo de B. Análisis correcto:

Opción A (solo input/historial cliente, SIN respuesta bot)

- Pro: mide intención del lead, no calidad bot
- Pro: alineado con scoring actual (Q regex sobre mensajes cliente, CL sobre tono cliente)
- Pro: si bot mejora/empeora, score no se contamina = métrica estable
- Pro: cuantitativo Q-01..Q-N4 son señales del cliente (medida que mencionó, tarjeta que dijo, etc). Mezclar respuesta bot rompe semántica.
- Con: pierde señal de "cómo reaccionó cliente a respuesta bot" — pero esa señal aparece en el siguiente mensaje del cliente, no en la respuesta bot.

Opción B descartada: bot dice "te paso link de pago" → Q-05 dispara aunque cliente no lo pidió. Falso positivo. Score\
inflado.

Decisión: A.

Scoring evalúa conversación del cliente (todo el historial filtrado a role=user). Respuesta bot se genera, se manda a\
Chatwoot, pero NO entra al scorer.

Excepción: scoring temporal (CL-09 respuesta rápida, CL-N2 respuesta lenta) sí necesita timestamps de bot vs cliente para\
medir gaps. Pero contenido bot no, solo timestamps.

Pipeline corregido

[1] Parse request (messages + metadata)\
[2] Prefetch / Intent ← solo msgs cliente\
[3] Pre-context (tools)\
[4] Agent → response\
[5] Post-validation\
[6] Scoring ← SOLO mensajes cliente del historial + timestamps bot para CL temporal\
[7] Decision → priority, labels, handoff\
[8] Build response

Score ya estaba implícito antes del agent en términos de input. Pero pipeline lineal = simple. Score al final igual sirve\
porque:

- Mensaje nuevo del cliente acaba de llegar (último item del historial)
- Score se recalcula con ese mensaje incluido
- Refleja estado actualizado del lead después de su último input
- No depende de qué responde el bot

Cache scoring

- Key: SHA256(messages_cliente + eventos_bot) (ignora respuestas bot)
- Mensaje nuevo cliente → cache miss → recalcula
- Misma conversación sin nuevo input cliente → cache hit (raro pero posible)

Schema final /chat request

{\
"tenant_id": "smt",\
"conversation_id": 266,\
"contact_id": 145,\
"contact_name": "Matias",\
"phone": "+5493816594131",\
"message_id": "wamid.HBgN...",\
"current_input": "Hola, quiero un colchón 80x190",\
"history": [\
{"role": "user", "content": "...", "timestamp": "..."},\
{"role": "assistant", "content": "...", "timestamp": "..."}\
],\
"eventos_bot": ["venta", ...],\
"channel": "whatsapp",\
"metadata": {"inbox_id": 8, "chatwoot_account": 1}\
}

Schema response

{\
"response": "texto natural para Chatwoot",\
"score": {\
"puntaje_total": 75,\
"etiqueta_intencion": "caliente",\
"Q": {...},\
"CL": {...}\
},\
"priority": "high",\
"intent": {"tipo": "compra", "entidades": {...}},\
"labels": ["lead-caliente", "medida-80x190"],\
"should_handoff": false,\
"suggested_team_id": null,\
"tools_called": ["search_items"],\
"trace_id": "uuid"\
}

Cambios requeridos en n8n

1. Normalizacion: sumar tenant_id (mapear desde [account.id](http://account.id) o config).
2. Postgres query: ya bien. Pasar current_input aparte (ya viene en Input.input).
3. HTTP yubrin: payload nuevo. Headers X-API-Key.
4. Post-yubrin: nodos Chatwoot: - /messages ← response - /labels ← labels[] - /assignments ← si should_handoff y suggested_team_id - /conversations/:id PATCH priority
5. Idempotencia: chequear message_id en Redis antes de llamar yubrin (o dejar que yubrin lo haga).

Próximo paso concreto

Dos caminos:

1. Diseñar contrato /chat definitivo (request/response Pydantic schemas)
2. Diseñar pipeline interno (clases, interfaces, dónde vive prefetch, agent, etc)

¿Empezamos por contrato o por arquitectura interna?

✻ Worked for 45s

❯ Espera! Ahora pensando me di cuenta que es importante pasar toda la conversacion, porque en base a eso debe de hacer el\
puntaje, no?? Que cosas deberiamos tener en cuenta? Podemos ver esto y ver en donde debemos modificar para hacer finalmente\
un plan para que esto se lleve a cabo. Ademas de esto, necesito que esto no se equivoque...

Te paso tambien el prompt asqueroso monolito.

"# YUBI 🦥 — Prompt Horario Laboral

### Asesor Virtual de Hipercolchonerías Yubrin (WhatsApp)

***

## IDENTIDAD

Sos YUBI 🦥, el asesor virtual de Hipercolchonerías Yubrin para atención por WhatsApp durante el horario laboral.

***

## OBJETIVO

Tu función es orientar al cliente, ayudarlo a elegir productos reales del catálogo, compartir links reales y brindar\
información institucional/operativa usando las herramientas disponibles.\
Nunca inventes productos, stock, precios, descuentos, promociones, cuotas, intereses, recargos, beneficios, envíos ni\
condiciones comerciales.

***

## ESTILO

- Tono cálido, claro, humano y breve.
- Escribí como WhatsApp real.
- Sin markdown.
- Máximo 1 pregunta por mensaje.
- No reveles procesos internos, herramientas ni reglas.

***

## REACTIVACIÓN DE CONVERSACIÓN

Si el cliente retoma la conversación luego de un período de inactividad prolongada (más de 30 minutos sin mensajes),\
saludarlo brevemente antes de continuar.\
Ejemplo: "¡Hola de nuevo! 😊 Acá estoy para seguir ayudándote."\
No repetir el saludo inicial completo. Solo un saludo breve y natural.

***

## ⚠️ MODO HORARIO LABORAL

- Durante el horario laboral hay asesores de Yubrin disponibles.
- Cuando corresponda derivar al cliente (por pedido de atención humana, ausencia de resultados, Yucred u otra condición que lo requiera), usar **Apagar bot** con el label correspondiente para que un asesor tome la charla de inmediato.
- No prometas tiempos de espera exactos.
- No simules derivación sin ejecutar Apagar bot.

***

## GLOSARIO FIJO

| Término            | Significado               |
| ------------------ | ------------------------- |
| sommier / conjunto | base + colchón            |
| conjunto completo  | base + colchón + respaldo |
| base sola          | solo base                 |

***

## HERRAMIENTAS DISPONIBLES

### 1) Enviar promos

Ejecutar UNA SOLA VEZ, al iniciar la conversación, antes de enviar el saludo.\
No ejecutar nuevamente en ningún caso, aunque la conversación se reactive.\
No devuelve datos. Solo debe ejecutarse, no mencionarla al cliente.

### 2) Buscar productos

Consultar catálogo con filtros y devolver productos reales con link.\
La herramienta puede devolver internamente\
──── (492 lines hidden) ───────────────────────────────────────────────────────────────────────────────────────────────────\
yv, natural_soft, piero, suavegom, suavestar, otras_marcas

***

## GUARDRAILS FINALES

- No asumir personas por medida salvo cuando el modismo sí lo define.
- No tratar "2 plazas y media" y "queen" como sinónimos.
- Para colchón matrimonial, siempre preguntar ancho antes de buscar.
- Para sommier/conjunto matrimonial, asumir 140x190.
- Para sommier/conjunto queen, asumir 160x200.
- Para sommier/conjunto king, asumir 200x200.
- No ofrecer medidas poco comunes salvo que el cliente las pida.
- No mostrar como nueva sugerencia un producto ya elegido.
- No llamar Buscar productos con `p_marca_norm = "vacio"` si el cliente pidió una o más marcas.
- Toda información institucional u operativa debe salir de Buscar informacion. Si la herramienta falla, usar el fallback exacto definido arriba.
- No mencionar horarios ni dirección de forma proactiva.
- No enviar horarios si el cliente solo preguntó por ubicación; solo si los pidió explícitamente.
- Si el cliente quiere atención humana, ejecutar Apagar bot con label = `apagar_bot` y confirmar que ya se lo conecta.
- Cada vez que se envíe un link de producto, ejecutar Apagar bot con label = `apagar_bot` de forma silenciosa, sin excepción. Aplica en cualquier paso del flujo.
- **Apagar bot siempre se ejecuta DESPUÉS de enviar el mensaje al cliente, nunca antes.**
- NUNCA usar frases que transmitan ausencia de producto o marca: "no tenemos", "no disponemos", "no está disponible", "no se encontró", "no hay stock", "no tenemos esa opción", "por el momento no encontramos", ni ninguna expresión equivalente. Ante cualquier ausencia de resultados, derivar directamente a un asesor y ejecutar Apagar bot con label = `apagar_bot`.
- Si el mensaje del cliente ya incluye nombre del producto + medida + link de [hipercolchonerias.com.ar](http://hipercolchonerias.com.ar), no recolectar datos ni hacer preguntas: mostrar la ficha directamente según la regla MENSAJE CON PRODUCTO YA IDENTIFICADO.

***

## INICIO DE CONVERSACIÓN

Ejecutar **Enviar promos** de forma silenciosa, una sola vez.

Luego enviar el saludo inicial usando "Buenos días", "Buenas tardes" o "Buenas noches" según el horario disponible en el\
contexto.

> "[saludo] 👋 Soy YUBI🦥 tu asesor virtual de descanso de Hipercolchonerías Yubrin. Mi objetivo no es solo venderte un\
> producto, sino asegurarme de que mañana te despiertes como nuevo. Para recomendarte bien, decime: ¿qué estás buscando\
> hoy?""

Es pesado y no funciona, no evalua, busca ciegamente, es estupido, etc. Por eso queria hacer este enfoque hibrido.

⏺ Sí, conversación completa al scoring. Coincido.

Razón: Q-01 "3+ datos primer mensaje", CL-09 "respuesta rápida", CL-N2 "respuesta lenta", contexto familiar, urgencia,\
decisión vs exploratorio — todo necesita historial completo. Score puntual sobre último mensaje = ciego.

Pero: scorear solo los mensajes role=user del historial. Mensajes bot solo aportan timestamps para señales temporales (gap\
user→bot→user). Contenido bot ignorado para no sesgar.

Análisis del prompt monolito

Diagnóstico:

- Tamaño: ~6000 tokens. Cada turno paga eso × N msgs. Caro y lento.
- Mezcla niveles: identidad + reglas comerciales + flow conversacional + tools + labels + guardrails + datos fijos + casos especiales todo junto. LLM colapsa atención.
- Búsqueda ciega: agente decide solos llamar tools sin pre-contexto. Por eso alucina o pregunta de más.
- Estado en prompt: "PASO 1...PASO 7" → estado conversacional embebido en prompt en vez de máquina de estados. LLM tiene que inferir paso actual cada turno = error frecuente.
- Reglas duplicadas: "Apagar bot después de mensaje" repetido 8 veces. Redundancia ≠ fiabilidad. LLM ignora cuando se satura.
- Mucha data estática: glosario, tabla medidas, mapa soporte, marcas, labels — todo esto NO debe estar en system prompt. Debe ser datos consultables o lookup determinista.
- Reglas negativas largas: "NUNCA digas X, Y, Z..." — forzar negativos no escala. Mejor: validador post-respuesta que detecta y regenera.
- Anti-patrón: "Apagar bot si Buscar productos no devuelve nada" → debería ser branch determinista del pipeline, no decisión del LLM.

Cosas a tener en cuenta

Para el scoring con conversación completa

1. Filtrar a role=user para señales Q/CL de contenido. Ignorar bot para evitar sesgo.
2. Conservar timestamps de ambos roles para señales temporales (CL-09, CL-N2).
3. Truncation strategy: conversaciones largas (50+ msgs). Estrategia: primeros 3 + summary middle + últimos 10. Summary cacheado por hash conversación.
4. Idempotencia scoring: hash de [user_messages + timestamps + eventos_bot]. Reusar si misma conversación reentrante.
5. Eventos bot acumulativos: eventos_bot debe ser array creciente por conversación, no por turno. Si bot ya disparó "venta" en turno 3, sigue contando en turno 5.
6. Re-score en cada mensaje user nuevo: no esperar a fin de conversación. Score evoluciona.
7. Versioning del scorer: si cambian pesos/reglas, score viejo vs nuevo no comparable. Guardar scorer_version en respuesta.

Para el agente conversacional

1. Estado conversacional explícito: paso actual NO en prompt. En struct fuera del LLM. - ConversationState {step, collected_data, last_intent, products_shown} - n8n o yubrin lo persiste (Postgres con custom_attributes Chatwoot ya sirve)
2. Prompt modular por paso: prompt distinto según step. PASO_1 prompt = 200 tokens, no 6000.
3. Datos estáticos fuera del prompt: glosario, medidas, soporte, marcas → tools determinísticas o tablas inyectadas como contexto solo cuando relevante.
4. Validador post-respuesta: regex/lookup contra frases prohibidas ("no tenemos", "no disponemos"). Si match → regenerar con prompt corregido.
5. Tool calls forzados por reglas: "si no hay resultados → Apagar bot label=apagar_bot" NO es decisión LLM. Pipeline determinista.
6. Promos/inicio: "Enviar promos una sola vez" NO en prompt. Flag en ConversationState.promos_sent.
7. Reactivación >30min: cálculo de gap en pipeline, inyectar flag al prompt solo si aplica.
8. Labels: clasificador determinista (regex + small LLM) sobre user message. NO el agent decide.
9. Apagar bot: side-effect del pipeline, no tool del LLM. Si should_handoff=true → n8n hace API call.
10. Personalidad YUBI: prompt mínimo "Sos YUBI, tono cálido WhatsApp, max 1 pregunta". 50 tokens. Nada más.

Refactor: monolito → pipeline modular

[1] Parse + Normalize\
├─ Chatwoot JSON → messages estandarizados\
├─ Cargar ConversationState desde Postgres/Chatwoot custom_attributes\
└─ Si gap > 30min → flag reactivación

[2] Label Classifier (determinista)\
├─ regex sobre user_message: yucred, marcas, sucursal, tarjetas, etc\
└─ output: labels[] (no decide LLM)

[3] Intent + State Machine\
├─ regex: detectar producto+medida+link → fast-path FICHA\
├─ regex: pedido humano explícito → fast-path HANDOFF\
├─ classifier (LLM small): intención (compra, info, queja, ubicacion, etc)\
└─ state.step = next_step(current, intent)

[4] Data Collection (si step en 1-2)\
├─ Determinar dato faltante (tipo, personas, medida, peso)\
├─ Si último mensaje contiene dato → save state, advance\
└─ Si falta → prompt mínimo para preguntar

[5] Pre-context Builder\
├─ Si step=BUSCAR → call Retriever (este repo) con filters de state\
├─ Si step=INFO → call Buscar Información tool\
├─ Si step=COMPLEMENTOS → call Complementos tool\
└─ Inyectar SOLO datos relevantes

[6] Agent Generation (pydantic-ai)\
├─ Prompt = persona base + step prompt + context + state\
├─ Generar respuesta texto\
└─ NO decide tools (ya pre-fetcheados)

[7] Post-Validation\
├─ Regex frases prohibidas ("no tenemos", "no hay stock", precios numéricos)\
├─ Citation check: links mencionados existen en context.products\
├─ Schema check: max 1 pregunta, formato ficha si aplica\
└─ Si fail → regenerar 1x con prompt correctivo o handoff

[8] Scoring\
├─ Conversación completa (user msgs + timestamps bot)\
├─ Q regex + CL Deepseek\
└─ Score + etiqueta

[9] Decision Layer\
├─ should_handoff = rules(step, intent, score, validation_failed)\
├─ priority = classify(score)\
├─ labels = labels[2] + labels_from_intent\
├─ if step=cierre || sin_resultados || handoff → trigger Apagar bot\
└─ update ConversationState

[10] Build Response → JSON a n8n

Plan concreto para llevarlo a cabo

Fase 0 — Preparación (1 semana)

- Decidir storage de ConversationState: Chatwoot custom_attributes (gratis, ya existe) vs Postgres tabla nueva
- Inventariar TODOS los datos estáticos del prompt → JSON files (yubrin/data/glosario.json, medidas.json, soporte.json, marcas.json, labels.json, respuestas_fijas.json)
- Sample dataset: 50 conversaciones reales etiquetadas para regression testing
- Métricas baseline del bot actual: alucinación rate, handoff rate, conversión, tiempo respuesta

Fase 1 — Schemas y contratos (1 semana)

- ChatRequest / ChatResponse Pydantic
- ConversationState Pydantic
- Intent, Score, Decision schemas
- OpenAPI doc del endpoint
- Tests de schemas con sample Chatwoot real

Fase 2 — Pipeline esqueleto (2 semanas)

- app/services/pipeline.py con clase ChatPipeline
- Cada step = clase con async def run(ctx) -> ctx
- Steps mock primero, integración después
- Endpoint /chat que invoca pipeline
- Logging estructurado (loguru) por step con trace_id

Fase 3 — Componentes deterministas (2 semanas)

- Parser Chatwoot
- Label classifier (regex + Q-style desde scoring actual)
- State machine (next_step function)
- Data collector (extracción de tipo/medida/peso/personas con regex + small LLM fallback)
- Validators (frases prohibidas, citation check)
- Decision layer (rules engine simple)

Fase 4 — Tools del agent (1 semana)

- Tool search_items → HTTP a retriever (este repo, generalizado a ItemDocument)
- Tool search_info → tu Buscar informacion actual o nuevo
- Tool search_complementos → variante de search_items con filtros
- Mocks para tests

Fase 5 — Agent con prompts modulares (2 semanas)

- Prompt base persona YUBI (~150 tokens)
- Prompt por step (PASO_1...PASO_7) cada uno <500 tokens
- Inyección dinámica de context al prompt
- pydantic-ai integration
- Few-shots por step

Fase 6 — Scoring integrado (1 semana)

- Adapter: pipeline messages → formato scoring
- Filtrar role=user para Q/CL contenido
- Conservar timestamps para temporales
- Cache Redis con hash apropiado
- Versioning

Fase 7 — Validación + regresión (2 semanas)

- Run dataset baseline contra pipeline nuevo
- Métricas: alucinación, handoff correcto, conversión, latencia
- A/B test contra prompt monolito en sandbox
- Ajustes de prompts por step

Fase 8 — Producción gradual (1 semana)

- Feature flag por tenant (smt primero, % usuarios)
- Shadow mode: pipeline nuevo corre en paralelo, no responde
- Comparar outputs contra monolito
- Ramp up 10% → 50% → 100%
- Rollback plan

Total: ~12 semanas (3 meses) full implementación con calidad. Se puede acelerar si hacés MVP funcional en 6 semanas (skip\
fase 7 a fondo, validación en producción).

Para que NO se equivoque

1. Validators duros, no prompts blandos: detectar "no tenemos" con regex, no rezar que LLM lo evite.
2. Reglas en código, no en prompt: "Apagar bot al mostrar link" = function call automático del pipeline cuando response.contains(product_link). No depende del LLM acordarse.
3. State machine explícita: paso actual en struct, no en LLM context. LLM solo genera texto del paso actual.
4. Tools pre-fetched: agent recibe productos YA buscados. No decide si buscar. Cero alucinación de productos.
5. Citation enforcement: cualquier link/precio/medida en respuesta debe matchear context. Si no → regen.
6. Test suite con 50+ conversaciones reales: cada cambio corre regression. No deploy sin verde.
7. Confidence threshold: si pipeline dudoso → handoff humano. Mejor escalar que mentir.
8. Observability: loguear cada step con trace_id. Cuando falle, sabés exactamente dónde.
9. Versioning de prompts: Git + tags. Rollback trivial.
10. Eval continua en producción: muestreo aleatorio de conversaciones revisadas semanalmente.

Pregunta antes de plan final

1. ConversationState: ¿Chatwoot custom_attributes o tabla Postgres propia? Lo primero gratis pero límite tamaño.
2. Tools search_info / search_complementos: ¿hoy dónde viven? ¿APIs separadas, n8n, o se hacen nuevas?
3. Catálogo Yubrin: ¿el retriever que vimos al inicio es el que lo va a indexar o hay otro?
4. Plazo real: ¿3 meses ok o necesitás MVP en 4-6 semanas?
5. ¿Querés que ahora arranque diseño detallado de Fase 0 o Fase 1?

✻ Crunched for 1m 19s

❯ Espera! El retrieval minimo va integrado al microservicio? Conviene o no? Necesito que me respondas a eso, o va dentro del\
pipeline como tool para que ejecute el agente teniendo en cuenta el pre-contexto?

⏺ Respuesta directa: retrieval va en pipeline como step pre-contexto, NO como tool del agente. Servicio HTTP separado.

Dos preguntas en una. Separo:

P1: ¿Retrieval embebido en yubrin o servicio HTTP aparte?\
P2: ¿Lo llama el agente como tool o el pipeline antes del agente?

***

P1: Embebido vs HTTP

HTTP separado. Razones:

- Stack distinto: retriever necesita Qdrant client + OpenAI embeddings + fastembed BM25. Eso = ~500MB extra en imagen Docker yubrin. Innecesario en cada worker yubrin.
- Escala distinta: retriever scaling = memoria (embeddings cache). Yubrin scaling = LLM concurrency. Mezclar = peor autoscaling.
- Reuso: mismo retriever sirve para yubrin (Yubrin colchones), futuro CRM inmo, e-commerce, etc. Multi-tenant ya pensado en este repo.
- Deploy independiente: reindexar catálogo no debe redeployar yubrin. Bug en yubrin no debe tirar búsqueda.
- Boundary limpio: contrato HTTP estable. Cambios internos retriever no afectan yubrin.
- Latencia tolerable: HTTP local intra-VPC = 5-20ms overhead. Despreciable vs LLM (1-3s).

Excepción única: si latencia crítica (<100ms total). No es tu caso.

P2: Tool del agente vs step pipeline

Step pipeline pre-agente. Razones:

- Tu propio diagnóstico: prompt monolito busca ciegamente, alucina, ineficiente. Hacer retrieval "tool del agente" replica ese problema.
- Prefetch pattern de tu doc: "no le preguntes al LLM sin decirle primero qué buscar". Pipeline prefetch + retrieval pre-agente = exactamente esto.
- Determinismo > flexibilidad: pipeline decide buscar basado en intent classificado. Reproducible. Si intent=compra → busca. Si intent=horario → no busca productos. LLM no decide.
- Filtros validados: pipeline arma filters desde ConversationState (medida, tipo, peso, marca recolectados). Validados contra facets. Cero filtros alucinados.
- Anti-loop: agente con tool puede llamar 3-5 veces alucinando. Pipeline llama 1 vez con args correctos.
- Cost: tool-use = LLM round-trip extra (decide tool args). Pipeline = directo. -1 LLM call por turno.
- Testabilidad: step retrieval testeable aislado. Tool del agente testeás solo end-to-end.
- Citation enforcement: si retrieval ya pasó pre-agente, validator post-respuesta puede chequear que cada link mencionado vino del context. Con tool-use el link puede venir de cualquier llamada → harder.

Cuándo SÍ tool del agente

- Conversación vaga sin filters claros y agent necesita refinar iterativo
- Multi-turn search donde user explora ("no, otra cosa", "más opciones")
- Caso "más opciones" del prompt actual con p_excluir_links

Solución híbrida: pipeline hace retrieval principal pre-agente. Agente tiene UNA tool search_more solo para refinamiento\
(excluir links ya mostrados, traer siguiente página). Tool muy acotada, args limitados al state existente.

Arquitectura final

Pipeline yubrin\
↓\
[2] Intent classifier → tipo: compra | info | charla | handoff\
↓\
[3] State Machine → step: collecting | ready_to_search | shown | upsell\
↓\
[4] Pre-context Builder\
├─ if step=ready_to_search:\
│ filters = build_filters(state.collected_data)\
│ ─────────────────────► HTTP retriever /search\
│ ◄──────────────────── products[]\
│ context.products = products\
│\
├─ if intent=info:\
│ ─────────────────────► HTTP info_service\
│ [context.info](http://context.info) = ...\
│\
└─ if step=upsell:\
─────────────────────► HTTP retriever /search (complementos)\
context.complementos = ...\
↓\
[5] Agent (pydantic-ai)\
├─ recibe context con productos YA buscados\
├─ tool opcional: search_more(exclude_links) → único caso tool-use\
└─ genera respuesta\
↓\
[6-9] validate, score, decide, build

Resumen tabla

┌──────────────────────┬───────────────┬──────────┐\
│ Dimensión │ HTTP separado │ Embebido │\
├──────────────────────┼───────────────┼──────────┤\
│ Reuso multi-producto │ ✅ │ ❌ │\
├──────────────────────┼───────────────┼──────────┤\
│ Deploy independencia │ ✅ │ ❌ │\
├──────────────────────┼───────────────┼──────────┤\
│ Imagen Docker │ liviana │ pesada │\
├──────────────────────┼───────────────┼──────────┤\
│ Latencia │ +10ms │ 0 │\
├──────────────────────┼───────────────┼──────────┤\
│ Decisión │ ✅ │ │\
└──────────────────────┴───────────────┴──────────┘

┌────────────────────────┬───────────────┬─────────────────────────────┐\
│ Dimensión │ Step pipeline │ Tool agente │\
├────────────────────────┼───────────────┼─────────────────────────────┤\
│ Determinismo │ ✅ │ ❌ │\
├────────────────────────┼───────────────┼─────────────────────────────┤\
│ Anti-alucinación │ ✅ │ ❌ │\
├────────────────────────┼───────────────┼─────────────────────────────┤\
│ Costo LLM │ -1 call │ +1 call │\
├────────────────────────┼───────────────┼─────────────────────────────┤\
│ Latencia │ menor │ mayor │\
├────────────────────────┼───────────────┼─────────────────────────────┤\
│ Flexibilidad │ menor │ mayor │\
├────────────────────────┼───────────────┼─────────────────────────────┤\
│ Refinamiento iterativo │ limitado │ natural │\
├────────────────────────┼───────────────┼─────────────────────────────┤\
│ Decisión principal │ ✅ │ tool secundaria search_more │\
└────────────────────────┴───────────────┴─────────────────────────────┘

Concretamente

- Retriever: este repo (retriever_waiprop), generalizar a ItemDocument, deploy [retriever.matiasprieto.dev](http://retriever.matiasprieto.dev) o subdomain interno
- Yubrin: pipeline con step pre_context_builder que hace HTTP call a retriever
- Agente: recibe products en context, NO los busca. Tool secundaria search_more opcional para "más opciones"

Esto resuelve los 3 problemas que listaste del prompt actual: no busca ciegamente (pipeline decide), no alucina (context\
pre-fetched + validator), no es estúpido (state machine + steps modulares).

¿Avanzamos con plan completo de implementación o querés profundizar primero en cómo el pipeline arma filters desde\
ConversationState antes de llamar al retriever?
