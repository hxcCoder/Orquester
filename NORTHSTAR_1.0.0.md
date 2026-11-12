NORTHSTAR 1.2.0
Recon Orchestrator — Project Charter, Architecture & Engineering Baseline
Versión: 1.2.0
Fecha: 2026-11-12
Estado: Baseline arquitectónico final, verificado y listo para implementación
Clasificación: Herramienta local para reconocimiento autorizado y reproducible
Documento: Fuente de verdad del diseño; las decisiones aceptadas se registran además como ADR.

Importante: Este documento define una arquitectura y criterios de aceptación. No afirma que exista una implementación funcional ni que las garantías estén verificadas hasta que el código, las pruebas y la evidencia de ejecución las demuestren.

0. Resumen ejecutivo
Northstar es un orquestador CLI modular de reconocimiento de activos para programas de bug bounty con autorización explícita y laboratorios/CTFs. Su propósito es ejecutar workflows declarativos, conectar herramientas mediante contratos tipados, conservar evidencia, mantener historial y detectar cambios verificables.

Northstar no es un wrapper de comandos ni un escáner indiscriminado. Es un motor de workflows con:

configuración declarativa y validación estricta a todos los niveles de anidamiento;

autorización y control de scope antes y durante la ejecución;

planificación determinista de DAG;

ejecución asíncrona con límites, cancelación y limpieza verificable de árboles de procesos;

adapters desacoplados y versionados;

artefactos inmutables y verificables;

persistencia histórica con SQLite y writer único con cola acotada;

cobertura explícita y diff conservador;

pruebas automatizadas y recuperación ante fallos.

Estado de madurez
Nivel	Estado
Arquitectura	Definida, corregida y verificada en este documento
Contratos	Propuestos; deben implementarse
MVP	No declarado completo
Seguridad	No certificada; requiere pruebas y revisión
Producción	No habilitada hasta cumplir DoD y gates
1. Visión, alcance y límites
1.1 Visión
Construir una herramienta local, extensible y reproducible que permita definir reconocimiento autorizado como workflows declarativos, ejecutar herramientas de manera controlada, almacenar evidencia y comparar observaciones históricas sin producir conclusiones falsas.

1.2 Objetivos
Definir módulos y workflows en YAML.

Encadenar outputs mediante artefactos y modelos tipados.

Ejecutar nodos independientes en paralelo con límites explícitos.

Aplicar scope, autorización, concurrencia, rate limits y timeout.

Conservar stdout, stderr, outputs parseados, logs y manifests.

Normalizar resultados en modelos comunes versionados.

Persistir targets, runs, nodos, activos, observaciones y findings en SQLite.

Detectar activos nuevos y cambios con evidencia comparable.

Producir reportes legibles y exportables.

Permitir adapters revisados y extensiones controladas.

Recuperarse de interrupciones sin declarar éxito ficticio.

Ser testeable sin ejecutar herramientas reales ni tráfico de red.

1.3 No objetivos del MVP
Explotación automática de vulnerabilidades.

Escaneo indiscriminado de Internet.

Expansión automática de scope.

Ejecución remota o distribuida.

Dashboard web.

Plugins de terceros sin revisión.

Interpretar ausencia de resultados como cierre, eliminación u offline.

Garantizar protección de red cuando una herramienta externa no permite controlar el destino efectivo de conexión.

Sustituir la autorización del programa por una configuración técnica.

1.4 Uso autorizado
Northstar se ejecutará únicamente sobre activos con autorización vigente y alcance explícito. La autorización debe ser verificable, acotada y registrada. La configuración no crea ni amplía permisos legales o contractuales.

2. Principios de ingeniería
Safe by default: sin autorización y scope válidos, no se ejecuta ningún proceso.

Fail closed: si una validación crítica no puede completarse, se bloquea la operación.

Declarative workflows: YAML describe; Python valida, planifica y ejecuta.

Separation of concerns: CLI, configuración, política, planner, scheduler, executor, adapters, pipeline, storage y reporting son capas separadas.

Reproducibility: cada run registra configuración efectiva, plan, versiones, timestamps, scope y evidencia.

Idempotency: repetir operaciones no duplica identidades ni observaciones.

Bounded concurrency: límites globales, por target y por herramienta.

Explicit states: SUCCESS, DEGRADED, FAILED, SKIPPED, CANCELLED y TIMED_OUT tienen semántica diferenciada.

Evidence first: los datos normalizados apuntan a artefactos originales verificables.

No false negatives: ausencia de resultados no significa cierre, eliminación ni indisponibilidad.

Deterministic validation: errores estructurales y semánticos se detectan antes de iniciar procesos.

Least privilege: no se requieren privilegios elevados; ejecutables y permisos se verifican.

No shell: procesos externos se invocan con argv, nunca con shell=True. asyncio.create_subprocess_exec es la única vía permitida.

Recoverability: un run interrumpido se recupera como incompleto, nunca como exitoso por defecto.

Observability: toda transición, decisión de política y resultado relevante deja registro.

Security boundaries explicit: no se promete pinning o control de redirects si el adapter/herramienta no lo implementa de manera verificable.

3. Arquitectura lógica
text
┌─────────────────────────────────────────────────────────────┐
│ CLI: validate · scope · run · runs · diff · report       │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ CONFIGURATION CORE                                         │
│ YAML loader · Pydantic strict models · effective config   │
│ schema version · defaults · secret references             │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ POLICY / AUTHORIZATION                                     │
│ authorization evidence · allowlist · exclusions · scope    │
│ target validation · network policy · rate limits          │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ WORKFLOW PLANNER                                           │
│ DAG · cycle detection · type compatibility · plan          │
│ deterministic ordering · plan fingerprint                 │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ EXECUTION ENGINE                                           │
│ scheduler · state machine · semaphores · cancellation      │
│ retries · recovery · event journal                         │
└──────────────┬──────────────────────────────┬───────────────┘
               ▼                              ▼
┌──────────────────────────┐  ┌──────────────────────────────┐
│ PROCESS SUPERVISOR      │  │ ARTIFACT / PIPELINE         │
│ spawn · stdout/stderr   │  │ raw · parser · normalization│
│ timeout · process group │  │ schema · coverage            │
│ terminate · kill · wait │  │ checksums · validation       │
└──────────────┬───────────┘  └──────────────┬───────────────┘
               └─────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ PERSISTENCE CORE                                           │
│ SQLite · migrations · transactions · single writer        │
│ targets · runs · nodes · assets · observations · findings │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ ANALYSIS / REPORTING                                       │
│ diff · coverage · findings · JSON/terminal exports         │
└─────────────────────────────────────────────────────────────┘
3.1 Dependencias entre capas
La CLI no implementa lógica de negocio.

El planner no ejecuta procesos.

El scheduler no interpreta outputs de herramientas.

El executor no decide scope ni normaliza activos.

Los adapters no escriben directamente en SQLite.

Los parsers no realizan conexiones de red.

El DiffEngine no modifica observaciones históricas.

Reporting no cambia el estado del run.

