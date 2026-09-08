# Plan exhaustivo de mejora de eficiencia de IA agéntica en repositorios ASP.NET Core

Lo estructuraría como un proyecto de optimización progresiva, donde cada fase debe demostrar que reduce exploración, tokens o tiempo antes de añadir complejidad adicional.

La arquitectura objetivo sería:

```text
                         ┌─────────────────────┐
                         │ ASP.NET Core Repo   │
                         └──────────┬──────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           │                        │                        │
           ▼                        ▼                        ▼
    Reglas/Workflows             Roslyn                  Repomix
   Instructions/Skills              │                        │
           │                        │                        ▼
           │                structural facts           LLM barato
           │                        │                        │
           │                        ▼                        ▼
           │                structural.json          semantic.json
           │                        │                        │
           │                        └──────────┬─────────────┘
           │                                   ▼
           │                            Index Builder
           │                                   │
           │                          .ai-cache/index.db
           │                                   │
           └──────────────────┬────────────────┘
                              ▼
                         MCP Repo Intelligence
                              │
              ┌───────────────┼─────────────────┐
              ▼               ▼                 ▼
        search_feature    related_code      impact_map
              │               │                 │
              └───────────────┼─────────────────┘
                              ▼
                      GitHub Copilot Agent
                              │
                     5-10 archivos reales
                              │
                              ▼
                           edición
                              │
                              ▼
                    build mínimo afectado
                              │
                              ▼
                     tests relacionados
                              │
                              ▼
                           Problems
                              │
                        error ─┴─► corregir
```

VS Code permite actualmente combinar instrucciones, `AGENTS.md`, prompt files, Agent Skills, custom agents, tool sets y herramientas MCP; además, los Skills están diseñados para cargarse progresivamente solo cuando son relevantes. Eso encaja especialmente bien con el objetivo de reducir contexto permanente.

## 0. Objetivo y métricas antes de modificar nada

Antes de optimizar, establecería un baseline con unas 10-20 tareas reales del proyecto.

Por ejemplo:

| Tipo de tarea | Ejemplo |
|---|---|
| Localización | Encontrar dónde se calcula X |
| Bug pequeño | Corregir validación |
| Cambio transversal | Añadir `CancellationToken` |
| Nueva feature | Añadir endpoint |
| Persistencia | Añadir campo a entidad |
| Refactor | Cambiar interfaz |
| Tests | Añadir cobertura |
| Diagnóstico | Encontrar causa de excepción |

Medir para cada una:

| Métrica | Objetivo |
|---|---:|
| archivos inspeccionados | ↓ |
| búsquedas realizadas | ↓ |
| tool calls | ↓ |
| tokens/contexto aproximado | ↓ |
| comandos ejecutados | ↓ |
| builds completos | ↓ |
| tests ejecutados innecesariamente | ↓ |
| tiempo hasta primera edición correcta | ↓ |
| ciclos editar→fallar→corregir | ↓ |
| errores por contexto incompleto | ↓ |
| tasa de tarea correcta | ≥ actual |

No optimizaría exclusivamente tokens. Una solución que consume 30 % menos pero introduce más errores sería una regresión.

Objetivo inicial razonable:

```text
50-80 % menos archivos inspeccionados
30-60 % menos contexto usado para orientación
50-90 % menos tests ejecutados por iteración
reducción apreciable del tiempo hasta primera modificación
sin empeorar la tasa de éxito
```

# MEDIDA 1 — Optimización básica del agente

Esta fase debe funcionar independientemente de Repomix.

Objetivo:

```text
Prompt
   ↓
Copilot
   ↓
localiza correctamente el área
   ↓
obtiene relaciones estructurales
   ↓
abre pocos archivos
   ↓
edita
   ↓
valida únicamente lo necesario
```

## 1.1. Limpiar primero la arquitectura física del repo

Idealmente:

```text
repo/
│
├── src/
│   ├── Api/
│   ├── Application/
│   ├── Domain/
│   └── Infrastructure/
│
├── tests/
│   ├── UnitTests/
│   └── IntegrationTests/
│
├── tools/
├── scripts/
├── .github/
└── Backend.sln
```

