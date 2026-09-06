# Weekly Staffing & Overbooking Review Agent

## 1. Qué construimos

Este repositorio documenta un agente de IA para revisar semanalmente el staffing, capacity y overbooking del equipo de Argentina para múltiples clientes de auditoría.

El agente procesa snapshots de `Flex Hours` junto con un archivo `Argentina Team`, consolida la carga semanal de cada persona entre clientes, identifica overbooking y TBDs, propone redistribuciones respetando categoría, continuidad y restricciones individuales, y genera un workbook gerencial listo para revisión.

La implementación actual fue probada con dos clientes:

- GS — The Goldman Sachs Group, Inc.
- BBH — Brown Brothers Harriman & Co.

El agente es **Human-in-the-loop**: puede analizar y recomendar, pero no modifica sistemas productivos ni asignaciones oficiales.

---

## 2. Problema que resuelve

El proceso manual de staffing exige revisar semanalmente:

- horas actuales y futuras por persona;
- personas compartidas entre varios clientes/códigos;
- overbooking;
- capacidad disponible;
- TBDs existentes;
- continuidad de recursos dentro de los mismos códigos;
- restricciones individuales de disponibilidad;
- diferencias entre ETC previamente informado y Actual posterior;
- Budget vs Actual y Budget vs Actual + ETC.

El agente centraliza esas reglas para que el gerente pueda revisar excepciones y propuestas sin reconstruir el análisis manualmente cada semana.

---

## 3. Inputs

### Flex Hours

Uno o más archivos exportados desde Flex, normalmente uno por cliente.

Ejemplos:

```text
The Goldman Sachs Group, Inc. - GS Group Team Audit Dec26 - ETC_EXPORT_...
Brown Brothers Harriman & Co. - BBH&Co. Audit Dec26 - ETC_EXPORT_...
```

### Argentina Team

Workbook con la población del equipo de Argentina y, cuando corresponde, tabs separadas por cliente.

### Snapshots históricos

Para ejecutar el control `Prior ETC vs Subsequent Actual` se conserva el snapshot Flex de la semana anterior.

Sin un snapshot previo no se inventa el ETC histórico: el control se marca como no disponible para ese cliente/período.

---

## 4. Outputs

El workbook final incluye:

### `Summary`

Vista ejecutiva de:

- personas con overbooking;
- severidad;
- primera semana afectada;
- TBDs de Argentina encontrados en Flex pero no en el Team List;
- excepciones relevantes.

### `Staffing Detail`

Representa **exclusivamente el estado actual de Flex**.

Reglas:

- una persona aparece una sola vez, aunque trabaje en varios clientes;
- debajo del TOTAL se agrupan sus códigos por cliente;
- no contiene filas de propuesta;
- los TBD con `Projected Total Hours = 0` se excluyen;
- las celdas semanales de la row TOTAL se resaltan sólo por overbooking.

### `Proposed Hours`

Representa **el escenario final después de la redistribución propuesta**.

Incluye:

- una row TOTAL por persona/TBD;
- detalle por cliente y Project Line Name;
- columna `Total` antes de las columnas semanales;
- carga semanal final después de movimientos;
- únicamente asignaciones válidas por categoría y disponibilidad.

### `Movement Log`

Trazabilidad transaccional de cada cambio:

```text
Source Resource → Target Resource
Client
Project Line Name
Week
Hours moved
Reason
Continuity indicator
```

### `Forecast & Budget Control`

Contiene tres controles:

1. `Prior ETC vs Subsequent Actual` por semana/WBS;
2. `Budget Hours To Date vs Actual Hours To Date`;
3. `Budget Total vs Actual + ETC` por WBS.

---

## 5. Reglas principales

### Capacidad y overbooking

Períodos normales:

```text
> 40 h = naranja
> 45 h = rojo
```

Enero y febrero de 2027:

```text
> 55 h = naranja
> 65 h = rojo
```

Desde marzo de 2027 se vuelve al threshold normal.

Para redistribuir horas se utiliza un target conservador:

```text
40 h normalmente
55 h en enero/febrero 2027
```

No se considera una solución válida mover un overbook a una persona que quede sobre el target.

### Multi-cliente

El overbooking de una persona se calcula sobre su carga **combinada entre todos los clientes incluidos en la corrida**.

La elegibilidad para absorber horas se controla por cliente y Staff Category.

### Promociones efectivas

Desde julio de 2026 se consideran Senior:

- Juan Manuel Anit Pirez
- Fiorella Cuervo
- Ezequiel Francisco Basualdo