4. Estructura del repositorio
text
northstar/
├── pyproject.toml
├── README.md
├── NORTHSTAR.md
├── northstar/
│   ├── __init__.py
│   ├── cli.py
│   ├── app.py
│   ├── errors.py
│   ├── logging.py
│   ├── config/
│   │   ├── loader.py
│   │   ├── models.py
│   │   ├── templates.py
│   │   └── secrets.py
│   ├── policy/
│   │   ├── authorization.py
│   │   ├── scope.py
│   │   ├── targets.py
│   │   ├── network.py
│   │   └── rate_limit.py
│   ├── workflow/
│   │   ├── models.py
│   │   ├── planner.py
│   │   ├── scheduler.py
│   │   ├── state_machine.py
│   │   ├── events.py
│   │   └── recovery.py
│   ├── runtime/
│   │   ├── executor.py
│   │   ├── process.py
│   │   ├── cancellation.py
│   │   └── pipeline.py
│   ├── adapters/
│   │   ├── base.py
│   │   ├── registry.py
│   │   ├── subfinder.py
│   │   ├── httpx.py
│   │   ├── nmap.py
│   │   └── resolver.py
│   ├── artifacts/
│   │   ├── models.py
│   │   ├── store.py
│   │   ├── integrity.py
│   │   └── parsers/
│   │       ├── lines.py
│   │       ├── httpx_jsonl.py
│   │       └── nmap_xml.py
│   ├── domain/
│   │   ├── assets.py
│   │   ├── observations.py
│   │   ├── coverage.py
│   │   └── findings.py
│   ├── storage/
│   │   ├── database.py
│   │   ├── migrations/
│   │   ├── repositories.py
│   │   └── writer.py
│   ├── diff/
│   │   ├── engine.py
│   │   └── compatibility.py
│   └── reporting/
│       ├── reports.py
│       └── exporters.py
├── configs/
│   ├── modules.yaml
│   ├── workflows.yaml
│   └── scope.example.yaml
├── data/
│   └── .gitkeep
├── runs/
│   └── .gitkeep
├── reports/
│   └── .gitkeep
├── docs/
│   ├── architecture.md
│   ├── security.md
│   ├── testing.md
│   └── adr/
│       ├── ADR-001-buffered-pipeline.md
│       ├── ADR-002-strict-config.md
│       ├── ADR-003-sqlite-writer.md
│       └── ...
└── tests/
    ├── unit/
    ├── integration/
    ├── contract/
    ├── security/
    ├── fault/
    └── fixtures/
Los directorios data/, runs/, reports/, secretos y scopes reales deben excluirse del control de versiones. Se versionan ejemplos sintéticos, no evidencia sensible.

5. Configuración declarativa
5.1 Reglas del loader
YAML se carga con un parser seguro; no se permiten tags arbitrarios.

Pydantic v2 valida modelos con modo estricto.

extra="forbid" en modelos de configuración.

La estrictitud no es recursiva por defecto en Pydantic v2. extra="forbid" y strict=True deben declararse explícitamente en cada modelo anidado (ConfigDict(extra="forbid", strict=True)). El loader debe recorrer la jerarquía de modelos y fallar si algún BaseModel de configuración carece de estas directivas. Un test de CI es obligatorio.

No se permite eval, ejecución de templates arbitrarios ni interpolación de shell.

Templates se resuelven mediante un motor restringido y variables tipadas.

Los defaults se materializan en la configuración efectiva.

La configuración efectiva se guarda sin secretos.

Toda configuración tiene versión de schema.

La validación estructural precede a la validación semántica.

validate no inicia herramientas ni tráfico de reconocimiento.

5.2 configs/modules.yaml — ejemplo conceptual
yaml
version: 1

modules:
  subfinder:
    module_kind: tool
    adapter: subfinder
    executable: subfinder
    executable_resolution: which_verified
    argv:
      - "-d"
      - "{{ params.domain }}"
      - "-silent"
    input:
      kind: none
    output:
      kind: lines
      parser: subfinder_lines
      parser_schema_version: 1
      artifact_type: discovered_hostname
    timeout_seconds: 300
    retry:
      max_attempts: 1

  httpx:
    module_kind: tool
    adapter: httpx
    executable: httpx
    executable_resolution: which_verified
    argv:
      - "-silent"
      - "-json"
      - "-status-code"
      - "-title"
    input:
      kind: stdin
      format: lines
      accepts_artifact_types:
        - authorized_hostname
    output:
      kind: jsonl
      parser: httpx_jsonl
      parser_schema_version: 1
      artifact_type: http_service_observation
    timeout_seconds: 300
    retry:
      max_attempts: 1

  nmap:
    module_kind: tool
    adapter: nmap
    executable: nmap
    executable_resolution: which_verified
    argv:
      - "-Pn"
      - "-sT"
      - "-p"
      - "80,443,8080,8443"
      - "-oX"
      - "-"
      - "{{ params.target }}"
    input:
      kind: none
    output:
      kind: xml
      parser: nmap_xml
      parser_schema_version: 1
      artifact_type: port_observation
    timeout_seconds: 600
    retry:
      max_attempts: 1

  scope_validate:
    module_kind: policy
    policy: scope_validate_v1
    input:
      kind: artifact
      accepts_artifact_types:
        - discovered_hostname
    output:
      kind: structured
      schema: authorized_hostname_v1
      artifact_type: authorized_hostname
    on_out_of_scope: mark
module_kind: tool indica que el nodo spawnea un proceso. module_kind: policy indica que el nodo ejecuta lógica pura del motor (ver §6.6). Esta distinción es estructural y afecta cómo el planner valida, cómo el scheduler ejecuta y cómo se registra el NodeExecution.

Este YAML es un ejemplo de contrato, no una configuración universal lista para ejecutar. Cada adapter debe comprobar que los argumentos son compatibles con la versión instalada y con la política de scope.

5.3 configs/workflows.yaml — ejemplo conceptual
yaml
version: 1

workflows:
  recon:
    description: Reconocimiento inicial de scope autorizado
    nodes:
      - id: discover
        module: subfinder
        params:
          domain: "{{ target.domain }}"

      - id: validate_targets
        module: scope_validate
        depends_on: [discover]
        input_from: discover

      - id: probe
        module: httpx
        depends_on: [validate_targets]
        input_from: validate_targets

    policy:
      max_parallel: 3
      per_target_max_parallel: 1
      per_tool_max_parallel: 2
      on_node_failure: abort-drain
Reglas:

depends_on define orden y condición de dependencia.

input_from define transporte de datos.

Todo input_from debe ser compatible con el tipo de output del productor.

Las referencias deben existir.

Los IDs deben ser únicos.

No se permiten ciclos.

Los parámetros se validan según el adapter o la política.

El planner resuelve templates y crea un plan inmutable.

El scope se valida antes del run y antes de cada operación relevante.

5.4 Semántica de templates
Solo variables declaradas y tipadas.

No se permite acceso a atributos arbitrarios.

No se permite evaluación de expresiones.

Los valores externos se validan antes de entrar en argv.

No se interpolan secretos en argumentos salvo que el adapter lo requiera explícitamente y exista un mecanismo seguro.

Los valores que comiencen con - no se insertan en argumentos posicionales sin validación y protección específica.

Se usa -- cuando la herramienta lo soporta y el contrato lo permite.

PEP 787 propone t-strings para subprocess y shlex con objetivo en Python 3.15. No se adopta en el MVP. Cuando la versión mínima soportada lo permita, se evaluará su adopción sin abandonar la lista de argumentos como mecanismo primario.

5.5 Namespaces y orden de resolución
Los templates operan sobre tres namespaces explícitos y mutuamente excluyentes. Un template que referencia un namespace no declarado falla en validación, no en ejecución.

Namespace	Origen	Disponible en	Ejemplo
target.*	El target activo del run, validado por scope	Todos los nodos	{{ target.domain }}
params.*	Los parámetros declarados en el nodo o en el módulo, ya validados por el adapter	Todos los nodos	{{ params.domain }}
env.*	Variables de entorno allowlisted en la política de entorno, no secretos	Todos los nodos	{{ env.HTTP_PROXY }}
Reglas de resolución:

El planner resuelve params.* primero, en el orden en que aparecen en el YAML; si un parámetro referencia otro parámetro, el planner resuelve en orden topológico y detecta ciclos.

El planner resuelve target.* a partir del target del run; los targets derivados no son visibles como target.* en este MVP, solo como artefactos de entrada (input_from).

