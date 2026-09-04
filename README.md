# Tests — Weekly Staffing & Overbooking Review Agent

## Objetivo

Este documento define casos de prueba mínimos para validar que el agente aplique correctamente las reglas críticas de matching, clasificación, overbooking, disponibilidad y redistribución.

Cada caso incluye:

- **Escenario**
- **Input**
- **Resultado esperado**
- **Criterio de aprobación**

Estos casos pueden ejecutarse manualmente hoy y convertirse en tests automatizados en una versión futura.

---

## TEST-01 — TBD con información adicional entre paréntesis

### Escenario

Argentina Team contiene un TBD identificado únicamente por su ID, mientras que Flex Hours agrega información descriptiva entre paréntesis.

### Input

**Argentina Team**

```text
TBD-0001
```

**Flex Hours**

```text
TBD-0001 (TBD ASR AC Argentina BCM Associate)
```

### Regla a validar

El agente debe:

1. utilizar `TBD-0001` como identificador principal;
2. leer el descriptor entre paréntesis;
3. confirmar que el descriptor contiene `Argentina`;
4. incluir el recurso dentro de la población analizada.

### Resultado esperado

```text
Match: TRUE
TBD ID: TBD-0001
Geography: Argentina
Staff Category: Staff
```

### Criterio de aprobación

**PASS** si el TBD es identificado correctamente y se clasifica como Staff.

**FAIL** si:

- queda como unmatched;
- se toma el texto completo como un ID diferente;
- se ignora la validación de Argentina.

---

## TEST-02 — TBD con misma referencia pero otra geografía

### Escenario

Flex Hours contiene un TBD con el mismo ID, pero correspondiente a otra geografía.

### Input

**Argentina Team**

```text
TBD-0001
```

**Flex Hours**

```text
TBD-0001 (TBD ASR AC Mexico BCM Associate)
```

### Regla a validar

Un TBD sólo puede considerarse del equipo si el descriptor entre paréntesis contiene `Argentina`.

### Resultado esperado

```text
Match as Argentina resource: FALSE
```

### Criterio de aprobación

**PASS** si el registro no se incorpora como recurso de Argentina.

**FAIL** si el agente lo incorpora únicamente porque coincide el ID `TBD-0001`.

---

## TEST-03 — Mismo TBD ID en distintas categorías

### Escenario

El mismo identificador TBD aparece en Flex Hours asociado a distintas categorías.

### Input

```text
TBD-0001 (TBD ASR AC Argentina BCM Associate)
TBD-0001 (TBD ASR AC Argentina BCM Senior Associate)
TBD-0001 (TBD ASR AC Argentina BCM Manager)
```

### Regla a validar

La unidad de análisis debe ser:

```text
TBD ID + Staff Category
```

y no únicamente TBD ID.

### Resultado esperado

El agente debe crear tres posiciones independientes:

```text
TBD-0001 — Staff
TBD-0001 — Senior
TBD-0001 — Manager
```

### Criterio de aprobación

**PASS** si las horas permanecen separadas por categoría.

**FAIL** si el agente consolida las tres posiciones como una única persona/TBD.

---

## TEST-04 — TBD de Argentina presente en Flex pero no en Team List

### Escenario

Flex Hours contiene un TBD de Argentina que no aparece en Argentina Team.

### Input

**Argentina Team**

```text
TBD-0001
```

**Flex Hours**

```text
TBD-0001 (TBD ASR AC Argentina BCM Associate)
TBD-RR-9999999 (TBD ASR AC Argentina BCM Senior Associate)
```

### Regla a validar

El agente debe hacer un control independiente sobre Flex Hours para identificar TBDs de Argentina ausentes del Team List.

### Resultado esperado

El Summary debe incluir:

```text
Argentina TBDs in Flex not included in Team List

TBD-RR-9999999 — Senior
```

El TBD no debe incorporarse silenciosamente como miembro aprobado del equipo.

### Criterio de aprobación

**PASS** si el TBD se reporta como excepción.

**FAIL** si:

- no se detecta;
- se incorpora directamente al equipo sin advertencia.

---

## TEST-05 — Bloque TBD superior a la capacidad semanal normal

### Escenario

Un único Project Line Name tiene 60 horas asignadas a un TBD durante una semana normal.

### Input

```text
Project Line Name: Project A
Staff Category: Senior
Week: 14-Sep-2026
TBD Hours: 60
```

### Regla a validar

Durante períodos normales, ningún recurso o Proposed TBD debe superar 40 horas semanales.

### Resultado esperado

El agente debe dividir las 60 horas entre al menos dos recursos.

Ejemplo válido:

```text
Senior A: 40 h
Proposed TBD Senior 01: 20 h
```

o:

```text
Senior A: 30 h
Senior B: 30 h
```

### Criterio de aprobación