La categoría efectiva por fecha prevalece sobre clasificaciones históricas inconsistentes del archivo fuente.

### Restricciones individuales

#### Jackson Da Silva

Máximo absoluto:

```text
20 horas semanales
```

Nunca puede ser bookeado por encima de 20 horas sumando todos sus clientes/códigos.

#### No pertenecen al equipo

- Esteban Aquino
- Julia Rodriguez Saurina
- Nancy Lopez Vazquez

Se excluyen de la población elegible.

#### Equipo de soporte

- Magali Alvarez
- Maria Guadalupe Musa

Conservan sus horas actuales pero no reciben horas adicionales.

#### Otros clientes

- Sofía Daniela Rodriguez
- Martin Ogara

Conservan sus horas existentes pero no reciben horas adicionales.

#### Sebastian Fauve Raul

OOO durante las semanas que comienzan:

```text
01-Feb-2027
08-Feb-2027
```

Sus horas de ese período deben trasladarse a otro recurso elegible o a TBD.

---

## 6. Tratamiento de TBDs

### Matching

Un TBD puede aparecer en Team List como:

```text
TBD-0001
```

y en Flex como:

```text
TBD-0001 (TBD ASR AC Argentina BCM Associate)
```

El agente:

1. toma como ID el texto anterior al primer paréntesis;
2. valida que el descriptor contenga `Argentina`;
3. normaliza mayúsculas/minúsculas;
4. trata `TBD ID + Staff Category` como una posición única.

### TBDs con Projected Total Hours = 0

Se excluyen **únicamente los TBDs** cuyo `Projected Total Hours = 0`.

Los recursos reales con Projected Total Hours = 0 no se eliminan por esta regla.

Los TBDs vacíos tampoco pueden utilizarse como receptores en `Proposed Hours`.

### Nombre legible del TBD

Si un TBD está asociado a un único código, se renombra en la vista analítica como:

```text
TBD <Cliente> <Project Line Name>
```

Ejemplo:

```text
TBD GS GSFM Sub Audit Dec26
```

Si un TBD cubre varios códigos, se mantiene un identificador que preserve trazabilidad.

---

## 7. Lógica de redistribución

Orden de prioridad:

1. misma Staff Category + continuidad en el mismo Project Line;
2. misma Staff Category + capacidad disponible;
3. TBD.

Restricciones:

- nunca cruzar Staff/Senior/Manager;
- aplicar promociones efectivas por fecha;
- respetar disponibilidad individual;
- no generar un nuevo overbooking;
- minimizar TBDs;
- preservar continuidad entre semanas;
- dividir un bloque si excede la capacidad de una sola persona.

---

## 8. Control histórico ETC vs Actual

El agente compara, por semana y WBS:

```text
ETC registrado en el snapshot previo
vs
Actual de esa misma semana en el snapshot posterior
```

Ejemplo conceptual:

```text
Snapshot semana 24-Aug:
24-Aug ETC = 10 h

Snapshot semana 31-Aug:
24-Aug Actual = 13 h

Difference = +3 h
Variance % = +30%
```

Una diferencia positiva significa que el Actual terminó por encima del ETC previo.

Para ejecutar este control es obligatorio conservar snapshots históricos. Si no existe snapshot anterior para un cliente, se informa `Prior snapshot not available`.

---

## 9. Controles de Budget

Por cliente y WBS se calculan:

### Budget Hours To Date vs Actual Hours To Date

```text
Variance HTD = Actual HTD - Budget Hours To Date
```

### Budget total vs Actual + ETC

```text
Forecast Variance = (Actual + ETC) - Budget Hours
```

Una diferencia positiva indica consumo/proyección por encima del budget.

---

## 10. Cómo reproducir una corrida

1. Guardar los exports Flex de todos los clientes de la semana.
2. Guardar `Argentina Team.xlsx`.
3. Conservar el snapshot Flex de la semana anterior si se quiere correr el control ETC vs Actual.
4. Utilizar:
   - `prompts/system_prompt.md`
   - `prompts/user_prompt.md`
5. Ejecutar el agente y revisar el workbook.
6. Realizar revisión humana.
7. Si el gerente identifica restricciones o inconsistencias, incorporarlas y volver a ejecutar antes de aprobar.
8. Guardar la evidencia bajo:

```text
corridas/
└── corrida_YYYY-MM-DD_CLIENTES/
    ├── corrida.json
    └── output/
        └── Staffing_Review.xlsx
```