El planner resuelve env.* solo para variables allowlisted; una variable no allowlisted en env.* falla en validación.

Los secretos nunca se resuelven vía template; se inyectan por el canal seguro del adapter (§8.6) y se referencian con secret://nombre, que el adapter resuelve en el último momento y no aparece en argv ni en logs.

La resolución de templates es determinista y se registra en el ExecutionPlan.effective_params sin secretos.

6. Contrato de módulos, adapters y artefactos
6.1 ModuleSpec
text
ModuleSpec
├── id
├── module_kind            # tool | policy
├── adapter_id             # solo si module_kind == tool
├── policy_id              # solo si module_kind == policy
├── executable             # solo si module_kind == tool
├── executable_resolution  # solo si module_kind == tool
├── argv_template          # solo si module_kind == tool
├── input_spec
├── output_spec
├── timeout                # solo si module_kind == tool
├── retry_policy           # solo si module_kind == tool
├── environment_policy     # solo si module_kind == tool
└── adapter_schema_version
La distinción tool vs policy es estructural, no cosmética. Un ModuleSpec con module_kind: tool que carezca de executable, argv_template o timeout falla en validación. Un ModuleSpec con module_kind: policy que declare executable o argv_template falla en validación.

6.2 Adapter contract
Cada adapter debe:

Declarar tipos de input y output.

Validar parámetros antes del spawn.

Verificar versión y capacidades de herramienta.

Construir argv sin shell.

Aplicar la política de autorización pertinente.

Ejecutar únicamente targets validados.

Parsear de forma determinista.

Reportar errores y cobertura.

No escribir directamente en la DB.

No ocultar outputs parciales.

No reinterpretar errores como ausencia de resultados.

Tener fixtures y tests de contrato.

Declarar explícitamente su modelo de control de red (pinning, resolución, redirects) y sus limitaciones, conforme a §8.3.1.

6.3 NodeExecution
text
NodeExecution
├── run_id
├── node_id
├── module_kind            # tool | policy
├── status
├── started_at
├── ended_at
├── exit_code              # solo si module_kind == tool
├── termination_signal     # solo si module_kind == tool
├── stdout_artifact_id     # solo si module_kind == tool
├── stderr_artifact_id     # solo si module_kind == tool
├── parsed_artifact_id
├── coverage_id
├── tool_version           # solo si module_kind == tool
├── adapter_version
├── policy_version         # solo si module_kind == policy
├── error_code
└── error_message_redacted
Un nodo policy no tiene exit_code ni termination_signal; su estado se deriva de la lógica de política (por ejemplo, si scope_validate no pudo evaluar un host por error interno, el nodo queda FAILED, no SKIPPED).

6.4 Artifact
text
Artifact
├── id
├── run_id
├── node_id
├── kind
├── format
├── schema_version
├── relative_path
├── checksum_sha256
├── size_bytes
├── created_at
└── integrity_status        # verified | incomplete | orphaned | quarantined
Los paths persistidos deben ser relativos a una raíz de artefactos controlada. Se rechazan traversal, symlinks inesperados y escrituras fuera de esa raíz.

6.5 Transporte
MVP:

Buffered.

Artifact-based.

Stdin mediante archivos/streams controlados cuando corresponda.

Streaming se pospone hasta implementar backpressure, límites de memoria, manejo de stderr, cancelación, cleanup y pruebas de presión.

6.6 Policy nodes
Un policy node es un nodo de workflow que ejecuta lógica del motor sin spawnear procesos externos ni realizar conexiones de red. Su función es transformar artefactos aplicando reglas de política verificables.

Contrato de un policy node:

Declara module_kind: policy y un policy_id versionado (por ejemplo, scope_validate_v1).

Recibe un artefacto de entrada (input_from) y produce un artefacto de salida con schema versionado.

Tiene NodeExecution con los mismos estados que un nodo tool (SUCCESS, DEGRADED, FAILED, CANCELLED), pero sin exit_code ni señales.

Puede producir un Coverage record si su transformación afecta la interpretabilidad de runs posteriores (por ejemplo, scope_validate debe registrar cuántos hosts entrantes fueron autorizados, descartados o fallados).

No escribe en SQLite; su salida es un artefacto que el pipeline consume.

Es determinista: mismo input, misma versión de política, mismo output.

Falla en cerrado: si no puede evaluar un ítem, el nodo queda FAILED o DEGRADED según contrato, nunca SUCCESS con ítems sin evaluar.

Se testea con unit tests de la función pura y con tests de contrato del artefacto de salida, sin necesidad de mocks de proceso.

El primer policy node del MVP es scope_validate. Otros policy nodes previstos en el roadmap (fuera del MVP): dedupe_assets, filter_by_allowlist, canonicalize_hostnames.

7. Workflow planner
El planner produce un ExecutionPlan determinista.

7.1 Validaciones
Schema de workflow.

IDs únicos.

Referencias válidas.

Ciclos.

Compatibilidad de tipos.

Parámetros por adapter o política.

module_kind coherente con los campos declarados (§6.1).

Scope y autorización.

Ejecutables disponibles y verificables (solo para module_kind: tool).

Capacidades de herramientas.

Límites de concurrencia.

Templates resolubles en los namespaces declarados (§5.5).

Rutas de artefactos válidas.

Configuración de retry y timeout.

Compatibilidad de schemas.

Política on_node_failure válida (§10.6).

7.2 ExecutionPlan
text
ExecutionPlan
├── plan_version
├── workflow_id
├── ordered_nodes
├── dependency_graph
├── effective_params
├── resolved_executables
├── adapter_versions
├── tool_versions
├── policy_versions
├── scope_hash
├── authorization_hash
├── policy_snapshot
└── plan_fingerprint
El plan se serializa de forma canónica y se hashea. Un run utiliza un plan inmutable; cualquier cambio requiere un nuevo plan/run.

8. ScopeGuard y autorización
8.1 Autorización vinculante
El scope debe incluir:

yaml
version: 1
authorization:
  authorization_ref: "evidence/authorization-redacted.pdf"
  authorization_sha256: "<sha256>"
  authorized_by: "<responsible-person-or-program>"
  valid_from: "2026-11-01T00:00:00Z"
  valid_until: "2026-12-01T00:00:00Z"
  program_reference: "<program-id-or-url>"

scope:
  domains:
    - example.com
  subdomains:
      policy: explicit
  cidrs: []
  exclusions:
    domains: []
    cidrs: []
Los valores son ficticios. La evidencia real debe mantenerse privada y fuera del repositorio.

validate y run fallan en duro si la autorización está ausente, inválida, expirada o no coincide con el scope.

8.2 Política de targets
Allowlist explícita de dominios, subdominios y CIDRs.

Exclusiones con precedencia sobre inclusiones.

Wildcards acotados y semántica inequívoca.

No se asume que autorizar un dominio autoriza todas sus IPs.

No se asume que autorizar una IP autoriza todos los servicios alojados allí.

Targets derivados se validan de nuevo.

No se amplía scope automáticamente.

Se registran las decisiones de autorización.

8.3 DNS y pinning
Una validación DNS seguida de una resolución independiente por una herramienta externa no garantiza pinning. OWASP es explícito: validar los nombres no detiene el DNS rebinding; hay que validar las IPs resueltas y conectar solo a esas IPs validadas, preservando el hostname para Host, SNI y verificación de certificado.

El contrato debe distinguir:

hostname autorizado;

resolución observada;

conjunto de IPs validado;

destino de conexión efectivo;

identidad TLS/HTTP preservada;

evidencia de que el adapter utilizó el destino validado.

Cuando una herramienta no permita controlar o verificar el destino efectivo, Northstar debe:

marcar el control como no garantizado;

limitar el uso según la política;

no afirmar protección completa contra DNS rebinding;

