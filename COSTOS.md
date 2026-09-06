# COSTOS.md — Análisis Económico del Agente

## Objetivo

Estimar el costo del agente **Weekly Multi-Client Staffing & Overbooking Review** en régimen semanal. Todos los tokens/latencias no provenientes de telemetría nativa se presentan como **estimaciones**, nunca como observaciones reales.

## 1. Criterio de elección de modelo

La regla de productivización es: **usar el modelo más chico que pase el 100% de los tests críticos de staffing, restricciones y QA**.

| Modelo | Precio input / cached / output por 1M tokens | Evaluación |
|---|---:|---|
| GPT-5.6 Sol | USD 4.00 / 0.40 / 20.00 | Baseline actualmente validado en el workflow |
| GPT-5.6 Terra | USD 2.00 / 0.20 / 12.00 | Candidato de menor costo; requiere la misma regresión antes de reemplazar Sol |
| GPT-5.6 Luna | USD 0.20 / 0.02 / 1.20 | Muy económico, pero no se declara apto para reglas críticas sin evidencia de tests |

Fuente oficial verificada al 05/09/2026:  
`https://developers.openai.com/api/docs/models`

No se afirma que Terra o Luna resuelvan el workflow hasta ejecutar la batería de regresión. Esto evita convertir una hipótesis de ahorro en un claim de calidad no probado.

## 2. Fórmula de costo

Sin caching:

```text
Costo_corrida =
(I / 1,000,000) × precio_input
+
(O / 1,000,000) × precio_output
```

Con Prompt Caching:

```text
Costo_corrida_cacheado =
(I_cache / 1,000,000) × precio_cached_input
+
(I_fresh / 1,000,000) × precio_input
+
(O / 1,000,000) × precio_output
```

donde `I = I_cache + I_fresh`.

## 3. Supuestos de volumen y frecuencia

Supuesto de planificación para una corrida semanal GS + BBH:

- frecuencia: **1 corrida/semana**, 52/año;
- input estimado: **47.000 tokens**, rango 35.000–60.000;
- output estimado: **5.000 tokens**, rango 3.000–8.000;
- los archivos Excel crudos deben procesarse con código; el modelo recibe instrucciones y resultados estructurados;
- las cifras son estimaciones de ingeniería, no `usage` observado.

## 4. Sensibilidad a Prompt Caching — GPT-5.6 Sol

Con `I = 47.000`, `O = 5.000`, input USD 4/MTok, cached input USD 0.40/MTok y output USD 20/MTok:

| Cache hit ratio | Costo/corrida | Costo/mes (4.33) | Costo/año (52) |
|---:|---:|---:|---:|
| 0% | USD 0.288 | USD 1.25 | USD 14.98 |
| 50% | USD 0.203 | USD 0.88 | USD 10.58 |
| 80% | USD 0.153 | USD 0.66 | USD 7.94 |
| 95% | USD 0.127 | USD 0.55 | USD 6.62 |

Ahorro estimado 95% vs 0%: aproximadamente **56%** en este mix de tokens.

## 5. Sensibilidad por modelo — sin caching

Mismos supuestos de 47k input / 5k output:

| Modelo | Costo/corrida | Costo/año (52) |
|---|---:|---:|
| GPT-5.6 Sol | USD 0.288 | USD 14.98 |
| GPT-5.6 Terra | USD 0.154 | USD 8.01 |
| GPT-5.6 Luna | USD 0.0154 | USD 0.80 |

La tabla muestra el incentivo económico de bajar de modelo, pero **la decisión sólo se toma después de pasar los tests críticos**.

## 6. Picos de carga / SLO

Usando Sol con 80% cached input:

| Escenario | Multiplicador aproximado | Costo anual |
|---|---:|---:|
| Base semanal | 1x | ~USD 8 |
| Rerun ad-hoc ocasional | ~1.02x | ~USD 8.1 |
| 10 pares de clientes | 10x | ~USD 79 |
| Monitoreo diario | 7x | ~USD 56 |

El riesgo de picos no es sólo costo: más corridas simultáneas aumentan probabilidad de 429/503 y pueden afectar el SLO. Por eso `runner.py` implementa backoff exponencial con jitter, timeout y límites duros.

## 7. Telemetría observada vs estimada

`runner.py` guarda en cada llamada API, cuando estén disponibles:

- `request_id`;
- `input_tokens`;
- `output_tokens`;
- `total_tokens`;
- `latency_ms`;
- retries;
- timestamps.

Si la corrida es interactiva y esa telemetría no está expuesta, los campos observados deben quedar `null`; las estimaciones económicas se guardan por separado.

## 8. ChatGPT Enterprise

El costo marginal visible para el usuario dentro de ChatGPT Enterprise puede no coincidir con un cargo API por token. Por eso este documento utiliza precios API como benchmark comparable y reproducible.

## 9. Limitaciones

- Los precios cambian; verificar la fuente oficial antes de presupuestar.
- El hit ratio real de caching depende de estabilidad del prompt y de las condiciones de cache del proveedor.
- Los rangos de tokens son supuestos de planificación hasta disponer de varias corridas API reales.