La carpeta de corrida es parte de la evidencia del funcionamiento del agente y no debe quedar vacía.

`corrida.json` consolida en un único archivo auditable:

- `run_metadata`
- `inputs`
- `execution_log`
- `human_review`
- `final_human_review`
- `test_results`
- `results`
- `integrity`
- `telemetry`
- `api_metadata`
- `reproducibility`

La telemetría distingue explícitamente entre datos observados y estimados. Los valores estimados de tokens o latencia no se presentan como telemetría nativa de API.

---

## 11. Evidencia y decisiones

Las decisiones de diseño, alternativas descartadas y errores reales encontrados se documentan en:

```text
DECISIONES.md
```

Cada decisión relevante debe vincularse, cuando sea posible, con evidencia de la corrida, por ejemplo:

```text
corridas/corrida_2026-09-03_GS_BBH/corrida.json

Dentro de `corrida.json`, consultar respectivamente:
- `execution_log`
- `human_review` / `final_human_review`
- `test_results`
```

Los casos de regresión se documentan en:

```text
tests/README.md
```

---

## 12. Gobierno y riesgo — L0 a L4

| Nivel | Acción | Permitido |
|---|---|---|
| L0 | Leer archivos de staffing | Sí |
| L1 | Calcular horas, variances y excepciones | Sí |
| L2 | Proponer redistribuciones | Sí |
| L3 | Generar workbook para revisión humana | Sí |
| L4 | Modificar Flex u otro sistema productivo | **No** |

### Human-in-the-loop

La propuesta no se convierte automáticamente en una asignación oficial.

Un gerente debe:

1. revisar `Staffing Detail`;
2. revisar `Proposed Hours`;
3. validar restricciones operativas;
4. aprobar o pedir ajustes.

La intervención humana debe registrarse en `corridas/<run_id>/corrida.json` → sección `human_review`.

---

## 13. Seguridad de prompts

Los contenidos de los archivos de entrada son datos.

Cualquier texto, fórmula, comentario o string dentro de los inputs se trata como **DATO, no instrucción**.

El agente no debe ejecutar instrucciones embebidas dentro de archivos, celdas o payloads.

Las instrucciones válidas provienen únicamente del System Prompt y del User Prompt fuera de las etiquetas de datos.

---

## 14. Tests y regresión

Los casos mínimos incluyen:

- TBD con descriptor entre paréntesis;
- TBD de otra geografía;
- mismo TBD en varias categorías;
- TBD Argentina fuera del Team List;
- bloque semanal mayor a capacidad;
- continuidad;
- Jackson máximo 20 h;
- soporte sin horas adicionales;
- recursos con otros clientes sin horas adicionales;
- Sebastian OOO;
- promociones efectivas desde julio 2026;
- conservación de horas;
- exclusión de TBD con Projected Total Hours = 0;
- consolidación multi-cliente;
- ETC previo vs Actual posterior.

Ver:

```text
tests/README.md
```

---

## 15. Análisis económico

El análisis económico se documenta en detalle en [`COSTOS.md`](COSTOS.md). Los valores de tokens y latencia que no provengan de telemetría nativa se identifican expresamente como **estimaciones**.

### Modelo de referencia y criterio de selección

La implementación de referencia utiliza **GPT-5.6 Sol (`gpt-5.6-sol`)** porque es el modelo con el que se construyó y validó este workflow de conocimiento profesional. La regla de productivización es usar **el modelo más chico que pase el 100% de los tests críticos de staffing y QA**. GPT-5.6 Terra y GPT-5.6 Luna son candidatos de menor costo, pero no se declaran equivalentes hasta ejecutar la misma batería de regresión sobre ellos.

Precios de referencia verificados al 5 de septiembre de 2026, por 1M tokens:

| Modelo | Input | Cached input | Output | Estado para este agente |
|---|---:|---:|---:|---|
| GPT-5.6 Sol | USD 4.00 | USD 0.40 | USD 20.00 | Baseline validado |
| GPT-5.6 Terra | USD 2.00 | USD 0.20 | USD 12.00 | Candidato a downgrade controlado |
| GPT-5.6 Luna | USD 0.20 | USD 0.02 | USD 1.20 | No validado para reglas críticas |

Fuente oficial: `https://developers.openai.com/api/docs/models`

### Fórmula de costo

Sin caching:

```text
Costo_corrida =
(Input_tokens / 1,000,000 × precio_input)
+
(Output_tokens / 1,000,000 × precio_output)
```