preferir adapters que permitan resolver, fijar y conectar de forma controlada.

8.3.1 Protocolo de verificación de red del adapter
Para que la garantía de “destino validado” sea verificable, cada adapter tool que realice conexiones de red debe producir un NetworkEvidenceRecord por cada request u operación relevante, adjunto a la observación y con su hash agregado en el Coverage record del nodo:

text
NetworkEvidenceRecord
├── authorized_hostname
├── resolved_ips              # set canónico de IPs observadas en la resolución controlada por el adapter
├── connected_ip              # IP efectiva a la que se conectó la herramienta
├── tls_sni                   # SNI usado, si aplica
├── tls_peer_ip               # IP del peer TLS observado en el handshake, si aplica
├── redirect_chain            # lista ordenada de (scheme, hostname, port, ip) por cada redirect seguido
├── connection_outcome        # connected | refused | timeout | tls_error | blocked_by_policy
└── evidence_hash             # SHA-256 de la serialización canónica del record
Reglas:

Si el adapter no puede obtener connected_ip o tls_peer_ip porque la herramienta no lo expone, debe declarar connected_ip: unavailable y el nodo correspondiente se marca DEGRADED por pérdida de garantía de pinning, no SUCCESS.

Si connected_ip no está en resolved_ips, la observación se marca con pinning_status: mismatch y el nodo se trata como FAILED para fines de confianza; no se consume como input de nodos posteriores.

Si redirect_chain contiene un destino fuera de scope, la observación se marca con redirect_status: out_of_scope y el nodo se marca FAILED con código REDIRECT_OUT_OF_SCOPE.

El evidence_hash agregado del nodo entra en el Coverage record; un run con pinning_status distinto de verified no puede producir MISSING en el DiffEngine.

Los adapters que no realizan conexiones de red (por ejemplo, resolver puro que solo consulta DNS sin conectar a los targets) declaran network_evidence: not_applicable y no producen este record, pero sí registran la resolución observada.

Este protocolo convierte la promesa de pinning en una garantía verificable por test: los fixtures incluyen casos de mismatch y out_of_scope y verifican la clasificación esperada.

8.4 Redirects
Cada redirect debe validar:

esquema;

hostname;

puerto;

IP efectiva;

scope;

exclusiones;

límites de redirect.

Si el adapter no puede validar el destino de cada redirect, debe deshabilitarlos o documentar explícitamente la limitación. El registro de cada redirect se materializa en NetworkEvidenceRecord.redirect_chain (§8.3.1).

8.5 Rate limits
Aplicar:

límite global;

límite por target;

límite por herramienta;

semáforos por target;

token bucket cuando corresponda;

límites conservadores por defecto;

cancelación y backoff configurables.

El límite de tasa debe corresponder a la unidad real de la herramienta (request, conexión, proceso o target); no se debe presentar un límite de procesos como si fuera un límite de requests.

8.6 Secretos
Nunca en YAML versionado, logs, manifests ni argv por defecto.

Resolución desde variables NORTHSTAR_SECRET_* o archivos con permisos restrictivos.

Redacción de valores conocidos en logs y errores.

No registrar el contenido de secretos.

Evitar heredar el entorno completo del usuario; construir un entorno permitido.

La redacción no debe depender solo de coincidencias de texto: debe minimizarse la exposición desde el origen.

Los secretos se referencian en templates como secret://nombre; el adapter resuelve el valor en el último momento y lo inyecta por el canal que la herramienta soporte (stdin, archivo temporal con permisos restrictivos, o variable de entorno allowlisted), nunca por argv por defecto.

9. Executor y Process Supervisor
9.1 Requisitos
asyncio.create_subprocess_exec(*argv).

Nunca shell=True. create_subprocess_shell es vulnerable a inyección de comandos; create_subprocess_exec recibe una lista de argumentos y evita la interpretación por el shell.

Ejecutable absoluto verificado antes del spawn.

Captura concurrente de stdout y stderr.

Timeout por nodo y opcionalmente por run.

start_new_session=True en POSIX.

Process group para cleanup.

Secuencia de terminación: SIGTERM → grace period → SIGKILL del grupo → wait.

Cancelación propagada correctamente.

Espera efectiva del proceso.

Registro de exit code, señal y motivo de terminación.

Entorno controlado.

Límites de output y política de disco.

No dejar procesos descendientes vivos.

No considerar éxito solo por stdout existente.

9.2 Semántica de resultado
Exit code 0 + output parseable + cobertura completa: candidato a SUCCESS.

Exit code 0 + warnings o cobertura parcial: DEGRADED.

Exit code distinto de 0: FAILED, salvo clasificación específica y documentada.

Timeout: TIMED_OUT.

Cancelación solicitada: CANCELLED.

Parser fallido: FAILED o DEGRADED según contrato; nunca SUCCESS silencioso.

Persistencia fallida: run no puede finalizar como SUCCESS.

Output truncado: artefacto marcado incompleto; no se consume como completo.

La presencia de stderr por sí sola no determina degradación: algunas herramientas escriben información normal en stderr. Cada adapter define cómo interpretar warnings, exit codes y cobertura.

9.3 Protección de recursos
Límites de bytes por stdout/stderr.

Cuotas o límites de espacio para runs.

Escritura incremental a archivos cuando el output sea grande.

Lectura concurrente para evitar pipe deadlock.

Backpressure en pipelines.

Cleanup de temporales.

Registro de errores de cleanup.

Si no se puede confirmar terminación de procesos, el run queda en estado de recuperación/inseguro y no se declara finalizado correctamente.

9.4 Plataforma
POSIX: process groups y os.killpg.

La verificación de cleanup no puede limitarse al proceso líder. El test debe comprobar que os.killpg(pgid, 0) falla con ProcessLookupError después del cleanup; si no falla, el run no puede marcarse SUCCESS.

Windows: estrategia documentada con grupos de procesos y fallback taskkill /T; debe probarse en Windows real.

La implementación no debe afirmar equivalencia perfecta entre plataformas sin tests específicos.

Si el MVP es POSIX-only, debe declararse explícitamente en README.md y en los classifiers de pyproject.toml.

10. Scheduler y máquina de estados
10.1 Estados
text
PENDING → READY → RUNNING
                    ├── SUCCESS
                    ├── DEGRADED
                    ├── FAILED
                    ├── TIMED_OUT
                    └── CANCELLED

PENDING/READY → SKIPPED
10.2 Transiciones permitidas
PENDING → READY: dependencias satisfechas y política aprobada.

PENDING → SKIPPED: dependencia fallida o política de aborto.

READY → RUNNING: recursos adquiridos y autorización revalidada.

READY → CANCELLED: cancelación antes del spawn.

RUNNING → SUCCESS: proceso, parser, artefactos y persistencia cumplen contrato.

RUNNING → DEGRADED: output útil pero cobertura/calidad incompleta.

RUNNING → FAILED: error no recuperable.

RUNNING → TIMED_OUT: deadline excedido.

RUNNING → CANCELLED: cancelación completada y cleanup confirmado.

No se permiten transiciones terminales arbitrarias. Los cambios de estado se persisten y registran como eventos.

10.3 Dependencias
SUCCESS habilita dependencias normalmente.

DEGRADED solo habilita consumidores si el workflow declara una política explícita y el consumidor acepta input parcial.

FAILED, TIMED_OUT y CANCELLED bloquean dependencias por defecto.

SKIPPED propaga skip salvo política explícita.

Ningún output parcial se presenta como completo.

10.4 Concurrencia
max_parallel: nodos activos globales.

per_target_max_parallel: operaciones simultáneas por target.

per_tool_max_parallel: procesos simultáneos por herramienta.

Rate limits separados de concurrencia.

Semáforos liberados en finally.