No hace falta aplicar Clean Architecture estrictamente, pero las fronteras deben ser reconocibles.

Evitar directorios ambiguos gigantes como:

```text
Helpers/
Utils/
Common/
Managers/
Misc/
Services/
```

si contienen responsabilidades heterogéneas.

El árbol del repositorio ya constituye un primer índice gratuito para la IA.

## 1.2. Crear una jerarquía de conocimiento, no un único `.md` gigante

Distribuiría el conocimiento así:

```text
.github/
├── copilot-instructions.md
│
├── instructions/
│   ├── api.instructions.md
│   ├── application.instructions.md
│   ├── domain.instructions.md
│   ├── infrastructure.instructions.md
│   └── tests.instructions.md
│
├── agents/
│   ├── backend-analysis.agent.md
│   └── backend-implementation.agent.md
│
├── prompts/
│   ├── analyze-impact.prompt.md
│   ├── implement-feature.prompt.md
│   └── fix-test.prompt.md
│
└── skills/
    ├── backend-testing/
    ├── ef-migration/
    └── endpoint-change/

AGENTS.md
src/Application/AGENTS.md
src/Infrastructure/AGENTS.md
tests/AGENTS.md
```

VS Code y Copilot soportan instrucciones globales, instrucciones por ruta y `AGENTS.md`; las reglas path-specific evitan cargar reglas de Infrastructure cuando el agente está trabajando solamente en Domain, por ejemplo.

### Reparto recomendado

`copilot-instructions.md`:

```text
arquitectura global
reglas innegociables
convenciones esenciales
cómo compilar
cómo validar
cómo encontrar información
```

Debe ser pequeño.

No meter:

```text
documentación exhaustiva
explicaciones históricas
reglas que solo afectan a EF
tutoriales
ejemplos enormes
```

## 1.3. `AGENTS.md` para reglas locales

Ejemplo:

```text
src/Application/AGENTS.md

- Application must not depend on Infrastructure.
- Commands are handled through MediatR.
- Validation uses FluentValidation.
- CancellationToken must be propagated.
- Prefer Result<T> for expected domain failures.
```

Y:

```text
tests/AGENTS.md

- Prefer existing fixtures.
- Unit tests must not use a real database.
- Test naming convention: Method_Condition_Result.
- Run filtered tests before running the complete project.
```

Así no necesitas repetir estas instrucciones en cada petición.

## 1.4. Sacar workflows de las instrucciones y convertirlos en Skills

Esta sería una mejora importante respecto a limitarse a `.md`.

Las instrucciones sirven para:

> cómo escribimos código.

Los Skills sirven mejor para:

> cómo realizamos una determinada clase de tarea.

VS Code indica que los Agent Skills pueden incluir instrucciones, scripts y recursos, y que el contenido relevante se carga progresivamente cuando es necesario.

Ejemplo:

```text
.github/skills/endpoint-change/
├── SKILL.md
├── validate-endpoint.ps1
└── examples.md
```

`SKILL.md`:

```text
When modifying an API endpoint:

1. Identify endpoint declaration.
2. Identify command/query/handler.
3. Determine services and repositories used.
4. Determine affected contracts.
5. Find corresponding tests.
6. Implement smallest coherent change.
7. Build affected project.
8. Execute related tests.
9. Inspect VS Code Problems.
```

No cargas este procedimiento cuando estás haciendo otra tarea.

## 1.5. Crear prompts reutilizables

Por ejemplo:

```text
.github/prompts/analyze-impact.prompt.md
```

para ejecutar:

```text
/analyze-impact IOrderRepository.SaveAsync
```

El flujo debería exigir:

```text
NO editar inicialmente.

1. Identificar símbolo.
2. Buscar implementaciones.
3. Buscar callers/references.
4. Identificar endpoints afectados.
5. Identificar tests.
6. Presentar conjunto mínimo de archivos.
7. Solo entonces proponer modificación.
```

VS Code permite que estos `.prompt.md` definan agente, modelo y herramientas disponibles.

## 1.6. Separar agente de investigación y agente de implementación

Actualmente VS Code permite custom agents definidos mediante `.agent.md`, incluyendo las herramientas que pueden utilizar.