**PASS** si ningún recurso recibe más de 40 horas como consecuencia de la propuesta.

**FAIL** si las 60 horas se asignan completas a una única persona o Proposed TBD.

---

## TEST-06 — Bloque TBD superior a la capacidad de enero/febrero 2027

### Escenario

Un TBD tiene 70 horas durante febrero de 2027.

### Input

```text
Project Line Name: Project B
Staff Category: Staff
Week: 08-Feb-2027
TBD Hours: 70
```

### Regla a validar

Durante enero y febrero de 2027, el target máximo para redistribución es 55 horas por recurso.

### Resultado esperado

Las horas deben dividirse entre al menos dos recursos/TBDs.

Ejemplo:

```text
Proposed TBD Staff 01: 55 h
Proposed TBD Staff 02: 15 h
```

### Criterio de aprobación

**PASS** si ningún receptor supera 55 horas.

**FAIL** si un recurso recibe las 70 horas.

---

## TEST-07 — Continuidad priorizada frente a mayor capacidad disponible

### Escenario

Dos Seniors tienen capacidad para absorber un TBD.

- Senior A ya trabaja en Project A y tiene 10 horas disponibles.
- Senior B no trabaja en Project A y tiene 25 horas disponibles.
- El TBD requiere 8 horas.

### Input

```text
Project: Project A
TBD Hours: 8

Senior A:
Current Project A Hours: 20
Total Weekly Hours: 30
Capacity to 40: 10

Senior B:
Current Project A Hours: 0
Total Weekly Hours: 15
Capacity to 40: 25
```

### Regla a validar

La continuidad tiene prioridad sobre seleccionar automáticamente al recurso con mayor capacidad.

### Resultado esperado

```text
Target Resource: Senior A
Hours to Move: 8
Continuity indicator: TRUE
```

### Criterio de aprobación

**PASS** si Senior A recibe las horas.

**FAIL** si Senior B es seleccionado solamente porque tiene mayor capacidad.

---

## TEST-08 — Jackson Da Silva limitado a 20 horas

### Escenario

Jackson ya tiene 18 horas y existe un TBD Staff de 6 horas.

### Input

```text
Jackson Da Silva:
Current Weekly Hours: 18

TBD:
Staff Category: Staff
Hours: 6
```

### Regla a validar

Jackson tiene un máximo absoluto de 20 horas semanales.

### Resultado esperado

Jackson puede recibir como máximo:

```text
2 horas
```

Las 4 horas restantes deben asignarse a otro recurso elegible o TBD.

### Criterio de aprobación

**PASS** si Jackson termina con un máximo de 20 horas.

**FAIL** si termina con más de 20 horas.

---

## TEST-09 — Recurso de soporte no recibe horas adicionales

### Escenario

Magali Alvarez tiene capacidad aparente en Flex.

### Input

```text
Magali Alvarez:
Current Hours: 10

TBD Staff Hours: 20
```

### Regla a validar

Magali pertenece al equipo de soporte y no debe recibir horas adicionales.

### Resultado esperado

```text
Hours moved to Magali Alvarez: 0
```

### Criterio de aprobación

**PASS** si conserva sus horas actuales pero no recibe horas provenientes de TBDs u overbooks.

**FAIL** si se le asignan horas adicionales.

La misma regla debe aplicar a:

```text
Maria Guadalupe Musa
```

---

## TEST-10 — Recursos con otros clientes no reciben horas adicionales

### Escenario

Sofía Daniela Rodriguez o Martin Ogara presentan capacidad aparente.

### Input

```text
Sofía Daniela Rodriguez:
Current Hours: 20

Senior TBD Hours: 10
```

### Regla a validar

Los recursos con otros clientes pueden conservar sus asignaciones existentes, pero no recibir nuevas horas de esta redistribución.

### Resultado esperado

```text
Additional Hours Assigned: 0
```

### Criterio de aprobación

**PASS** si no reciben horas adicionales.

**FAIL** si son seleccionados como target de una redistribución.

---

## TEST-11 — Recursos fuera del equipo deben excluirse

### Escenario

Flex Hours contiene alguno de estos recursos:

```text
Esteban Aquino
Julia Rodriguez Saurina
Nancy Lopez Vazquez
```

### Regla a validar

No pertenecen al equipo y deben excluirse del análisis y de cualquier propuesta.

### Resultado esperado

No deben aparecer como:

- recurso disponible;
- receptor de horas;
- integrante elegible para reducir overbooking.

### Criterio de aprobación

**PASS** si están completamente excluidos de la lógica de redistribución.

**FAIL** si alguno recibe horas.

---

## TEST-12 — Sebastian Fauve Raul OOO en febrero 2027

### Escenario

Sebastian tiene horas asignadas durante las semanas en que estará Out of Office.

### Input