No se inicia un nodo si no se pueden adquirir todos los recursos requeridos.

Evitar deadlocks mediante orden estable de adquisición o un coordinador central.

10.5 Retry
Retry desactivado por defecto para operaciones de reconocimiento salvo configuración específica.

Solo errores transitorios elegibles.

No retry ante scope violation, configuración inválida o autorización ausente.

Backoff exponencial con jitter y máximo total.

Cada intento se registra.

El retry no borra ni sobrescribe evidencia anterior.

El run conserva la relación entre intento y nodo lógico.

10.6 Política on_node_failure
La política on_node_failure se declara en el workflow y admite tres valores mutuamente excluyentes. El planner rechaza cualquier otro valor.

Valor	Semántica
abort-drain (default)	No se lanzan nuevos nodos. Los nodos en RUNNING se dejan terminar naturalmente o por su propio timeout. Los nodos en PENDING/READY pasan a SKIPPED. Al finalizar el último RUNNING, el run termina con el peor estado observado.
abort-cancel	No se lanzan nuevos nodos. Los nodos en RUNNING reciben cancelación inmediata (SIGTERM → gracia → SIGKILL según §9.1). Los nodos en PENDING/READY pasan a SKIPPED. El run termina con CANCELLED si la cancelación se completó y cleanup se verificó.
abort-subtree	No se lanzan nuevos nodos. Solo se cancelan los nodos RUNNING que son descendientes (transitivos) del nodo fallido. Los RUNNING no descendientes terminan naturalmente. Los PENDING/READY descendientes pasan a SKIPPED; los no descendientes continúan.
Reglas adicionales:

Un run con abort-cancel o abort-subtree que no logre verificar cleanup de algún proceso pasa a estado de recuperación (§9.3) y no puede declararse SUCCESS ni CANCELLED limpio.

La política se registra en el ExecutionPlan.policy_snapshot y se persiste en el run para que recover pueda reproducir la decisión.

El DiffEngine no infiere MISSING de un run abortado: la cobertura incompleta por aborto produce UNKNOWN (§13.4).

11. Persistencia SQLite
11.1 Tablas
schema_migrations

targets

runs

run_events

nodes

node_attempts

artifacts

assets

observations

coverage

findings

orphan_artifacts

11.2 Relaciones e invariantes
assets deduplica por (target_id, asset_type, canonical_value).

observations deduplica por (run_id, asset_id, observation_key) según contrato.

nodes tiene unicidad (run_id, node_id).

node_attempts conserva cada intento.

artifacts referencian run y nodo.

findings referencian observaciones y evidencia.

orphan_artifacts registra artefactos encontrados en filesystem sin referencia en DB, para reconciliación.

Foreign keys activadas en cada conexión.

Constraints e índices para invariantes críticas.

Migraciones versionadas, transaccionales cuando sea posible.

first_seen y last_seen en UTC.

Escrituras por lotes.

11.3 SQLite
Configuración inicial recomendada:

sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;
La configuración debe verificarse al abrir cada conexión. synchronous=NORMAL es una decisión de rendimiento/durabilidad: se documenta y se permite cambiar a FULL cuando la prioridad sea mayor durabilidad frente a cortes de energía.

El busy_timeout debe estar presente en cada conexión, no solo en la del writer. Aunque el writer único serializa las escrituras en proceso, los lectores concurrentes y cualquier acceso externo a la DB siguen sujetos al lock de SQLite.

11.4 Single writer
Una corrutina writer consume una asyncio.Queue(maxsize=N).

Operaciones agrupadas en transacciones.

Lectores usan conexiones separadas.

El writer informa errores al productor y al run.

La cola tiene capacidad definida o política de backpressure.

No se descartan operaciones silenciosamente.

Shutdown drena o cancela de manera explícita.

Un fallo permanente del writer detiene la finalización exitosa del run.

Test de saturación obligatorio: llenar la cola más allá de maxsize y verificar que el productor recibe backpressure o un error controlado, nunca una pérdida silenciosa.

11.5 Consistencia DB/Filesystem
No existe una transacción atómica única entre SQLite y filesystem.

Estrategia de escritura:

escribir artefacto temporal en <artifact_root>/tmp/;

cerrar y calcular checksum;

validar tamaño/integridad;

mover atómicamente al destino final dentro del mismo filesystem;

persistir referencia en DB;

reconciliar temporales/orfandad en recuperación.

Protocolo de reconciliación (northstar recover):

recover ejecuta una reconciliación determinista y no destructiva por defecto. Los pasos son:

Escanear <artifact_root> y detectar archivos sin fila correspondiente en artifacts. Cada uno se registra en orphan_artifacts con (relative_path, checksum_sha256, size_bytes, detected_at, reason).

Detectar referencias sin archivo: filas en artifacts cuyo relative_path no existe en filesystem. Se marcan con integrity_status: incomplete y se registra un evento ARTIFACT_MISSING.

Cuarentena de huérfanos: los archivos en tmp/ o con nombre que no cumpla el patrón de artefacto final se mueven a <artifact_root>/quarantine/<run_id>/ conservando checksum; su registro en orphan_artifacts se actualiza con state: quarantined.

No eliminar por defecto. recover nunca borra evidencia. La eliminación de cuarentena requiere el flag explícito northstar recover --purge-quarantine, que exige confirmación interactiva y deja registro en run_events.

Marcar runs incompletos: un run cuyo nodes tenga algún nodo en RUNNING sin ended_at, o cuyo runs.status no sea terminal, se marca como INCOMPLETE y se registra RECOVERY_REQUIRED. El exit code correspondiente es 9 (§14.2).

Idempotencia: ejecutar recover dos veces seguidas no produce cambios en la segunda ejecución; los huérfanos ya cuarentenados no se mueven de nuevo.

La recuperación detecta artefactos sin referencia y referencias sin archivo. Nunca elimina evidencia automáticamente sin una política explícita.

12. Identidad y normalización de activos
12.1 Canonicalización
La canonicalización debe ser determinista, versionada y testeada.

Tipo	Identidad canónica inicial
Subdomain/hostname	lowercase, IDNA, sin punto final
HTTP service	scheme + hostname + port normalizados
Port	IP + protocolo + puerto
DNS record	hostname + tipo + valor normalizado
IP	representación canónica IPv4/IPv6
No se deben eliminar atributos útiles de la observación. Identidad y atributos observados son conceptos separados.

12.2 Modelo
text
NormalizedAsset
├── asset_type
├── canonical_value
├── attributes
├── normalization_version
└── evidence_artifact_id

Observation
├── asset_id
├── run_id
├── node_id
├── observed_at
├── attributes
├── coverage_id
└── evidence_artifact_id
Los parsers deben tolerar campos opcionales conocidos, rechazar estructuras incompatibles y registrar los errores de forma explícita.

12.3 Contrato del policy node scope_validate
scope_validate es el primer policy node del MVP. Su contrato es determinista y verificable por tests puros, sin mocks de proceso.

Input: artefacto discovered_hostname (uno o más hostnames descubiertos por un adapter tool).

Output: artefacto authorized_hostname con el siguiente schema versionado:

text
AuthorizedHostnameRecord
├── hostname                   # canonicalizado según §12.1
├── authorization_decision     # authorized | excluded | out_of_scope | invalid
├── matched_rule               # regla de scope que aplicó (para auditoría)
├── matched_rule_type          # domain | subdomain | cidr | exclusion
├── decision_at                # timestamp UTC
├── canonicalization_version
└── scope_hash                 # hash del scope vigente al momento de la decisión
Comportamiento por decisión:

Decisión	Efecto en el artefacto de salida	Efecto en el nodo
authorized	Se incluye en authorized_hostname y puede consumirse por nodos posteriores.	Contribuye a processed_targets.
excluded	No se incluye en el output consumible. Se registra en el artefacto completo con fines de auditoría.	Contribuye a omitted_targets con reason: excluded.
out_of_scope	No se incluye en el output consumible. Se registra en el artefacto completo.	Contribuye a omitted_targets con reason: out_of_scope.
invalid	No se incluye en el output consumible. Se registra en el artefacto completo.	Contribuye a failed_targets con reason: invalid_hostname.
Reglas:

El nodo produce siempre dos artefactos: el artefacto consumible (authorized_hostname, solo con authorized) y el artefacto de auditoría (scope_validation_log, con todos los registros y sus decisiones).

El Coverage record del nodo cumple processed_targets + omitted_targets + failed_targets == intended_targets (§13.1).

Si intended_targets == 0 (el productor no encontró hosts), el nodo es SUCCESS con completeness: complete_empty; el DiffEngine no puede inferir MISSING de un nodo con complete_empty sin cruzar con el run base.

Si el nodo no puede evaluar un host por error interno (por ejemplo, scope malformado), el nodo es FAILED con código SCOPE_VALIDATION_ERROR, y ningún host del input entra al output consumible.

La política on_out_of_scope es configurable (mark default, drop, fail). mark registra y continúa; drop descarta silenciosamente (no recomendado, pero permitido con registro); fail hace que el nodo sea FAILED si encuentra algún out_of_scope, forzando revisión humana.

El scope_hash se incluye en cada AuthorizedHostnameRecord para que la trazabilidad autorización → observación sea verificable a posteriori.

13. Cobertura y Diff
13.1 Coverage record
text
Coverage
├── operation
├── intended_targets
├── processed_targets
├── omitted_targets
├── failed_targets
├── scan_profile
├── tool_version
├── adapter_version
├── parser_schema_version
├── completeness              # complete | complete_empty | partial | failed
├── warnings
├── errors
├── network_evidence_hash     # hash agregado de NetworkEvidenceRecord (§8.3.1)
└── coverage_fingerprint
La representación de targets debe ser canónica y ordenada antes del hash. Los campos relevantes deben quedar definidos por operación; no se debe usar un único fingerprint genérico para todas las herramientas.

Invariante de completitud verificable: antes de permitir cualquier conclusión que dependa de cobertura, el sistema debe comprobar que processed_targets + omitted_targets + failed_targets == intended_targets. Si la suma no cuadra, completeness no puede ser complete ni complete_empty.

13.2 Fingerprint
SHA-256 de una serialización canónica versionada que incluya:

operación;

targets previstos y perfil;

parámetros que afectan cobertura;

versión de herramienta y adapter;

política de resolución;

exclusiones relevantes;

schema de parser/normalización;

network_evidence_hash.

El fingerprint ayuda a comparar, pero no prueba por sí mismo que la ejecución haya sido completa.

13.3 Categorías del Diff
NEW: primera observación válida.

EXISTING: observado previamente y nuevamente.

CHANGED: atributos comparables cambiaron.

MISSING: no observado en un run con cobertura suficiente y equivalente.

UNKNOWN: cobertura insuficiente, incompatible o incierta.

RESOLVED: solo mediante evidencia y política explícita de resolución.

13.4 Condiciones para MISSING
Solo se permite MISSING cuando:

el run y nodo productor terminaron en SUCCESS;

parser y artefactos son íntegros;

la cobertura está completa (processed_targets + omitted_targets + failed_targets == intended_targets);

el target estaba previsto y fue efectivamente procesado;

el perfil es comparable con el run base;

fingerprint y reglas de compatibilidad coinciden;

no existen omisiones o errores que invaliden esa conclusión;

el pinning_status del nodo productor es verified (para adapters que producen NetworkEvidenceRecord);

el run no fue abortado por on_node_failure (§10.6).

En cualquier otro caso, UNKNOWN.

La ausencia en un output nunca significa automáticamente closed, deleted o offline.

13.5 Puertos
text
PortObservation
├── target
├── ip
├── protocol
├── port
├── state
├── scan_profile
├── observed_at
└── evidence_artifact_id
Estados:

open

closed

filtered

open|filtered

not_scanned

unknown

error

No inferir cierre por ausencia de un puerto en el output.

14. CLI y códigos de salida
14.1 Comandos
text
northstar validate
northstar scope check --target example.com
northstar run recon --target example.com
northstar runs --target example.com
northstar diff --target example.com
northstar report --run <run-id>
northstar assets --target example.com --type subdomain
northstar artifacts --run <run-id>
northstar inspect run <run-id>
northstar verify artifacts --run <run-id>
northstar recover --run <run-id>
Los comandos de ejemplo usan dominios ficticios. run exige autorización válida.

14.2 Exit codes
Código	Significado
0	Operación completada según contrato
1	Error de configuración/validación
2	Scope o autorización inválida
3	Uno o más nodos fallaron
4	Persistencia o integridad
5	Cancelación
6	Ejecutable no disponible/no verificable
7	Error interno no clasificado
8	Run incompleto/degradado cuando la política lo exige
9	Estado de recuperación pendiente: el run quedó inconsistente y requiere northstar recover
El código 0 no debe ocultar errores de nodos ni cobertura degradada. La semántica exacta de DEGRADED debe definirse por comando y documentarse. Un run que termina con procesos sin cleanup verificado, o con la DB en estado inconsistente respecto al filesystem, retorna 9 y no 0, 3 ni 8.

15. Logging, eventos y observabilidad
Logging estructurado.

Timestamps UTC.

run_id, node_id, attempt_id en contexto.

Redacción de secretos.

No registrar contenido sensible innecesario.

Eventos append-only por transición.

Errores con código, categoría y mensaje seguro.

Artefactos y logs con límites de retención configurables.

Registro de versiones y configuración efectiva.

No usar stdout de herramientas como log del motor; conservarlo como artefacto.

16. Testing y verificación
16.1 Capas
Unit tests: canonicalización, modelos, validadores, estados, policy nodes.

Contract tests: adapters, parsers, policy nodes.

Integration tests: planner, scheduler, DB, pipeline.

Security tests: scope, argumentos, redirects, DNS, pinning.

Fault tests: timeout, cancelación, disco, DB, parser y writer fallidos.

End-to-end: workflow local con herramientas mock.

Regression tests: cada bug confirmado genera una prueba.

Platform tests: POSIX y Windows cuando se declare soporte.

16.2 Matriz mínima
Área	Casos
YAML	malformado, campos extra, tipos, defaults, anidamiento no estricto, namespaces inválidos
Planner	ciclo, referencia inválida, DAG correcto, module_kind incoherente, on_node_failure inválido
Scope	allowlist, exclusión, wildcard, CIDR
DNS	múltiples IP, cambio de resolución, rebinding
Redirect	destino permitido/no permitido, out_of_scope en cadena
Pinning	connected_ip en resolved_ips (ok), mismatch, unavailable
Executor	exit code, timeout, output grande
Process	SIGTERM ignorado, hijo, cleanup verificado con killpg(pgid,0)
Parser	JSONL/XML truncado, inválido, schema incompatible, DTD/entidades
Policy node	scope_validate con authorized/excluded/out_of_scope/invalid, suma de cobertura
DB	rollback, duplicado, foreign key, writer caído, saturación de cola
Artifact	checksum incorrecto, archivo faltante, path traversal, huérfano
Scheduler	skip, cancelación, dependencias, retry, abort-drain/abort-cancel/abort-subtree
Diff	cobertura completa, parcial e incompatible, MISSING vs UNKNOWN, abortado
Recovery	run interrumpido, reconciliación, idempotencia de recover, no borrado
CLI	exit codes y errores reproducibles, incluido código 9
16.3 Gates de CI
Cada pull request debe ejecutar:

lint;

format check;

type check;

unit tests;

