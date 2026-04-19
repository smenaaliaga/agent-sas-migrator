# agent-sas-migrator

 Sistema de Agentes: SAS (.egp) → Python

  Filosofía general

  Orquestación en pipeline con especialistas, no un agente monolítico. Cada fase   
  produce artefactos verificables (JSON/Markdown) que la siguiente consume. La     
  migración tiene checkpoints humanos obligatorios — no hay traducción ciega.      

  Principio clave: el .egp es la intención, las muestras son la evidencia, y las   
  preguntas cubren el gap entre ambas.

  ---
  Arquitectura en 8 fases

  [0] Intake → [1] Discovery → [2] Profiling → [3] Clarify (HITL)
        ↓                                              ↓
  [7] Docs ← [6] Validate ← [5] Generate ← [4] Plan ←┘

  Cada fase tiene criterios de salida (gate). Si falla el gate, vuelve a Clarify.  

  ---
  Agentes / Skills propuestos

  1. Orchestrator (agente maestro)

  Mantiene el estado global (migration_state.json): fase actual, artefactos,       
  preguntas abiertas, decisiones tomadas. Decide qué agente invocar según el estado
   y si hay ambigüedades sin resolver bloquea el pipeline.

  2. EGP Extractor

  - Descomprime el .egp (es un ZIP con XML desde EG 7.1; binario antes)
  - Extrae: nodos de código SAS, referencias a datasets, LIBNAME, el process flow  
  (DAG de dependencias), metadatos de conexiones
  - Salida: flow_graph.json + árbol de nodos .sas

  3. SAS Code Analyzer

  Por cada nodo clasifica:
  - DATA step (con SET, MERGE, BY, RETAIN, LAG, FIRST./LAST.)
  - PROC SQL, PROC SORT, PROC MEANS/FREQ/SUMMARY, PROC TRANSPOSE, PROC
  REG/LOGISTIC, etc.
  - Macros (%MACRO, %LET, %IF)
  - Formats/Informats
  - Entradas y salidas físicas/lógicas

  Salida: AST lógico + tabla de lineage columna-a-columna.

  4. Data Profiler

  Sobre las muestras entregadas (Excel/CSV/Parquet):
  - Schema inferido (tipos, nullability, cardinalidad)
  - Estadísticas (min/max, percentiles, valores únicos top-K, distribución de      
  nulos)
  - Detección de encoding, separadores, fechas ambiguas (DD/MM vs MM/DD)
  - Detecta si la muestra parece completa o parcial (cortes abruptos, fechas)      

  5. Schema Reconciler

  Cruza lo que dice el SAS (columnas esperadas por LIBNAME/PROC SQL) con lo que hay
   en la muestra. Flaguea:
  - Columnas renombradas entre origen real y muestra
  - Tipos que no calzan (e.g. SAS numérico con formato DATE9. vs string en CSV)    
  - Valores especiales SAS (.A, .B, .) perdidos al exportar

  6. Interviewer (skill crítico — humano-en-el-loop)

  Solo pregunta lo no deducible del código o las muestras. Agrupa preguntas por    
  lote para no fragmentar al usuario. Ejemplos:

  Sobre origen/destino
  - ¿La entrada original es tabla de BD (Oracle/Teradata/SQL Server) o archivo? Si 
  es BD, ¿la muestra es SELECT * completo o filtrada?
  - ¿El volumen productivo es ~10K, ~10M o ~1B filas? (define pandas vs polars vs  
  Spark/DuckDB)
  - ¿Python correrá en Airflow/notebook/container/lambda?

  Sobre semántica
  - Los valores faltantes especiales SAS (.A–.Z) ¿tienen significado de negocio?   
  ¿Se mapean a categorías distintas en Python?
  - Formatos como PROCENTAJE8.2 o $CAT. ¿son sólo display o cambian la lógica de   
  comparación?
  - BY-processing: ¿el orden de filas importa aguas abajo, o se puede reordenar?   

  Sobre tolerancias
  - ¿Diferencias numéricas aceptables? (default sugerido: 1e-9 relativo)
  - ¿Orden de filas debe coincidir exacto o basta con igualdad de conjuntos?       
  - ¿Columnas derivadas/intermedias deben existir, o sólo el output final?

  Sobre alcance
  - Macros con lógica dinámica (%EVAL, CALL EXECUTE): ¿migrar literal o
  refactorizar?
  - Código comentado / nodos desconectados del flujo: ¿ignorar o preservar?        
  - ¿Preferencia de stack? (pandas idiomático, polars, pyspark, SQL + duckdb)      

  7. Translator

  Genera Python idiomático, no transliterado. Reglas:
  - DATA step lineal → pipeline de assign/pipe en pandas
  - MERGE con BY → merge(how=...) explícito; si es SQL usa duckdb
  - PROC SQL → mantener como SQL sobre duckdb (más legible que traducir)
  - Macros → funciones Python parametrizadas
  - Fechas SAS (epoch 1960-01-01) → pd.to_datetime(... + offset)
  - Produce módulos separados por etapa del flow, no un script gigante

  8. Test Synthesizer + Validator

  - Genera un harness pytest que carga muestra de entrada, ejecuta el pipeline     
  Python, compara contra muestra de salida
  - Comparación multi-nivel: schema → row count → aggregates → row-by-row con      
  tolerancia configurable
  - Cuando hay mismatch, el Reconciliation Auditor diagnostica causa: ¿encoding?   
  ¿round-half-even vs half-up? ¿collation de string? ¿null semantics?

  9. Improvement Advisor

  Lee el AST y flagea:
  - Datasets intermedios no usados
  - PROC SORT seguidos de MERGE reemplazables por join sin ordenar
  - Filtros aplicables antes del join (predicate pushdown)
  - Hardcodes (paths, fechas, cortes) → candidatos a parámetros/config
  - Bloques duplicados → funciones
  - PROCs obsoletos con alternativas modernas
  - Oportunidades de paralelización / chunking

  10. Documentation Writer

  Produce:
  - README.md (cómo correr, requisitos, inputs/outputs)
  - LINEAGE.md (tabla columna-origen → columna-destino con transformación)
  - MIGRATION_NOTES.md (decisiones tomadas, preguntas respondidas, assumptions)    
  - Diagrama del DAG (Mermaid) antes y después
  - CHANGELOG.md de mejoras aplicadas vs literales

  ---
  Criterios de salida por fase (gates)

  ┌─────────────┬───────────────────────────────────────────────┐
  │    Fase     │                     Gate                      │
  ├─────────────┼───────────────────────────────────────────────┤
  │ 1 Discovery │ DAG extraído + todos los nodos clasificados   │
  ├─────────────┼───────────────────────────────────────────────┤
  │ 2 Profiling │ Schemas de todas las muestras + stats         │
  ├─────────────┼───────────────────────────────────────────────┤
  │ 3 Clarify   │ Cero preguntas bloqueantes abiertas           │
  ├─────────────┼───────────────────────────────────────────────┤
  │ 4 Plan      │ Mapa SAS→Python aprobado por usuario          │
  ├─────────────┼───────────────────────────────────────────────┤
  │ 5 Generate  │ Código corre sin error sintáctico             │
  ├─────────────┼───────────────────────────────────────────────┤
  │ 6 Validate  │ Validación pasa dentro de tolerancia acordada │
  ├─────────────┼───────────────────────────────────────────────┤
  │ 7 Docs      │ Artefactos generados y revisados              │
  └─────────────┴───────────────────────────────────────────────┘

  ---
  Decisiones de diseño que sugiero discutir contigo

  1. ¿Traducción literal primero, refactor después, o todo-en-uno? Recomiendo
  literal validado → refactor iterativo. Facilita validar contra SAS original.
  2. Stack target por defecto: pandas + duckdb es el punto dulce (SQL-heavy SAS se
  traduce 1:1, DATA steps a pandas). Polars si el volumen lo exige.
  3. Persistencia del Interviewer: guardar Q&A en decisions.yaml versionado —
  futuras migraciones similares reciclan respuestas.
  4. Validación en CI: el harness generado debería correr en cada cambio, no sólo
  al final.

  ---
  Riesgos que el sistema debe cubrir explícitamente

  - Muestra no representativa (sin edge cases como nulos, fechas extremas,
  caracteres especiales) → Profiler debe advertir "cobertura baja"
  - Lógica implícita SAS (auto-retain, _N_, _ERROR_) fácil de perder → Analyzer
  debe detectarla
  - Encoding (WLATIN1 vs UTF-8) → romperá strings silenciosamente si no se atiende
  - Macros dinámicas que generan código en runtime → requieren decisión humana caso
   por caso

  ---
  Próximos pasos concretos

  ¿Quieres que avance con alguno de estos?
  1. Esqueleto de repo (/agents, /skills, /prompts, state schema)
  2. Prototipo del EGP Extractor (parser del ZIP/XML de .egp)
  3. Catálogo de reglas SAS→Python (tabla de mapeos por construcción)
  4. Template del Interviewer (banco de preguntas con lógica condicional)

  Dime por cuál partir y si hay algo del diseño que quieras ajustar antes de
  implementar.

✻ Worked for 1m 37s

※ recap: Diseñamos un sistema multi-agente para migrar código SAS .egp a Python con                                                                                        
   8 fases (discovery, profiling, clarify, plan, generate, validate, docs) y 10
  agentes especialistas. Falta que elijas por dónde partir: esqueleto de repo,
  parser .egp, catálogo de reglas SAS→Python, o template del Interviewer. (disable
  recaps in /config