Crearía:

```text
backend-analysis.agent.md
```

Herramientas:

```text
search
usages
read
Problems
CodeGraph
```

Sin herramientas de edición.

Su trabajo:

```text
problema
 ↓
mapa de impacto
 ↓
archivos candidatos
 ↓
plan
```

Y:

```text
backend-implementation.agent.md
```

con:

```text
read
edit
CodeGraph
terminal
Problems
```

Flujo:

```text
Analysis Agent
      ↓
plan reducido
      ↓
Implementation Agent
      ↓
edición
```

Esto evita que el agente empiece a cambiar cosas mientras todavía está descubriendo la arquitectura.

## 1.7. Limitar herramientas con Tool Sets

VS Code permite crear conjuntos reutilizables de herramientas, incluidos tools built-in, MCP y de extensiones.

Ejemplo conceptual:

```json
{
  "backendResearch": {
    "tools": [
      "search/codebase",
      "search/usages",
      "read/problems",
      "codegraph/*"
    ]
  },

  "backendImplementation": {
    "tools": [
      "search/codebase",
      "read/problems",
      "codegraph/*",
      "edit",
      "terminal"
    ]
  }
}
```

La filosofía es:

```text
menos herramientas disponibles
        ↓
menos decisión del LLM
        ↓
menos llamadas irrelevantes
```

No activaría veinte MCP permanentemente si una tarea de backend solo necesita cinco herramientas.

## 1.8. Crear scripts deterministas para build y tests

Estructura:

```text
scripts/
├── build-project.ps1
├── test-project.ps1
├── test-filter.ps1
├── test-changed.ps1
└── validate-backend.ps1
```

Por ejemplo:

```text
test-filter.ps1 "CreateOrderCommandHandlerTests"
```

en vez de permitir que Copilot tenga que averiguar cada vez cómo ejecutar xUnit.

El agente debe preferir:

```text
cambio en Application
     ↓
build Application.csproj
     ↓
tests relacionados
```

antes que:

```text
dotnet build Backend.sln
dotnet test Backend.sln
```

## 1.9. Añadir `test-changed`

Es una de las mejoras con mayor retorno.

Inicialmente puede utilizar algo sencillo:

```text
git diff
    ↓
proyectos modificados
    ↓
proyectos de test asociados
```

Posteriormente:

```text
símbolos modificados
    ↓
CodeGraph
    ↓
tests que los referencian
```

Resultado:

```text
OrderService.cs cambiado

Related tests:
  OrderServiceTests
  CreateOrderHandlerTests

Execute:
  dotnet test ... --filter ...
```

## 1.10. Crear CodeGraph MCP con Roslyn

Aquí empieza la optimización estructural real.

Roslyn ya puede resolver referencias, implementaciones y overrides a nivel de solución mediante `SymbolFinder`, por lo que estas relaciones no deberían delegarse a un LLM.

Primera versión del MCP:

```text
find_symbol
find_references
find_implementations
find_callers
```

Nada más.

### `find_symbol`

Entrada:

```text
IOrderRepository
```

Salida:

```json
{
  "symbol": "Company.Application.Orders.IOrderRepository",
  "kind": "interface",
  "file": "src/Application/Orders/IOrderRepository.cs",
  "line": 8
}
```

### `find_references`

Entrada:

```text
IOrderRepository.SaveAsync
```

Salida:

```text
CreateOrderHandler.cs:45
UpdateOrderHandler.cs:57
OrderRepositoryTests.cs:81
```

### `find_implementations`

```text
IOrderRepository
    ↓
OrderRepository
FakeOrderRepository
```

### `find_callers`

```text
OrderService.CancelAsync
    ↓
CancelOrderCommandHandler.Handle
OrderExpiryWorker.ExecuteAsync
```

## 1.11. Regla crítica: MCP devuelve metadatos, no código

Incorrecto:

```text
find_references()
→ 24 archivos completos
→ 70k tokens
```

Correcto:

```text
find_references()
→ 12 referencias
→ path
→ símbolo
→ línea
→ opcionalmente 1 línea de contexto
```

Después Copilot abre solo lo necesario.

## 1.12. Extender CodeGraph para ASP.NET Core