contract tests;

tests de seguridad;

cobertura de tests;

dependency audit cuando esté disponible.

Gates adicionales obligatorios:

Test de arquitectura: northstar/config/ no importa asyncio.subprocess ni northstar/runtime/; northstar/adapters/ no importa northstar/storage/; northstar/diff/ no importa northstar/runtime/.

Test de shell=True: fallo estático si aparece en northstar/runtime/ o northstar/adapters/.

Test de strict anidado: fallo si algún BaseModel en la jerarquía de configuración carece de extra="forbid" y strict=True.

Test de module_kind: fallo si un ModuleSpec mezcla campos de tool y policy.

Test de namespaces: fallo si un template referencia un namespace no declarado en §5.5.

Test de invariante de cobertura: fallo si processed + omitted + failed != intended en cualquier fixture.

No se fija un porcentaje de cobertura como sustituto de calidad. Se exige cobertura de invariantes y casos críticos.

16.4 Definition of Done de una funcionalidad
Contrato definido.

Implementación tipada.

Tests unitarios.

Tests de integración pertinentes.

Tests negativos.

Logs y errores claros.

Documentación actualizada.

Sin secretos.

Sin regresiones.

Evidencia reproducible.

17. Roadmap
Milestone 0 — Contracts & Foundation
□ Crear paquete y CLI.
□ Definir schemas Pydantic con estrictitud recursiva verificada.
□ Definir module_kind y los contratos de tool vs policy.
□ Definir namespaces de templates y orden de resolución.
□ Definir errores y exit codes, incluido código 9.
□ Definir contratos de adapters/artifacts.
□ Definir máquina de estados y on_node_failure.
□ Validar YAML, referencias, ciclos y tipos.
□ Definir autorización y scope.
□ Tests de configuración y planner.
Gate: northstar validate no ejecuta procesos y rechaza cualquier configuración inválida, incluyendo submodelos no estrictos, module_kind incoherente y namespaces no declarados.

Milestone 1 — Security & Process Supervisor
□ Resolver ejecutables verificables.
□ Ejecutar argv sin shell.
□ Capturar stdout/stderr concurrentemente.
□ Timeout y cancelación.
□ Process groups y cleanup con verificación de killpg(pgid,0).
□ ArtifactStore e integridad.
□ Manifest por run.
□ Tests de procesos mock y descendientes.
Gate: no se declara proceso terminado sin wait/cleanup verificable y sin ProcessLookupError en la comprobación del grupo.

Milestone 2 — Workflow Runtime
□ Planner determinista.
□ Scheduler y estados.
□ Concurrencia global/target/tool.
□ Buffered pipeline.
□ Propagación de errores.
□ Retry limitado.
□ Implementación de on_node_failure en sus tres variantes.
□ Eventos persistibles.
□ Recuperación básica.
Gate: DAG válido ejecuta correctamente; DAG inválido no lanza procesos; las tres variantes de on_node_failure producen los estados esperados en tests.

Milestone 3 — Data Core
□ SQLite y migraciones.
□ Single writer con cola acotada.
□ Assets y observations.
□ Normalización versionada.
□ Coverage records con invariante de suma.
□ Idempotencia.
□ Integridad DB/filesystem.
□ Tabla orphan_artifacts y protocolo de reconciliación.
Gate: run repetido no duplica identidades; fallos de persistencia y saturación de cola no se ocultan; recover es idempotente y no borra evidencia por defecto.

Milestone 4 — Adapters
□ Resolver adapter.
□ Subfinder.
□ Httpx.
□ Nmap con defusedxml y forbid_dtd=True cuando no se requiera DTD.
□ Policy node scope_validate.
□ NetworkEvidenceRecord en adapters de red.
□ Verificación de versiones.
□ Fixtures reales controlados.
□ Scope efectivo y límites documentados.
Gate: cada adapter y policy node pasa contrato, parser y tests de seguridad XML y de pinning.

Milestone 5 — Diff & Reporting
□ NEW/EXISTING/CHANGED.
□ MISSING/UNKNOWN conservadores.
□ Compatibilidad de cobertura.
□ Reporte terminal/JSON.
□ Historial por target.
□ Evidencia asociada a findings.
Gate: no hay falsos MISSING en escenarios incompletos, abortados ni con pinning_status distinto de verified.

Milestone 6 — Hardening & Release
□ Fault injection.
□ Recovery y reconciliación.
□ Rendimiento y límites.
□ Auditoría de dependencias.
□ Documentación de instalación.
□ Release reproducible.
□ Pruebas de plataforma declarada.
□ Checklist de seguridad.
Gate: todos los criterios de aceptación críticos pasan y los riesgos residuales están documentados.

18. Definition of Done del MVP
El MVP solo se considera listo cuando:

□ Configuración inválida, ciclo o incompatibilidad se rechaza antes del spawn.
□ Ningún modelo anidado de configuración carece de extra="forbid" y strict=True.
□ Ningún ModuleSpec mezcla campos de tool y policy.
□ Ningún template referencia un namespace no declarado.
□ Un target no autorizado no ejecuta comandos ni conexiones.
□ La autorización es vigente, verificable y vinculada al scope.
□ Los adapters declaran sus límites reales de control de red y emiten NetworkEvidenceRecord cuando aplica.
□ shell=True no aparece en el runtime ni en adapters.
□ Timeout/cancelación no dejan procesos conocidos activos y el cleanup se verifica con killpg(pgid, 0).
□ Las tres variantes de on_node_failure funcionan según contrato.
□ stdout/stderr y manifests se conservan.
□ Artefactos tienen checksum y verificación.
□ Parser inválido no genera datos confiables; XML no confiable pasa por defusedxml con DTD/entidades/externos bloqueados según contrato.
□ SQLite aplica constraints y migraciones; el writer fallido o la cola saturada impiden éxito ficticio.
□ Segundo run diferencia activos nuevos y conocidos.
□ Diff respeta cobertura y compatibilidad; MISSING requiere completitud verificable y pinning_status: verified.
□ Resultados ausentes no se interpretan automáticamente como cierre.
□ Recovery detecta runs incompletos y artefactos huérfanos, es idempotente y no borra evidencia por defecto.
□ Exit code 9 se retorna cuando el run queda en estado de recuperación pendiente.
□ Tests críticos pasan en CI.
□ Documentación coincide con el código.
□ No existen secretos en repo, logs ni fixtures.
□ Los riesgos no resueltos están explícitamente documentados.
19. ADR — decisiones arquitectónicas
ADR	Decisión	Estado
ADR-001	Buffered/artifact-based primero	Aceptada para MVP
ADR-002	Pydantic v2 strict + extra forbid aplicado explícitamente a cada modelo anidado	Aceptada
ADR-003	SQLite con writer único y cola acotada	Aceptada para MVP
ADR-004	JSONL como intercambio cuando aplique	Propuesta
ADR-005	Scope y autorización fail-closed	Obligatoria
ADR-006	Nmap solo con targets explícitos autorizados y defusedxml endurecido	Obligatoria
ADR-007	Diff basado en observations y cobertura con invariante de suma	Obligatoria
ADR-008	Authorization reference obligatoria	Obligatoria
ADR-009	defusedxml con forbid_dtd=True, forbid_entities=True, forbid_external=True donde el formato lo permita	Obligatoria
ADR-010	Exit codes estables, incluido código 9 para recuperación pendiente	Aceptada
ADR-011	ExecutionPlan inmutable y determinista	Aceptada
ADR-012	Artefactos inmutables con SHA-256	Aceptada
ADR-013	Estado terminal exige cleanup verificado y persistencia	Aceptada
ADR-014	Recovery/reconciliación explícita, no destructiva por defecto	Aceptada
ADR-015	Streaming fuera del MVP	Aceptada
ADR-016	PEP 787 (t-strings) evaluado para Python ≥3.15; no adoptado antes	Registrada
ADR-017	Policy nodes como module_kind distinto, con contrato propio y NodeExecution sin proceso	Aceptada
ADR-018	Namespaces de templates target/params/env con resolución determinista	Aceptada
ADR-019	NetworkEvidenceRecord obligatorio en adapters de red; pinning_status afecta MISSING	Aceptada
ADR-020	on_node_failure con tres variantes: abort-drain (default), abort-cancel, abort-subtree	Aceptada
Cada ADR debe registrar: contexto, decisión, alternativas, consecuencias, riesgos, fecha y estado. Las decisiones aceptadas no se cambian silenciosamente.