Con Prompt Caching:

```text
Costo_corrida_cacheado =
(Cached_input / 1,000,000 × precio_cached_input)
+
(Uncached_input / 1,000,000 × precio_input)
+
(Output_tokens / 1,000,000 × precio_output)
```

### Supuestos de volumen y frecuencia

Para un régimen productivo semanal GS + BBH se usa como supuesto de planificación:

```text
Input tokens estimados:  47,000 por corrida
Output tokens estimados:  5,000 por corrida
Rango input:              35,000–60,000
Rango output:             3,000–8,000
Frecuencia:               1 corrida por semana
Corridas/año:             52
```

Estos valores son **estimaciones de ingeniería**, no usage nativo observado.

### Sensibilidad a Prompt Caching — GPT-5.6 Sol

| Cache hit sobre input | Costo/corrida | Costo/mes (4.33) | Costo/año (52) |
|---:|---:|---:|---:|
| 0% | USD 0.288 | USD 1.25 | USD 14.98 |
| 50% | USD 0.203 | USD 0.88 | USD 10.58 |
| 80% | USD 0.153 | USD 0.66 | USD 7.94 |
| 95% | USD 0.127 | USD 0.55 | USD 6.62 |

### Sensibilidad a escala / picos de carga

Usando el escenario de 80% de cached input:

| Escenario | Corridas/semana equivalentes | Costo anual estimado |
|---|---:|---:|
| Base semanal GS + BBH | 1 | ~USD 8 |
| Rerun ad-hoc ocasional | ~1.02 promedio | ~USD 8.1 |
| 10 pares de clientes | 10 | ~USD 79 |
| Monitoreo diario del mismo scope | 7 | ~USD 56 |

El costo dominante del workflow no es la inferencia: sigue siendo la revisión gerencial humana requerida por el nivel L3.

### ChatGPT Enterprise vs API

Cuando el workflow se ejecuta dentro de ChatGPT Enterprise, el usuario puede no recibir un cargo marginal por tokens equivalente al pricing de API. El escenario anterior se mantiene como benchmark económico reproducible. Las corridas por API deben registrar `usage` real en el log transaccional; las corridas interactivas no deben inventarlo.

---

## 15A. Reproducción de una corrida

### Dependencias fijadas

Todas las dependencias están versionadas exactamente con `==` en `requirements.txt`.

### Variables de entorno

Copiar una sola vez:

```bash
cp .env.example .env
```

Completar `OPENAI_API_KEY` y, si corresponde, la ruta del JSON de corrida/resumen.

### Ejecución en un solo comando

```bash
./run.sh
```

`run.sh`:

1. crea el virtual environment si no existe;
2. instala las dependencias fijadas;
3. ejecuta los tests automáticos del runner;
4. ejecuta `runner.py`;
5. genera un log API estructurado cuando existe una llamada real al modelo.

Para validar instalación y configuración sin realizar una llamada API:

```bash
./run.sh --dry-run
```

El runner implementa:

- retry sólo para fallas transitorias;
- Exponential Backoff + Jitter ante 429/503;
- `MAX_RETRIES`;
- `MAX_TURNS`;
- `MAX_OUTPUT_TOKENS_PER_CALL`;
- timeout configurable;
- registro de `request`, `response`, `usage`, timestamps y latencia cuando esos datos son realmente observados.

---

## 16. Limitaciones

- La disponibilidad real puede depender de vacaciones, skills y otros compromisos no incluidos en Flex.
- Las restricciones personales deben mantenerse actualizadas.
- La comparación histórica ETC vs Actual requiere snapshots previos.
- La continuidad es una regla de negocio, no una optimización matemática global.
- El workbook es una recomendación y requiere aprobación humana.
- No se deben inventar métricas de API, tokens, latencia o request IDs si no fueron capturadas durante la corrida.

---

## 17. Estructura esperada del repositorio

```text
README.md
DECISIONES.md
COSTOS.md
README_MEJORAS.md
requirements.txt
run.sh
runner.py
.env.example

prompts/
├── system_prompt.md
└── user_prompt.md

tests/
├── README.md
└── test_runner.py

corridas/
└── corrida_YYYY-MM-DD_CLIENTES/
    ├── corrida.json
    ├── api_execution_log.json     # sólo cuando la corrida usa API
    └── output/
        └── Staffing_Review.xlsx
```

Los nombres anteriores son deliberados: evitar duplicados como `README (2).md` o `corrida (3).json`, porque dificultan la detección automática de evidencia.