Cuando las cuatro operaciones básicas sean fiables:

```text
list_endpoints
trace_endpoint
resolve_service
describe_entity
find_related_tests
impact_analysis
```

### Endpoints

```text
POST /api/orders
   ↓
OrdersController.Create
   ↓
CreateOrderCommand
```

### Dependency Injection

```text
IOrderRepository
      ↓
OrderRepository
      ↓
Scoped
      ↓
Infrastructure/DependencyInjection.cs:73
```

### EF Core

```text
Order
 ├─ DbSet: DbContext.Orders
 ├─ Configuration: OrderConfiguration
 ├─ Table: Orders
 └─ Relationships:
      Customer
      OrderLines
```

### Tests

```text
CreateOrderHandler
    ↓
CreateOrderHandlerTests
OrderControllerTests
```

## 1.13. Crear `impact_analysis`

Ésta acabaría siendo una de las herramientas más importantes.

Entrada:

```text
IOrderRepository.SaveAsync
```

Salida:

```text
Definitions: 1

Implementations: 2

Direct callers: 4

Affected features:
  Orders/Create
  Orders/Update

Affected endpoints:
  POST /api/orders
  PUT /api/orders/{id}

Related tests:
  CreateOrderHandlerTests
  UpdateOrderHandlerTests
  OrderRepositoryTests

Suggested files to inspect: 8
```

El agente ha pasado de:

```text
repo de 3.000 archivos
```

a:

```text
8 archivos candidatos
```

antes de gastar contexto serio.

# RESULTADO DE MEDIDA 1

El workflow objetivo:

```text
                  PROMPT
                     │
                     ▼
              Copilot Agent
                     │
       ┌─────────────┴──────────────┐
       │                            │
 semantic/code search           CodeGraph
       │                            │
 significado                  relaciones
       │                            │
       └─────────────┬──────────────┘
                     ▼
                impact map
                     │
                     ▼
                5-10 files
                     │
                     ▼
                   edit
                     │
                     ▼
            smallest possible build
                     │
                     ▼
                related tests
                     │
                     ▼
                 Problems
                     │
               error ───► edit
```

Con esto ya debería existir una reducción grande de consumo incluso sin Repomix.

# MEDIDA 2 — Repomix + modelo gratuito como precocinador semántico

La segunda medida no debe sustituir la primera.

Debe añadir:

```text
"qué significa este código"
```

a la información estructural de:

```text
"qué símbolo depende de qué símbolo"
```

## 2.1. Principio de diseño

No:

```text
Repo
 ↓
Repomix
 ↓
LLM gratis
 ↓
super-resumen.md
 ↓
Copilot lee 50k tokens
```

Eso simplemente cambia una forma de contexto grande por otra.

Debe ser:

```text
Repo
 ↓
Repomix
 ↓
LLM barato
 ↓
datos estructurados
 ↓
índice consultable
 ↓
MCP
 ↓
500-2.000 tokens relevantes
 ↓
Copilot
```

## 2.2. Repomix como ETL, no como memoria final

Repomix puede producir XML, Markdown, JSON o plain text. También dispone de `--compress`, que extrae estructura esencial del código, y permite seleccionar rutas mediante globs.

Crearía dos perfiles.

### Perfil semántico

```bash
repomix \
  --style xml \
  --compress \
  --output .ai-build/semantic-source.xml
```

Objetivo:

```text
LLM gratis entiende:
módulos
responsabilidades
features
conceptos
```

### Perfil de detalle

Solo bajo demanda:

```bash
repomix \
  --include "src/Application/Orders/**,src/Domain/Orders/**" \
  --output .ai-build/orders.xml
```

Repomix permite `--include`, `--ignore` y división de salidas grandes, por lo que no es necesario enviar siempre el repo completo.

## 2.3. Nunca enviar secretos al modelo gratuito

Excluir:

```text
.env
appsettings.Production.json
secrets.json
*.pfx
*.key
credentials
bin/
obj/
logs/
generated/
```

Y cualquier código que contractualmente no pueda enviarse a ese proveedor.

Ésta debe ser una condición previa obligatoria.

## 2.4. El modelo barato debe producir JSON rígido