20. Stack tecnológico
Capa	Tecnología
CLI	Typer
Configuración	PyYAML + Pydantic v2 (strict + extra="forbid" en cada modelo anidado)
Runtime	asyncio
Subprocess	asyncio.create_subprocess_exec
DB	SQLite + aiosqlite
XML	defusedxml con forbid_dtd=True, forbid_entities=True, forbid_external=True
Logging	logging estructurado
Terminal	Rich
Tests	pytest + pytest-asyncio
Lint	Ruff
Type checking	mypy o pyright (elegir uno y fijarlo)
Packaging	pyproject.toml
CI	GitHub Actions u otro runner equivalente
Las versiones de Python y dependencias deben fijarse en pyproject.toml y lockfile. No se debe depender de versiones flotantes en releases reproducibles. Versión mínima recomendada: Python 3.12+.

21. Seguridad operacional
Northstar no solicita ni requiere privilegios root por defecto.

No ejecutar herramientas contra targets no autorizados.

No usar configuraciones reales de scope en tests públicos.

No almacenar secretos en el repositorio.

No publicar artefactos que contengan información sensible.

Respetar límites y políticas de cada programa.

Deshabilitar módulos no revisados.

Mantener herramientas y dependencias actualizadas.

Revisar cambios de versión antes de actualizar adapters.

Registrar limitaciones de seguridad conocidas.

No afirmar que una prueba aislada constituye una auditoría de seguridad completa.

22. Riesgos residuales y límites explícitos
Las herramientas externas pueden cambiar comportamiento o formato.

Algunas herramientas resuelven DNS internamente y no permiten pinning verificable; en esos casos el NetworkEvidenceRecord marca connected_ip: unavailable y el nodo es DEGRADED, nunca SUCCESS.

SQLite y filesystem no comparten transacción atómica.

La terminación de procesos descendientes depende de plataforma y comportamiento del sistema; la verificación con killpg reduce pero no elimina el riesgo en plataformas no POSIX.

La cobertura observada depende de la calidad y semántica de cada herramienta.

Un checksum detecta alteración, pero no demuestra por sí mismo autenticidad del origen.

La autorización puede expirar o cambiar fuera de Northstar.

El software no puede garantizar ausencia absoluta de bugs.

La ejecución reproducible no implica que la red remota produzca resultados idénticos.

Los controles deben ser revisados y probados en el entorno real de uso.

defusedxml bloquea vectores conocidos, pero no sustituye límites de tamaño de entrada ni validación de esquema.

La cola acotada del writer introduce backpressure; si el productor no la maneja, puede degradar el throughput del run.

NetworkEvidenceRecord depende de que la herramienta exponga la información de conexión; si no lo hace, la garantía de pinning se degrada a unavailable y el usuario debe decidir si acepta ese nivel de confianza para su caso.

La política on_node_failure con abort-drain puede mantener el run vivo hasta el timeout de los nodos en RUNNING; no es una cancelación inmediata.

23. Próximo paso de implementación
Orden recomendado:

Crear estructura de paquete.

Implementar errors.py y modelos Pydantic con estrictitud recursiva.

Implementar loader y northstar validate, incluyendo validación de module_kind, namespaces y on_node_failure.

Implementar modelos de scope/autorización y validación.

Implementar planner determinista.

Implementar Process Supervisor aislado con mocks y verificación de cleanup.

Implementar ArtifactStore y manifest.

Implementar SQLite, migraciones y writer con cola acotada.

Implementar scheduler, on_node_failure y recovery.

Incorporar adapters uno por uno, empezando por Resolver y Subfinder, y el policy node scope_validate.

Implementar normalización, coverage con invariante de suma y NetworkEvidenceRecord.

Implementar diff y reporting.

Ejecutar hardening y gates de release.

No incorporar streaming, explotación, ejecución remota ni plugins externos antes de completar y verificar este baseline.

24. Registro de cambios
Versión	Fecha	Cambios
0.1.0	2026-11-12	Documento fundacional
0.2.0	2026-11-12	Scope, DNS pinning, cobertura, writer único, secretos, estados y manifests
1.0.0	2026-11-12	Consolidación de contratos, planner determinista, máquina de estados, recuperación, integridad, testing, gates y riesgos residuales
1.0.0	2026-11-12	Corrección: estrictitud Pydantic recursiva, defusedxml endurecido, invariante de suma en cobertura, verificación de cleanup con killpg, cola acotada del writer, gates de CI adicionales y riesgos residuales ampliados
1.1.0	2026-11-12	Cierre de cabos sueltos: module_kind y policy nodes (ADR-017); namespaces de templates (ADR-018); contrato de scope_validate; NetworkEvidenceRecord y pinning_status (ADR-019); on_node_failure con tres variantes (ADR-020); exit code 9; protocolo de reconciliación recover; invariante de cobertura ampliada a failed_targets
1.2.0	2026-11-12	Versión final verificada. Se refuerza con referencias explícitas a OWASP SSRF/DNS rebinding, la no-recursividad de Pydantic v2, la seguridad de create_subprocess_exec frente a shell=True, el patrón de writer único con cola acotada y los defaults de defusedxml. Se fija Python 3.12+ como versión mínima recomendada. Documento cerrado para implementación.
25. Referencias técnicas oficiales
Python asyncio subprocess: https://docs.python.org/3/library/asyncio-subprocess.html

Python asyncio tasks/cancellation: https://docs.python.org/3/library/asyncio-task.html

Pydantic strict mode: https://docs.pydantic.dev/latest/concepts/strict_mode/

Pydantic v2 — estrictitud no recursiva (confirmación): 

SQLite WAL: https://www.sqlite.org/wal.html

SQLite foreign keys: https://www.sqlite.org/foreignkeys.html

SQLite transactions: https://www.sqlite.org/lang_transaction.html

SQLite single writer con cola acotada (confirmación): 

Python packaging / pyproject.toml: https://packaging.python.org/en/latest/specifications/pyproject-toml/

Nmap XML output: https://nmap.org/book/output-formats-xml-output.html

ProjectDiscovery Subfinder: https://github.com/projectdiscovery/subfinder

ProjectDiscovery Httpx: https://github.com/projectdiscovery/httpx

OWASP SSRF Prevention Cheat Sheet — DNS rebinding y validación de IP: 

defusedxml documentation — defaults de forbid_dtd/forbid_entities: 

asyncio.create_subprocess_exec vs shell=True (confirmación): 

PEP 787 (t-strings para subprocess/shlex): https://peps.python.org/pep-0787/ (objetivo Python 3.15)

Las referencias son documentación de apoyo; deben consultarse contra las versiones concretas fijadas por el proyecto.

Cierre técnico
Northstar debe avanzar con una regla simple:

Primero contratos e invariantes; después implementación; luego pruebas; finalmente funcionalidades.

Este documento es el baseline arquitectónico propuesto para Northstar 1.2.0, la primera versión cerrada, verificada y lista para implementarse. La implementación solo podrá considerarse verificada cuando el repositorio real cumpla sus contratos y supere los gates definidos.