```text
Sebastian Fauve Raul

01-Feb-2027: 30 h
08-Feb-2027: 25 h
15-Feb-2027: 30 h
```

### Regla a validar

Sebastian está OOO hasta el 15-Feb-2027.

Las semanas afectadas son:

```text
01-Feb-2027
08-Feb-2027
```

### Resultado esperado

Propuesta:

```text
01-Feb-2027:
Sebastian = 0 h
30 h → TBD / recurso elegible

08-Feb-2027:
Sebastian = 0 h
25 h → TBD / recurso elegible

15-Feb-2027:
Sebastian puede mantener o recibir horas normalmente
```

### Criterio de aprobación

**PASS** si tiene cero horas propuestas en las semanas del 1-Feb y 8-Feb.

**FAIL** si conserva cualquier hora durante esas semanas.

---

## TEST-13 — No resolver un overbook creando otro

### Escenario

Senior A tiene 48 horas y Senior B tiene 38 horas durante una semana normal.

Se quieren mover 8 horas de A.

### Input

```text
Senior A: 48 h
Senior B: 38 h
Hours to Move: 8
```

### Regla a validar

El target recomendado es 40 horas semanales.

### Resultado esperado

Senior B puede absorber como máximo:

```text
2 horas
```

Las 6 restantes deben ir a otro Senior elegible o a un Proposed TBD.

### Criterio de aprobación

**PASS** si Senior B queda en 40 horas o menos.

**FAIL** si el agente mueve las 8 horas completas y deja a Senior B en 46.

---

## TEST-14 — Consolidación de TBDs

### Escenario

Existen dos requerimientos Staff diferentes en la misma semana:

```text
Project A: 15 h
Project B: 20 h
```

Ningún recurso real tiene capacidad.

### Regla a validar

Debe minimizarse el número de TBDs manteniendo los límites de capacidad.

### Resultado esperado

Un único recurso puede cubrir:

```text
Proposed TBD Staff 01
Project A: 15 h
Project B: 20 h
Total: 35 h
```

### Criterio de aprobación

**PASS** si se crea un único Proposed TBD.

**FAIL** si se crean dos TBDs sin una razón operativa que lo justifique.

---

## TEST-15 — Validación de conservación de horas

### Escenario

Se redistribuyen horas desde recursos overbooked o TBDs.

### Regla a validar

El agente no puede crear ni eliminar horas.

### Ejemplo

Antes:

```text
Source Resource: 10 h a mover
```

Después:

```text
Target A: 6 h
Target B: 4 h
```

### Resultado esperado

```text
Source Hours Removed = 10
Target Hours Added = 10
Difference = 0
```

### Criterio de aprobación

**PASS** si la diferencia es cero.

**FAIL** ante cualquier pérdida o duplicación de horas.

---

# Matriz mínima de regresión

| Test | Riesgo cubierto | Resultado esperado |
|---|---|---|
| TEST-01 | TBD con paréntesis | Match correcto |
| TEST-02 | TBD de otra geografía | Excluir |
| TEST-03 | TBD multicategoría | Separar por categoría |
| TEST-04 | TBD fuera del Team List | Reportar excepción |
| TEST-05 | Bloque >40 h | Dividir |
| TEST-06 | Bloque >55 h ene/feb | Dividir |
| TEST-07 | Continuidad | Priorizar mismo código |
| TEST-08 | Jackson 20 h | Nunca >20 |
| TEST-09 | Equipo soporte | No asignar nuevas horas |
| TEST-10 | Otros clientes | No asignar nuevas horas |
| TEST-11 | Fuera del equipo | Excluir |
| TEST-12 | Sebastian OOO | Cero horas durante OOO |
| TEST-13 | Nuevo overbooking | No generarlo |
| TEST-14 | Consolidación TBD | Minimizar TBDs |
| TEST-15 | Conservación de horas | Diferencia = 0 |

---

# Criterio general de aprobación

Una corrida del agente puede considerarse satisfactoria cuando:

1. todos los tests críticos de restricciones individuales pasan;
2. ningún Proposed Resource queda por encima de su capacidad objetivo;
3. ningún TBD se mezcla entre categorías;
4. TBDs de Argentina fuera del Team List son detectados;
5. las horas totales antes y después de la redistribución son iguales;
6. la propuesta prioriza continuidad antes de crear un nuevo recurso;
7. el output final sigue requiriendo aprobación humana antes de modificar sistemas productivos.

## Próximo paso recomendado

Convertir estos casos en fixtures de prueba y automatizar las validaciones mediante un runner, por ejemplo:

```text
tests/
├── README.md
├── fixtures/
│   ├── tbd_parenthesis/
│   ├── tbd_multicategory/
│   ├── large_block/
│   └── availability_restrictions/
└── test_staffing_rules.py
```

Esto permitiría ejecutar regresiones cada vez que se modifica el System Prompt o la lógica del agente.