No pedir:

> Analiza mi repositorio.

Pedir:

```text
Classify the supplied code.

Do not suggest modifications.

Return valid JSON only.

For every detected feature provide:

{
  "name": "",
  "description": "",
  "keywords": [],
  "entry_points": [],
  "main_symbols": [],
  "related_files": [],
  "dependencies": [],
  "database_concepts": [],
  "tests": [],
  "confidence": 0.0
}
```

Resultado:

```json
{
  "name": "OrderCancellation",
  "description": "Cancels an existing order after authorization and status checks.",
  "keywords": [
    "cancel order",
    "order cancellation",
    "refund"
  ],
  "entry_points": [
    "OrdersController.Cancel"
  ],
  "main_symbols": [
    "CancelOrderCommand",
    "CancelOrderCommandHandler"
  ]
}
```

## 2.5. Utilizar varias pasadas

En vez de una llamada gigantesca:

```text
PASADA A
clasificación por módulos

PASADA B
features dentro de módulos

PASADA C
resumen semántico de feature

PASADA D
keywords/sinónimos

PASADA E
relaciones conceptuales globales
```

Esto permite usar modelos más pequeños.

## 2.6. Roslyn y LLM nunca deben competir

Reparto estricto:

| Pregunta | Responsable |
|---|---|
| ¿Dónde se declara X? | Roslyn |
| ¿Quién referencia X? | Roslyn |
| ¿Quién implementa X? | Roslyn |
| ¿Qué método llama X? | Roslyn |
| ¿Qué proyecto referencia X? | Roslyn/MSBuild |
| ¿Qué hace esta feature? | LLM |
| ¿Qué términos humanos describen esta lógica? | LLM |
| ¿Qué archivos parecen conceptualmente relacionados? | LLM |
| ¿Qué módulo trata sobre facturación? | LLM |
| ¿Qué test referencia X? | Roslyn |
| ¿Qué endpoint llega a X? | Roslyn + extractor ASP.NET |

Nunca permitir:

```text
LLM: "creo que OrderService implementa IOrderService"
```

si Roslyn puede saberlo exactamente.

## 2.7. Estructura de caché

Propongo:

```text
.ai-cache/
├── metadata.json
│
├── structural/
│   ├── projects.json
│   ├── symbols.json
│   ├── references.json
│   ├── implementations.json
│   ├── endpoints.json
│   ├── di.json
│   ├── ef.json
│   └── tests.json
│
├── semantic/
│   ├── modules.json
│   ├── features.json
│   ├── concepts.json
│   └── keywords.json
│
├── summaries/
│   ├── orders.json
│   ├── auth.json
│   └── billing.json
│
└── index.db
```

Pero `.ai-cache/index.db` sería la interfaz principal.

Los JSON pueden servir como debugging/exportación.

## 2.8. Usaría SQLite primero

No Neo4j.

Tablas:

```text
Files
Symbols
Relations
Modules
Features
Endpoints
Entities
Tests
Keywords
Summaries
Commits
```

Ejemplo:

```text
Symbols
-------
Id
Name
QualifiedName
Kind
FileId
Line

Relations
---------
SourceSymbolId
TargetSymbolId
Type

Type:
calls
implements
references
injects
tests
maps_to
```

Y:

```text
Features
--------
Id
Name
Description
ModuleId
Confidence
```

## 2.9. Relacionar significado con estructura

Éste es el paso decisivo.

El LLM dice:

```text
Feature:
"Order cancellation"

Main symbols:
CancelOrderCommandHandler
OrderService.CancelAsync
```

El builder resuelve esos nombres mediante Roslyn:

```text
CancelOrderCommandHandler
       ↓
SymbolId = 812
```

Y almacena:

```text
Feature 21
    ├─ Symbol 812
    ├─ Symbol 923
    └─ Endpoint 18
```

El conocimiento semántico deja así de ser texto suelto.

## 2.10. Añadir keywords y sinónimos

Muy importante para prompts humanos.

Por ejemplo:

```text
Feature: UserAuthentication

keywords:
login
sign in
authenticate
JWT
access token
session
credentials
```

El desarrollador puede preguntar:

> cambia el login.

