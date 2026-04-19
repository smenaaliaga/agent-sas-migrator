Componentes del sistema                                                                                                                                                             
                                                                                                                                                                                        El sistema tiene 8 componentes en total: 2 módulos deterministas (código Python puro, sin LLM) y 6 agentes LLM especializados.                                                      
                                                                                                                                                                                        Módulos deterministas:                                                                                                                                                                                                                                                                                                                                                      - Orchestrator: gobernante del sistema. Mantiene el estado global, controla las 8 fases, valida gates de salida, maneja los checkpoints humanos bloqueantes. No razona, ejecuta     
  lógica.
  - EGPExtractor: descomprime el archivo .egp (que es un ZIP con XML desde SAS EG 7.1 en adelante), extrae los nodos de código SAS, construye el DAG de dependencias conectando nodos 
  por referencias input/output entre datasets.

  Agentes LLM:

  - CodeAnalyst: lee el código SAS completo en una sola pasada y produce clasificación de nodos (DATA step, PROC SQL, PROC SORT, macro, etc.), lineage columna-a-columna, detección de
   code smells (macros dinámicas, comparaciones directas de floats, nodos desconectados del DAG, dependencias externas ausentes) y propuestas de mejora escaneando ocho categorías.   
  - DataMatcher: trabaja sobre las muestras de input/output que entrega el usuario. Perfila schema y estadísticas, detecta artefactos humanos (hojas múltiples en Excel, celdas       
  combinadas, fórmulas, layouts pivoteados, headers desplazados, valores centinela), matchea cada archivo al nodo del DAG que le corresponde con puntaje de confianza, reconcilia     
  columnas con matching fuzzy y diccionario de sinónimos de dominio, detecta transformaciones implícitas manuales (splits, bucketings, escalas, recodificaciones).
  - Interviewer: ejecuta las dos entrevistas obligatorias con el usuario. Recibe listas estructuradas de "ambigüedades detectadas" de los otros agentes y las convierte en preguntas  
  agrupadas por tema. No inventa preguntas, responde a hallazgos concretos.
  - Translator: genera el código Python o los notebooks finales. Internamente modularizado por tipo de construcción SAS (sub-prompt para DATA steps, otro para PROC SQL, otro para    
  macros). Adapta el empaquetado según la estrategia de output elegida por el usuario.
  - Validator: corre los tests generados (parte código) y diagnostica mismatches cuando hay diferencias (parte LLM), identificando causa raíz probable: encoding, redondeo, semántica 
  de nulos, collation, orden de filas.
  - DocWriter: al final consolida todo el estado acumulado en documentación legible.

  ---
  Las 8 fases del pipeline

  Fase 0 — Intake. El Orchestrator cataloga los archivos que entrega el usuario sin abrirlos profundamente. Identifica candidatos a .egp, muestras tabulares (xlsx/csv/parquet/txt),  
  documentación (pdf/docx), otros. Produce un catálogo inicial con rol sospechado de cada archivo. Gate: hay al menos un candidato a .egp.

  Fase 1 — Entrevista Inicial (bloqueante). Antes de tocar el código SAS, el Interviewer abre una ronda de preguntas al usuario en seis bloques. Gate: se responden las preguntas     
  mínimas de cada bloque.

  Los bloques de esta entrevista son:

  - Identificación de archivos: cuál es el .egp principal, cuáles son muestras de entrada, cuáles de salida, cuáles intermedios, cuáles documentación, qué falta.
  - Contexto de negocio: qué hace el flujo, para qué se usa el output, frecuencia de ejecución, criticidad, quién lo ejecuta hoy y cómo.
  - Origen y destino de datos: de dónde vienen en producción (BD, archivos, servicios), volumen productivo aproximado, encoding, dónde va la salida.
  - Objetivos de la migración: traducción literal vs literal+refactor, stack Python preferido, dónde correrá el código, restricciones de librerías, formato de salida deseado (tres   
  opciones explicadas más abajo).
  - Tolerancias de validación: diferencias numéricas aceptables, orden de filas exacto o igualdad de conjuntos, columnas intermedias obligatorias o solo schema final.
  - Disponibilidad: si el usuario puede hacer una segunda ronda de preguntas tras analizar el código.

  Fase 2 — Análisis del código SAS. Se abre el .egp. El EGPExtractor produce el DAG y extrae los nodos. El CodeAnalyst clasifica cada nodo, construye el lineage, detecta code smells,
   identifica nodos desconectados, referencias externas, macros dinámicas, usos de auto-RETAIN, _N_, _ERROR_, formatos personalizados. Gate: DAG completo, todos los nodos
  clasificados, lineage al menos para inputs y outputs principales.

  Fase 3 — Profiling y Matching. El DataMatcher perfila las muestras, matchea cada archivo al nodo correspondiente del DAG con puntaje de confianza, reconcilia nombres de columnas   
  con matching fuzzy, detecta artefactos humanos y transformaciones implícitas manuales. Todo lo que detecte como pre-proceso manual se acumula en un "playbook candidato". Gate: cada
   archivo tiene mapping con confianza alta o está en cola de preguntas para la entrevista siguiente.

  Fase 4 — Entrevista Post-Análisis (bloqueante, la más importante). El Interviewer abre la segunda ronda con evidencia acumulada. Gate: cero preguntas bloqueantes abiertas y lista  
  de mejoras decidida (cada una aprobada, rechazada o pospuesta).

  Los bloques de esta entrevista son:

  - Confirmación de mapping archivo-a-nodo donde la confianza quedó baja.
  - Alcance del flujo: qué nodos ignorar (código muerto, diagnóstico, comentado), qué hacer con datasets huérfanos, cómo tratar referencias a libraries externas ausentes.
  - Confirmación del pre-proceso manual detectado: cada ajuste candidato se confirma, corrige o descarta.
  - Ambigüedades específicas del código SAS: macros dinámicas, semántica de nulos especiales (.A, .B, etc.), formatos con significado, uso de LAG y BY-processing.
  - Propuesta de mejoras, que es el entregable más crítico de esta fase: el CodeAnalyst listó todos los hallazgos en ocho categorías, cada uno presentado como ficha con ID (M-001,   
  M-002...), descripción, justificación, impacto, esfuerzo, riesgo y recomendación; el usuario aprueba, rechaza o pospone cada una individualmente.
  - Cierre: decisiones de arquitectura que el usuario quiera fijar antes de generar.

  Fase 5 — Plan de traducción. El Orchestrator consolida todo: mapeo nodo-a-módulo o nodo-a-notebook, orden topológico de ejecución, estrategia por tipo de nodo, lista de mejoras    
  aprobadas con dónde aplicarán, lista de nodos ignorados, estructura de carpetas del proyecto, dependencias Python. Muestra el plan al usuario y pide aprobación explícita. Gate:    
  usuario aprueba el plan.

  Fase 6 — Generación. El Translator produce los artefactos de salida. Genera siempre un preprocess.py que automatiza el pre-proceso manual detectado (unpivot, splits, lookups,      
  normalización). El flujo principal se genera según la estrategia de output elegida en Fase 1. Genera también config.yaml con parámetros externalizados, schemas.py con validaciones,
   requirements.txt o pyproject.toml, y esqueleto de tests.

  Fase 7 — Validación. El Validator corre los tests en cascada: schema → row count → agregados (sum, count, mean por columna) → row-by-row con tolerancia configurable. Cuando hay    
  mismatch no asume, diagnostica la causa raíz y propone fix al usuario. Gate: tests pasan dentro de tolerancias acordadas o el usuario acepta explícitamente las desviaciones        
  documentadas.

  Fase 8 — Documentación. El DocWriter genera el README con instrucciones de instalación y ejecución, el documento de lineage con tabla columna-origen a columna-destino por nodo, el 
  documento de decisiones consolidando las Q&A de ambas entrevistas con supuestos y razones, el documento de mejoras con las aplicadas (con antes/después) y las rechazadas/pospuestas
   (con motivo), el diagrama Mermaid del DAG original y el migrado, y el runbook de operación.

  ---
  Las tres estrategias de output (decisión del usuario en Fase 1)

  - Notebook por flujo: cada nodo significativo del DAG se convierte en su propio .ipynb numerado (01_extraccion.ipynb, 02_limpieza.ipynb, 03_agregacion.ipynb). Los datos intermedios
   se persisten en data/intermediate/ en formato parquet entre notebooks. Ventaja: espejo del .egp, fácil de debuggear etapa por etapa, permite que distintos analistas abran sólo la 
  parte que les toca. Desventaja: requiere orquestación externa (un run_all.py o Papermill), dependencias entre notebooks frágiles.
  - Notebook único: todo el flujo en un solo flow.ipynb con celdas agrupadas por sección con headers markdown. Ventaja: correr de punta a punta es trivial, un solo archivo.
  Desventaja: se vuelve gigante, diff de .ipynb en git es ruidoso, varias personas no pueden tocarlo a la vez.
  - Híbrido, default sugerido: notebooks por etapa del DAG (como la primera opción) importando funciones tipadas desde un directorio src/. Los notebooks quedan finos, solo con       
  orquestación, visualización y chequeos. preprocess.py y schemas.py quedan como código puro porque son utilitarios. Ventaja: lo mejor de ambos mundos, código testeable y exploración
   visual. Desventaja: estructura un poco más compleja de entender al inicio.

  ---
  Las 8 categorías del catálogo de mejoras

  El CodeAnalyst debe escanear exhaustivamente cada una de estas categorías y declarar explícitamente cuando una no tiene hallazgos. No se omite ninguna.

  - Performance: predicate pushdown (filtros antes de joins), eliminación de sorts innecesarios, evitar materializaciones intermedias no reutilizadas, vectorización en vez de loops, 
  chunking o streaming para volúmenes grandes, paralelización donde aplique, parquet en vez de CSV intermedio, caching de lookups repetidos.
  - Calidad y correctitud: manejo explícito de nulos considerando la semántica distinta entre SAS, pandas NaN y NA, validación de schema en boundaries con Pandera o Pydantic,        
  detección de duplicados por PK lógica, reemplazo de comparación directa de floats por tolerancia, manejo de zonas horarias, normalización de strings (trim, case, encoding),        
  detección de tipos mixtos en columnas.
  - Mantenibilidad: modularización de scripts monolíticos, parametrización de fechas, cortes, paths, umbrales, eliminación de código muerto, renombrado de variables con nombres      
  descriptivos, type hints, docstrings mínimos donde aporten, DRY para bloques duplicados, configuración externa vs hardcoded.
  - Reproducibilidad: versionado de inputs (hash o fecha), determinismo con orden estable y seeds fijos, tests de regresión con muestras, dependencias pineadas, idempotencia.        
  - Modernización del idiom: DATA step a pipe de pandas/polars en vez de iterrows, PROC SQL a duckdb (mantiene el SQL sin reescribir), macros a funciones Python parametrizadas,      
  formats SAS a Categorical o dict lookups, fechas SAS epoch a pd.Timestamp nativo, IF/THEN anidados a np.select o pd.cut.
  - Seguridad y robustez: credenciales fuera del código con env vars o secret managers, manejo de errores explícito sin except: pass, timeouts en conexiones, retry con backoff para  
  IO, validación de inputs externos.
  - Observabilidad: logging estructurado por etapa, row counts por etapa como métrica de salud, tiempos de ejecución por nodo, alertas de data quality cuando los nulos están fuera   
  del rango esperado.
  - Arquitectura: separación input/transform/output estilo hexagonal, abstracción de fuentes (el mismo código debería correr con CSV o con BD), modo dry-run o simulación,
  orquestación con decorators para Airflow/Prefect si aplica.

  ---
  Estado persistente

  Todo artefacto intermedio vive en disco en el directorio state/. Nada queda solo en memoria. Los archivos clave son:

  - migration_state.json: fase actual, artefactos producidos, preguntas abiertas y resueltas, decisiones tomadas con timestamp, mejoras propuestas con estado
  (aprobada/rechazada/pospuesta).
  - intake.json: catálogo inicial de archivos.
  - initial_interview.yaml: respuestas estructuradas de la primera entrevista.
  - flow_graph.json: DAG completo con nodos y aristas.
  - nodes/<id>.json: un archivo por nodo SAS con clasificación y metadatos.
  - lineage.json: mapa columna-a-columna por nodo.
  - code_smells.json: hallazgos problemáticos del código.
  - profile_report.json: perfiles de todas las muestras con artefactos humanos detectados.
  - file_mapping.json: archivo a nodo con confianza y razones.
  - column_mapping.yaml: reconciliación de columnas archivo-a-schema-esperado.
  - preprocess_candidates.yaml: pasos de pre-proceso manual candidatos a confirmar.
  - improvements_proposed.yaml: todas las mejoras encontradas con su ficha.
  - post_analysis_interview.yaml: respuestas de la segunda entrevista.
  - approved_improvements.yaml: subset de las aprobadas por el usuario.
  - ignored_nodes.yaml: nodos que el usuario decidió no migrar.
  - preprocess_playbook.yaml: playbook final consolidado de pre-proceso.
  - translation_plan.md: plan aprobado por el usuario.
  - validation_report.json: resultado de los tests con diagnósticos.

  Cada artefacto se valida contra un JSON Schema en schemas/ cuando cruza un gate entre fases. Si un agente produce un artefacto que no valida, el Orchestrator falla fuerte, no      
  tolera estado corrupto.

  ---
  Capacidades del Orchestrator expuestas al usuario

  - run: ejecuta desde cero.
  - resume: reanuda desde el último estado válido sin repetir trabajo.
  - status: reporta en qué fase está, qué artefactos se produjeron, qué preguntas quedan abiertas.
  - replay --phase N: re-ejecuta una fase específica usando el estado persistido de las previas, útil para iterar el Translator sin repetir el análisis.
  - --dry-run: corre hasta Fase 6 sin escribir el código Python o notebooks finales.

  ---
  Cómo pasarle todo esto a un LLM constructor para que lo implemente

  Para que un LLM constructor aplique esto bien necesita tres piezas, pasadas juntas en el mismo mensaje.

  Pieza 1 — Contrato de comportamiento. Un documento que describe exhaustivamente cómo se comporta el sistema resultante desde la perspectiva del usuario. Es lo que está arriba:     
  identidad, misión, principios inviolables, las 8 fases con sus gates, los bloques de cada entrevista, el catálogo de 8 categorías de mejoras, las reglas críticas que nunca se      
  violan, las tres estrategias de output. Este documento termina siendo el system prompt del Orchestrator y del Interviewer una vez construidos. Es la fuente de verdad funcional del 
  sistema.

  Pieza 2 — Encargo de construcción. Un documento separado con las instrucciones para el constructor. Debe incluir:

  - Rol del constructor: ingeniero senior que construye el sistema descrito en la Pieza 1, no ejecuta migraciones.
  - Stack obligatorio: Python 3.11+, Claude Agent SDK con claude-opus-4-7 por defecto, pandas + duckdb, Pandera o Pydantic, pytest, networkx para el DAG, Papermill para ejecutar     
  notebooks en tests, openpyxl para Excel, prompt caching habilitado en todo prompt estable mayor a 1024 tokens.
  - Arquitectura a implementar: los 8 componentes con responsabilidad, input, output y si son código puro o LLM.
  - Estructura de repo esperada: árbol de carpetas con src/sas_migrator/ para código, prompts/agents/ para prompts externalizados de cada agente, schemas/ para JSON Schema de cada   
  artefacto, tests/ con fixtures, examples/mini_pipeline/ con ejemplo funcional.
  - Contratos Pydantic mínimos: lista de modelos a implementar (MigrationState, FlowGraph, SASNode, DataProfile, FileMapping, ColumnMapping, PreprocessPlaybook, Improvement,
  InterviewQA, ValidationReport).
  - Comportamientos obligatorios del Orchestrator: validación de gates con JSON Schema antes de avanzar, reanudable desde disco, checkpoints humanos bloqueantes, dry-run, replay por 
  fase, logging estructurado por fase con tokens consumidos.
  - Reglas de implementación: prompts externos en archivos .md nunca hardcoded, todo output de agente siempre persistido en state/ nunca solo en memoria, prompt caching obligatorio, 
  el Translator recibe la estrategia de output como input y adapta el empaquetado, el Validator detecta formato (notebook o .py) y adapta el runner de tests, el Interviewer nunca    
  inventa preguntas responde a ambigüedades estructuradas.
  - Criterios de aceptación medibles: pytest pasa al 100%, el ejemplo mini_pipeline/ corre end-to-end produciendo código que valida, status reporta estado legible, ambas entrevistas 
  bloquean avance hasta tener respuestas mínimas, la entrevista post-análisis siempre incluye sección de mejoras con fichas, los artefactos de state/ son válidos contra sus JSON     
  Schema, matar el proceso a mitad y reanudar funciona.
  - Orden de implementación sugerido: esqueleto y modelos Pydantic primero; parser de .egp con tests; CodeAnalyst; DataMatcher; Orchestrator con gates y resume; Interviewer con CLI  
  interactiva; Translator modularizado por tipo de nodo; Validator con diagnóstico; DocWriter; ejemplo end-to-end último.
  - Prohibiciones explícitas: no implementar la migración SAS a Python en sí (se implementa el sistema que la hace), no hardcodear prompts, no usar frameworks multi-agente pesados   
  tipo CrewAI o AutoGen (orquestación explícita en Python es más auditable), no asumir estructura del .egp sin verificar con un ejemplo real, no usar un único prompt gigante para el 
  Translator.

  Pieza 3 — Fixture mínimo. Un archivo .egp pequeño real con 3 a 5 nodos simples (una lectura, una transformación, una agregación, una escritura), más las muestras correspondientes  
  de input y output esperado, más algún archivo con artefactos humanos intencionales (un Excel con hoja de ajustes, o con layout pivoteado, o con columnas mal nombradas). Sin fixture
   real, el constructor trabaja ciego y produce algo que parece correcto pero se rompe con datos reales.

  ---
  Formato de instrucción final al constructor

  La instrucción al LLM constructor se pasa con esta estructura:

  - Bloque "CONTEXTO" que contiene la Pieza 1 completa.
  - Bloque "ENCARGO" que contiene la Pieza 2 completa.
  - Bloque "FIXTURE" que contiene referencias o adjuntos a la Pieza 3.
  - Bloque "INSTRUCCIÓN FINAL" con pasos concretos: primero leer CONTEXTO completo; segundo resumir en cinco bullets los puntos no negociables extraídos; tercero detenerse y
  preguntar si detecta contradicciones entre CONTEXTO y ENCARGO (el CONTEXTO manda sobre el ENCARGO si hay conflicto); cuarto proceder con el orden de implementación; quinto al      
  terminar producir un STATUS.md con qué quedó implementado, qué quedó fuera y riesgos conocidos; sexto no implementar la migración SAS-a-Python, implementar el sistema que la hace. 

  ---
  Cinco condiciones que hacen que el constructor trabaje bien

  - Separar claramente "qué hace el sistema" (Pieza 1, contrato) de "cómo construirlo" (Pieza 2, encargo). Dos documentos distintos evita que el constructor mezcle objetivos.        
  - Todo artefacto tipado con Pydantic models más JSON Schema en los boundaries. Esto fuerza al constructor a definir contratos antes de codear lógica y hace que los errores
  aparezcan temprano.
  - Prompts de los sub-agentes como archivos .md externos. Prohibido hardcodearlos en Python. Permite iterar los prompts después sin tocar código, los versiona como artefactos de    
  primera clase y permite A/B testing del mismo agente con prompts distintos.
  - Fixture real y concreto. Un .egp de verdad con sus muestras. Sin esto el constructor produce algo que se rompe con datos reales.
  - Criterios de aceptación objetivos y medibles. "Tests pasan", "ejemplo corre end-to-end", "es reanudable". No "está bien hecho". El constructor necesita señales claras de cuándo  
  terminó.