Aunque en el código todo se llame:

```text
AuthenticationCommandHandler
JwtTokenFactory
```

El índice conecta ambos mundos.

## 2.11. Índice jerárquico

No devolver directamente archivos.

Resolver:

```text
consulta
 ↓
módulo
 ↓
feature
 ↓
symbols
 ↓
files
```

Por ejemplo:

```text
"cambiar cancelación de pedidos"

        ↓

Module:
Orders

        ↓

Feature:
OrderCancellation

        ↓

Symbols:
CancelOrderCommand
CancelOrderHandler
OrderService.CancelAsync

        ↓

Roslyn expansion

        ↓

7 files
```

## 2.12. MCP único como interfaz del agente

Copilot no debería conocer cómo está construido el índice.

Solo ve:

```text
repo-intelligence MCP
```

Herramientas iniciales:

```text
repo_overview
search_feature
describe_feature
find_symbol
related_code
trace_endpoint
related_tests
impact_map
```

No expondría directamente:

```text
query_sql
read_semantic_json
read_structural_json
```

porque obligaría al modelo a entender detalles internos.

## 2.13. `search_feature`

Entrada:

```text
"order cancellation"
```

Salida:

```json
{
  "feature": "OrderCancellation",
  "confidence": 0.94,
  "description": "...",
  "main_symbols": [
    "CancelOrderCommandHandler",
    "OrderService.CancelAsync"
  ],
  "candidate_files": [
    "...",
    "..."
  ]
}
```

Objetivo: <1.000 tokens.

## 2.14. `related_code`

Entrada:

```text
OrderCancellation
```

Combina:

```text
semantic index
+
Roslyn
```

Resultado:

```text
Primary:
  CancelOrderCommand.cs
  CancelOrderCommandHandler.cs
  OrderService.cs

Dependencies:
  IOrderRepository.cs
  OrderRepository.cs

Entry point:
  OrdersController.cs

Tests:
  CancelOrderCommandHandlerTests.cs
```

## 2.15. `impact_map`

Ésta sería la herramienta estrella del sistema completo.

Entrada:

```text
"Add cancellation reason to cancelled orders"
```

Proceso:

```text
semantic search
      ↓
OrderCancellation feature
      ↓
main symbols
      ↓
Roslyn expansion
      ↓
EF mapping
      ↓
tests
      ↓
endpoint
```

Salida:

```text
Likely affected:

Domain:
  Order.cs

Application:
  CancelOrderCommand.cs
  CancelOrderCommandHandler.cs

Infrastructure:
  OrderConfiguration.cs
  OrderRepository.cs

Api:
  OrdersController.cs

Tests:
  CancelOrderHandlerTests.cs

Confidence: 0.91
```

Ahora Copilot abre 6 archivos.

## 2.16. Verificación obligatoria contra código real

Norma del agente:

```text
MCP output = navigation hint
source code = truth
```

Antes de editar:

```text
index
 ↓
candidate files
 ↓
read actual source
 ↓
verify
 ↓
edit
```

Nunca:

```text
index
 ↓
edit blindly
```

## 2.17. Actualización incremental

No regenerar todo el índice en cada commit.

Guardar:

```json
{
  "commit": "9f7a21c",
  "schemaVersion": 1
}
```

Luego:

```text
git diff oldCommit..HEAD
        ↓
changed files
        ↓
Roslyn re-analysis
        ↓
affected modules/features
        ↓
Repomix only affected scope
        ↓
free LLM
        ↓
update SQLite
```

Ejemplo:

```text
Changed:
  CancelOrderHandler.cs
  Order.cs

          ↓

Affected feature:
  OrderCancellation

          ↓

Regenerate:
  structural nodes affected
  semantic/orders.json

NOT:
  Authentication
  Billing
  Reporting
```

## 2.18. Invalidación mediante hashes

Incluso mejor:

```text
File
path
SHA256
lastIndexedCommit
```

Si:

```text
hash actual == hash indexado
```

no procesar.

## 2.19. Dos niveles de actualización

### Rápida

Cada cambio local:

```text
Roslyn
 ↓
structural cache
```

Barato y determinista.

### Semántica

Por ejemplo:

```text
antes de iniciar sesión importante
al cambiar feature
al hacer commit
manualmente
CI
```

Ejecuta:

```text
Repomix
 ↓
LLM gratis
 ↓
semantic cache
```

No es necesario invocar al LLM después de modificar una coma.

## 2.20. CI opcional

Repomix dispone de integración documentada con GitHub Actions, por lo que la generación puede automatizarse si termina interesándote mantener la caché de forma centralizada.

Posible pipeline:

```text
push main
   ↓
detect changed modules
   ↓
Roslyn index
   ↓
Repomix changed modules
   ↓
semantic model
   ↓
validate JSON
   ↓
build index
```

Pero lo introduciría tarde, no inicialmente.

# 3. Flujo final del agente

Una tarea ideal terminaría funcionando así:

```text
USER
"Permite cancelar una Order indicando motivo"
              │
              ▼
        Copilot Agent
              │
              ▼
     search_feature(...)
              │
              ▼
       OrderCancellation
              │
              ▼
        impact_map(...)
              │
              ▼
     6 candidate files
              │
              ▼
       open real files
              │
              ▼
      verify assumptions
              │
              ▼
         modify 4 files
              │
              ▼
     build Application.csproj
              │
              ▼
      related_tests(...)
              │
              ▼
       execute 7 tests
              │
              ▼
         VS Code Problems
              │
          ┌───┴────┐
          │        │
         OK      ERROR
          │        │
          ▼        ▼
        done     repair
```

# 4. Orden de implementación recomendado

Lo haría exactamente en este orden:

| Fase | Implementación | Complejidad | Retorno esperado |
|---|---|---:|---:|
| 0 | Benchmark inicial | baja | imprescindible |
| 1 | Limpiar estructura repo | variable | alto |
| 2 | `copilot-instructions.md` reducido | baja | alto |
| 3 | `.instructions.md` por capa | baja | alto |
| 4 | `AGENTS.md` locales | baja | medio-alto |
| 5 | scripts build/test | baja | muy alto |
| 6 | tests filtrados | baja | muy alto |
| 7 | prompt files | baja | medio |
| 8 | Agent Skills | baja-media | alto |
| 9 | Analysis/Implementation agents | baja | medio-alto |
| 10 | Tool Sets | baja | medio |
| 11 | Roslyn MCP básico | media | muy alto |
| 12 | referencias/implementaciones/callers | media | muy alto |
| 13 | ASP.NET endpoints + DI | media | alto |
| 14 | EF + tests | media | alto |
| 15 | `impact_analysis` | media | muy alto |
| 16 | Repomix compressed | baja | preparación |
| 17 | LLM → módulos/features JSON | media | alto |
| 18 | SQLite index | media | alto |
| 19 | unir Roslyn + semántica | media-alta | muy alto |
| 20 | MCP `search_feature` | media | muy alto |
| 21 | MCP `related_code` | media | muy alto |
| 22 | MCP `impact_map` híbrido | alta | máximo |
| 23 | actualización incremental | media-alta | imprescindible a escala |
| 24 | CI/cache compartida | media | opcional |

# 5. Primer MVP que realmente merece la pena

No construiría todo antes de probar.

El MVP debería ser:

```text
.github/copilot-instructions.md
.github/instructions/
AGENTS.md

scripts/
   test-filter.ps1
   build-project.ps1

Roslyn MCP:
   find_symbol
   find_references
   find_implementations
   find_callers

Repomix
   ↓
free LLM
   ↓
features.json

MCP:
   search_feature
```

Esto ya permite:

```text
prompt humano
     ↓
search_feature
     ↓
Roslyn expansion
     ↓
archivos candidatos
```

Si esto no mejora significativamente las métricas, no merece la pena construir las siguientes 15 piezas.

# 6. Segunda versión

Después:

```text
resolve_service
list_endpoints
related_tests
```

Más:

```text
SQLite
incremental hashes
```

Y:

```text
semantic module summaries
```

# 7. Tercera versión

Solo entonces:

```text
trace_endpoint
describe_entity
related_code
impact_map
test-changed
incremental semantic refresh
```

Aquí ya tienes una verdadera capa de inteligencia del repositorio.

# 8. Qué NO haría

Evitaría:

```text
❌ meter todo Repomix en cada prompt
❌ crear architecture.md de 100 páginas
❌ usar embeddings para cosas que Roslyn sabe exactamente
❌ dejar que el LLM invente call graphs
❌ devolver archivos completos desde herramientas MCP
❌ exponer 50 tools al agente
❌ ejecutar toda la solution en cada iteración
❌ recalcular semántica de todo el repo con cada cambio
❌ introducir Neo4j antes de demostrar que SQLite no basta
❌ permitir que la cache sea autoridad frente al source
❌ indexar bin/, obj/ o código generado innecesariamente
```

# 9. Presupuesto de contexto objetivo

Un resultado ideal para una tarea mediana sería algo semejante a:

```text
Instructions                  1-2k
Skill específico              1k
Search feature                0.5k
Impact map                    1k
4-8 archivos reales          8-20k
tests relevantes              3-8k
----------------------------------
TOTAL aproximado             14-32k
```

en lugar de:

```text
Repo / Repomix              150-500k+
exploración adicional
resultados grep
archivos descartados
```

Es precisamente ahí donde estaría el ahorro.

# 10. KPI final

Después de implementar todo, repetiría exactamente las tareas del baseline.

Consideraría el proyecto exitoso si aproximadamente consigue:

```text
↓ 60-90 % archivos irrelevantes abiertos

↓ 40-70 % contexto consumido en descubrimiento

↓ 50-90 % coste de validación por iteración

↓ 30-70 % llamadas de búsqueda/navegación

↓ tiempo hasta primera edición correcta

≈ o ↑ porcentaje de tareas correctas

≈ 0 alucinaciones adicionales causadas por cache
```

La métrica particularmente interesante sería:

```text
Relevant files opened
──────────────────────
Total files opened
```

Si inicialmente Copilot abre:

```text
8 relevantes / 30 totales = 27 %
```

y después:

```text
7 relevantes / 9 totales = 78 %
```

has conseguido exactamente lo que pretende esta arquitectura.

# 11. Arquitectura final recomendada

Mi objetivo a largo plazo sería éste:

```text
                          SOURCE CODE
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
    Project rules          Roslyn            Repomix
           │                  │                  │
           │             exact facts         free LLM
           │                  │                  │
           │                  ▼                  ▼
           │           structural graph    semantic map
           │                  │                  │
           │                  └────────┬─────────┘
           │                           │
           │                           ▼
           │                    SQLite index
           │                           │
           │                    version + hashes
           │                           │
           └──────────────┬────────────┘
                          ▼
                   Repo Intelligence MCP
                          │
             ┌────────────┼─────────────┐
             │            │             │
             ▼            ▼             ▼
       search_feature related_code  impact_map
             │            │             │
             └────────────┼─────────────┘
                          ▼
                      COPILOT
                          │
                    real source
                          │
                       editing
                          │
                  minimal build
                          │
                  related tests
                          │
                      Problems
```

El principio rector sería: **Roslyn determina hechos, el modelo gratuito precalcula significado y Copilot utiliza ambos únicamente para seleccionar el mínimo contexto real necesario para resolver la tarea.** Repomix sirve de mecanismo de extracción/transporte hacia el modelo auxiliar, no como contexto permanente del agente principal. Repomix está específicamente diseñado para empaquetar repositorios para procesamiento por LLM y permite comprimir estructuralmente y seleccionar subconjuntos, mientras que Roslyn ofrece las operaciones deterministas de símbolos/referencias que necesitas para que la parte estructural no dependa de inferencias probabilísticas.

En términos de prioridad, yo dedicaría aproximadamente el **primer 40 % del esfuerzo a la Medida 1** y no empezaría la infraestructura sofisticada de Repomix hasta comprobar que `Roslyn + tests selectivos + Skills/agents bien diseñados` ya funciona. El **60 % restante** iría a convertir la segunda medida en una auténtica memoria semántica consultable e incremental. Ahí es donde puedes conseguir que un modelo gratuito haga gran parte del trabajo repetitivo de comprensión y reservar Copilot para razonamiento, edición y validación.
