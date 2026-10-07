# Modelo Financiero FinOps
## Azure AI Foundry (Serverless / Provisioned / Managed Compute) vs. modelos Open Weight en GPU dedicada de Azure

| Campo | Valor |
|---|---|
| Tipo de documento | Modelo económico reproducible (TCO, $/M tokens y punto de equilibrio) |
| Fecha de corte de precios | 7 de octubre de 2026 |
| Región de referencia | East US 2 |
| Moneda | USD, precio de lista (*retail*) sin descuentos EA, MACC, CSP ni Savings Plan |
| Fuente de precios oficiales | Azure Retail Prices API (`prices.azure.com`), la misma fuente que alimenta la Azure Pricing Calculator. Incluye los catálogos de Foundry Models, Virtual Machines, Managed Compute, Log Analytics, Storage, Bandwidth, Load Balancer y Backup. |
| Audiencia | Dirección de TI, Finanzas, Arquitectura y CCoE/FinOps |

### Convención de etiquetas (obligatoria en todo el documento)

| Etiqueta | Significado |
|---|---|
| **[O] Oficial** | Precio publicado por Microsoft y obtenido de la Retail Prices API en la fecha de corte |
| **[S] Supuesto** | Valor sintético definido por este modelo. Debe validarse con datos reales del cliente o con benchmarks |
| **[D] Derivado** | Resultado calculado a partir de [O] y [S] con las fórmulas de la sección 2 |

> **Aviso.** Los modelos no son equivalentes en calidad: un Llama 70B autohospedado **no** sustituye funcionalmente a GPT-5.4. El modelo compara **costo por capacidad entregada**; la decisión final debe incluir pruebas de calidad (exactitud, *groundedness*, tasa de alucinación) sobre los casos de uso reales.

---

# 1. Supuestos

## 1.1 Caso base de demanda

| # | Parámetro | Valor | Tipo |
|---|---|---|---|
| B1 | Usuarios | 10 | [S] (dato del cliente) |
| B2 | Días laborables por mes | 22 | [S] |
| B3 | Prompts por usuario por día | 500 | [S] |
| B4 | Tokens promedio por prompt | 12,000 | [S] |
| B5 | Distribución input / output | 80% / 20% | [S] |
| B6 | Prompts diarios | 5,000 | [D] = B1 × B3 |
| B7 | Tokens diarios | 60,000,000 | [D] = B6 × B4 |
| B8 | **Tokens mensuales** | **1,320,000,000** (1.32B) | [D] = B7 × B2 |
| B9 | Tokens de input / mes | 1,056M (9,600 por prompt) | [D] |
| B10 | Tokens de output / mes | 264M (2,400 por prompt) | [D] |

## 1.2 Supuestos de comportamiento y operación

| # | Supuesto | Valor | Tipo | Comentario |
|---|---|---|---|---|
| S1 | % del input que se sirve desde *prompt cache* | 30% | [S] | Habitual en asistentes de código y RAG con contexto repetido. Si el modelo no tiene tarifa de caché, se cobra como input normal |
| S2 | Perfil de demanda **Laboral** | 22 días × 8 h = **176 h activas/mes**, factor pico **2.0×** | [S] | Equipos humanos (developers, analistas) |
| S3 | Perfil de demanda **24x7** | **730 h/mes**, factor pico **1.5×** | [S] | Agentes, *batch*, integraciones sistema a sistema |
| S4 | Utilización máxima de dimensionamiento GPU | 70% del throughput pico | [S] | Margen para latencia y colas |
| S5 | Velocidad de decodificación por sesión | 40 tokens/s | [S] | Latencia ≈ 2,400 / 40 + 2 s de *prefill* ≈ **62 s por prompt** |
| S6 | Tokens de razonamiento ocultos (escenario Razonamiento) | +100% sobre el output visible (factor 2×) | [S] | Se facturan como output |
| S7 | Costo FTE de operación (SRE/MLOps, *fully loaded*) | $12,000 / mes | [S] | Ajustar al país. En LATAM suele ser menor |
| S8 | Operación GPU dedicada (IaaS) | 0.40 FTE + 0.05 FTE por nodo adicional | [S] | Parches, drivers CUDA/ROCm, vLLM, escalamiento, guardias |
| S9 | Operación Managed Compute (Foundry) | 0.20 FTE + 0.03 FTE por nodo adicional | [S] | Microsoft opera la plataforma y el cliente opera el modelo |
| S10 | Operación Foundry Serverless / PTU | 0.10 FTE | [S] | Gobierno de cuotas, costos y evaluaciones |
| S11 | Ingeniería inicial amortizada a 36 meses | GPU: $1,250/mes ($45K únicos) · Managed Compute: $625/mes · Foundry: $250/mes | [S] | |
| S12 | Disaster Recovery GPU (*pilot light* + réplica de pesos) | 10% del costo GPU | [S] | |
| S13 | Contingencia | GPU / Managed Compute: 15% · Foundry: 5% | [S] | Riesgo de capacidad GPU vs. variación de precio por token |
| S14 | Telemetría de infraestructura por nodo GPU | 91 GB/mes (≈3 GB/día) | [S] | |
| S15 | Log de prompts y respuestas | 4 bytes/token | [S] | Aplica a ambas opciones |
| S16 | Almacenamiento por nodo | 1 disco Premium SSD P40 (2 TiB) + repositorio de pesos de 2 TB en Blob Hot | [S] | |
| S17 | Networking privado (Private Link, DNS, firewall; prorrateo) | $300 / mes | [S] | |
| S18 | Capacidad PTU (familia GPT-5.x) | 1 PTU ≈ 2,500 TPM de input equivalente; 1 token de output = 4 de input | [S] | **Validar con la calculadora de capacidad de Foundry.** Varía según el modelo |
| S19 | Mínimo de PTU Global | 15 PTU, en incrementos de 5 | [S] | Validar con la documentación vigente |

## 1.3 Throughput sintético de inferencia (Open Weight autohospedado)

Estimación de tipo *roofline* (cómputo de *prefill* + ancho de banda de *decode*) con vLLM y *continuous batching*, para la mezcla 9,600 tokens de entrada / 2,400 de salida. **Todo es [S] y debe reemplazarse por un benchmark real.**

| Config | Modelo de referencia | Precisión | Throughput total (tokens/s) |
|---|---|---|---|
| 1×A10 / 2×A10 | Clase 8B (no aloja 70B) | BF16 | 1,700 / 3,400 |
| 1×A100 80GB | Llama 3.3 70B | INT4 (AWQ) | 680 |
| 2×A100 | Llama 3.3 70B | BF16 (poca memoria para KV cache) | 1,050 |
| 4×A100 | Llama 3.3 70B | BF16, TP4 | 2,600 |
| 8×A100 | Llama 3.3 70B | 2 réplicas TP4 | 5,100 |
| 1×H100 NVL | Llama 3.3 70B | FP8 | 1,670 |
| 2×H100 NVL | Llama 3.3 70B | FP8, TP2 | 4,100 |
| 4×H100 | Llama 3.3 70B | FP8 | 9,600 |
| 8×H100 | Llama 3.3 70B | FP8, 4 réplicas TP2 | 19,000 |
| 8×H200 | Llama 3.3 70B / MoE clase DeepSeek-Kimi | FP8 | 21,500 / 9,000 |
| 8×MI300X | Llama 3.3 70B / MoE clase DeepSeek-Kimi | FP8 (ROCm) | 18,000 / 8,000 |

> Los modelos MoE de 600B a 1T parámetros (DeepSeek V3/V4, Kimi K2) requieren 8×H200 u 8×MI300X por réplica. No caben en 8×H100 (640 GB) en FP8.

## 1.4 Precios oficiales de modelos en Foundry [O]

Despliegue *Global Standard* (serverless), East US 2, USD por **1M de tokens**.

| Familia | Modelo | Input | Cached input | Output |
|---|---|---|---|---|
| GPT | GPT-5.4 nano | 0.20 | 0.02 | 1.25 |
| GPT | GPT-5.4 mini | 0.75 | 0.075 | 4.50 |
| GPT | GPT-5.4 | 2.50 | 0.25 | 15.00 |
| GPT | GPT-4.1 | 2.00 | 0.50 | 8.00 |
| GPT (razonamiento) | o3 | 2.00 | 0.50 | 8.00 |
| GPT (open weight) | gpt-oss-120B | 0.15 | — | 0.60 |
| DeepSeek | V4 Flash | 0.19 | 0.028 | 0.51 |
| DeepSeek | V4 Pro | 1.74 | 0.145 | 3.48 |
| DeepSeek | R1 | 1.35 | — | 5.40 |
| DeepSeek | V3.2 | 0.58 | — | 1.68 |
| Kimi | K2.5 Thinking | 0.60 | 0.10 | 3.00 |
| Kimi | K2.6 Thinking | 0.95 | 0.16 | 4.00 |
| Kimi | K2.7 Code | 0.95 | 0.19 | 4.00 |
| Qwen | Qwen3 32B (*solo medidor fine-tuned*) | 0.30 | — | 1.20 (+ $0.30/h de hosting) |
| Llama | Llama 3.3 70B | 0.71 | — | 0.71 |
| Llama | Llama 4 Maverick 17B | 0.25 | — | 1.00 |
| Mistral | Mistral Large 3 | 0.50 | — | 1.50 |
| Mistral | Codestral | 0.30 | — | 0.90 |

> **Qwen:** en la fecha de corte no se encontró un medidor serverless estándar para Qwen base. El medidor de Qwen3 32B *fine-tuned* se usa solo como **referencia [S]**.

## 1.5 Precios oficiales de otras modalidades de Foundry [O]

| Concepto | Precio oficial | Unidad |
|---|---|---|
| Provisioned Throughput (PTU) Global, por hora | $1.00 | PTU-hora |
| PTU Data Zone / Regional, por hora | $1.10 / $2.00 | PTU-hora |
| Reserva PTU Global, mensual | $260 | PTU-mes |
| Reserva PTU Global, anual | $2,652 | PTU-año (≈ $221/mes) |
| Managed Compute A100 80GB | $3.95 | GPU-hora |
| Managed Compute H100 80GB | $7.91 | GPU-hora |
| Managed Compute H200 141GB | $9.10 | GPU-hora |
| Managed Compute MI300 192GB | $7.91 | GPU-hora |

> **[D] Interpretación:** la tarifa de Managed Compute por GPU-hora es similar a la de la VM equivalente (A100: $3.95 vs. $3.673). Por eso el modelo la trata como un **costo de cómputo completo**. Confirmarlo en la Pricing Calculator antes de un compromiso.

## 1.6 Precios oficiales de VMs GPU [O]

Linux, pago por uso (PAYG), East US 2.

| GPU | 1 GPU | 2 GPU | 4 GPU | 8 GPU |
|---|---|---|---|---|
| **A10** | NV36ads_A10_v5 **$3.20/h** | NV72ads_A10_v5 **$6.52/h** | N/D (sin SKU) | N/D (sin SKU) |
| **A100** | NC24ads_A100_v4 **$3.673/h** | NC48ads_A100_v4 **$7.346/h** | NC96ads_A100_v4 **$14.692/h** | ND96amsr_A100_v4 **$32.77/h** |
| **H100** | NC40ads_H100_v5 **$6.98/h** | NC80adis_H100_v5 **$13.96/h** | N/D en VM → Managed Compute $31.64/h [D] | ND96isr_H100_v5 **$98.32/h** |
| **H200** | N/D en VM → Managed Compute $9.10/h [D] | Managed Compute $18.20/h [D] | Managed Compute $36.40/h [D] | ND96isr_H200_v5 **$84.80/h** |
| **MI300X** | N/D en VM → Managed Compute $7.91/h [D] | Managed Compute $15.82/h [D] | Managed Compute $31.64/h [D] | ND96isr_MI300X_v5 **$48.00/h** |

### Costo hora, mensual y anual por configuración

PAYG, operación 24x7 (730 h/mes y 8,760 h/año).

| Config | SKU | Costo/hora | Costo mensual | Costo anual | RI 1 año (equivalente mensual) [D] | RI 3 años (equivalente mensual) [D] |
|---|---|---|---|---|---|---|
| 1×A10 | NV36ads_A10_v5 | $3.20 | $2,336 | $28,032 | $1,495 | $1,028 |
| 2×A10 | NV72ads_A10_v5 | $6.52 | $4,760 | $57,115 | — | — |
| 1×A100 | NC24ads_A100_v4 | $3.673 | $2,681 | $32,175 | $1,753 | $995 |
| 2×A100 | NC48ads_A100_v4 | $7.346 | $5,363 | $64,351 | — | — |
| 4×A100 | NC96ads_A100_v4 | $14.692 | $10,725 | $128,702 | $7,011 | $3,980 |
| 8×A100 | ND96amsr_A100_v4 | $32.77 | $23,922 | $287,065 | — | — |
| 1×H100 | NC40ads_H100_v5 | $6.98 | $5,095 | $61,145 | — | — |
| 2×H100 | NC80adis_H100_v5 | $13.96 | $10,191 | $122,290 | $7,643 | $5,605 |
| 4×H100 | Managed Compute | $31.64 | $23,097 | $277,166 | — | — |
| 8×H100 | ND96isr_H100_v5 | $98.32 | $71,774 | $861,283 | $45,935 | $31,509 |
| 1/2/4×H200 | Managed Compute | $9.10 / $18.20 / $36.40 | $6,643 / $13,286 / $26,572 | $79,716 / $159,432 / $318,864 | — | — |
| 8×H200 | ND96isr_H200_v5 | $84.80 | $61,904 | $742,848 | $33,960 | $30,822 |
| 1/2/4×MI300X | Managed Compute | $7.91 / $15.82 / $31.64 | $5,774 / $11,549 / $23,097 | $69,292 / $138,583 / $277,166 | — | — |
| 8×MI300X | ND96isr_MI300X_v5 | $48.00 | $35,040 | $420,480 | $22,426 | $15,383 |

> Las reservas oficiales [O] son pagos totales: por ejemplo, ND96isr_MI300X_v5 cuesta $269,107 a 1 año y $553,772 a 3 años. "—" indica que no se encontró un precio de reserva en la región.

## 1.7 Precios oficiales de servicios de soporte [O]

| Servicio | Medidor | Precio |
|---|---|---|
| Azure Monitor (Log Analytics) | Analytics Logs Data Ingestion | $2.76 / GB (31 días de retención incluidos) |
| Azure Monitor (Log Analytics) | Retención adicional | $0.12 / GB-mes |
| Storage | Premium SSD P30 LRS / P40 LRS / P30 ZRS | $122.88 / $235.52 / $184.32 por mes |
| Storage | Blob Hot LRS | $0.0184 / GB-mes |
| Networking | Load Balancer Standard (5 reglas incluidas) | $0.025 / h + $0.005 / GB procesado |
| Networking | Egress a Internet (primer tramo, después de 100 GB) | $0.087 / GB |
| Networking | Transferencia entre regiones | $0.02 / GB |
| Backup | Azure VM Protected Instance | $10 / mes |
| Backup | Backup Standard ZRS Data Stored | $0.028 / GB-mes |

---

# 2. Fórmulas

## 2.1 Demanda y utilización

```
Tokens_mes            = Usuarios × Días × Prompts_usuario_día × Tokens_prompt
Prompts_mes           = Tokens_mes / Tokens_prompt
Horas_activas         = 176 (Laboral) | 730 (24x7)
Prompts/s promedio    = Prompts_mes / (Horas_activas × 3,600)
Tokens/s promedio     = Tokens_mes  / (Horas_activas × 3,600)
Tokens/s pico         = Tokens/s promedio × Factor_pico
Concurrencia pico     = Prompts/s pico × Latencia_por_prompt (62 s)
Utilización %         = Tokens/s pico / (Throughput_nodo × Nodos)
Nodos requeridos      = ⌈ Tokens/s pico / (Throughput_nodo × 70%) ⌉   (+1 si hay HA)
Riesgo de saturación  = Baja < 50% · Media 50–70% · Alta 70–85% · Crítica > 85%
```

## 2.2 Costo de modelo en Foundry (Serverless)

```
Input_no_cache  = Tokens_mes × 80% × (1 − 30%) × Precio_input
Input_cache     = Tokens_mes × 80% × 30% × Precio_cached
Output          = Tokens_mes × 20% × Factor_razonamiento × Precio_output
Costo_tokens    = Input_no_cache + Input_cache + Output

Precio_blended ($/M) = 0.8 × (0.7 × P_in + 0.3 × P_cache) + 0.2 × F_raz × P_out

Otros costos    = Monitor (tokens × 4 B × $2.76/GB) + 0.10 FTE + Ingeniería amortizada
Costo Modelo Mensual (TCO Foundry) = (Input + Output + Otros costos) × (1 + 5% contingencia)
```

## 2.3 Costo de Provisioned Throughput (PTU)

```
TPM_equivalente  = TPM_input_pico + 4 × TPM_output_pico
PTU              = max(15, ⌈ TPM_equivalente / 2,500 ⌉ redondeado a múltiplos de 5)
Costo PTU        = PTU × $1.00 × 730 (por hora) | PTU × $260 (reserva mensual) | PTU × $2,652 / 12 (reserva anual)
$/M efectivo PTU = (Costo PTU anual/12) / (Tokens reales servidos / 1M)
```

## 2.4 TCO de infraestructura GPU (autohospedado o Managed Compute)

```
GPU          = Precio_hora × Horas_encendido × Nodos          (730 h en 24x7 | 220 h en horario laboral)
Storage      = $235.52 × Nodos + 2,048 GB × $0.0184
Monitor      = 91 GB × $2.76 × Nodos + (Tokens × 4 B) × $2.76/GB
Networking   = $18.25 × Nodos (Load Balancer) + datos procesados + egress del output + $300 de red privada
Backup       = ($10 + 1,024 GB × $0.028) × Nodos
DR           = 10% × GPU
Operación    = $12,000 × (0.40 + 0.05 × (Nodos − 1))
Ingeniería   = $1,250
Contingencia = 15% × (suma anterior)

TCO = GPU + Storage + Networking + Monitor + Backup + DR + Operación + Ingeniería + Contingencia
```

## 2.5 Indicadores comparativos

```
Costo efectivo $/M     = TCO_mensual / (Tokens_mes / 1,000,000)
Diferencia %           = (TCO_GPU − TCO_Foundry) / TCO_Foundry
Punto de equilibrio (M tokens/mes) = (Costo_fijo_GPU − Costo_fijo_Foundry) / (Precio_blended_Foundry − Costo_variable_GPU_por_M)
Condición de existencia: TCO_GPU $/M a utilización máxima < Precio_blended_Foundry
Score ponderado        = 0.30·Costo + 0.20·Escalabilidad + 0.15·Operación + 0.15·Gobernanza + 0.10·Seguridad + 0.10·Flexibilidad
```

---

# 3. Matriz de escenarios

## 3.1 Escenarios de modelo

Precio *blended* por cada 1M de tokens con la mezcla 80/20 y 30% de caché.

| Escenario | Modelo representativo | Alternativas en la misma familia | $/M sin caché [D] | **$/M blended [D]** | Costo de tokens del caso base (1.32B) [D] |
|---|---|---|---|---|---|
| **Económico** | DeepSeek V4 Flash | gpt-oss-120B ($0.24), Llama 4 Maverick ($0.40), GPT-5.4 nano ($0.37) | $0.254 | **$0.215** | **$284** |
| **Balanceado** | GPT-5.4 mini | Kimi K2.7 Code ($1.38), Kimi K2.6 ($1.37), Mistral Large 3 ($0.70) | $1.500 | **$1.338** | **$1,766** |
| **Premium** | GPT-5.4 | GPT-4.1 ($2.84), DeepSeek V4 Pro ($1.71) | $5.000 | **$4.460** | **$5,887** |
| **Razonamiento** | o3 (output 2×) | DeepSeek R1, Kimi K2.6 Thinking | $4.800 | **$4.440** | **$5,861** |
| **Open Weight (serverless)** | Llama 3.3 70B | Mistral Large 3 ($0.70), DeepSeek V3.2 ($0.80), Qwen3 32B ref. ($0.48) | $0.710 | **$0.710** | **$937** |
| **Open Weight (autohospedado)** | Llama 3.3 70B en GPU | Qwen, Mistral o DeepSeek/Kimi en 8×H200/MI300X | — | Ver sección 5 | Ver sección 4 |

### Desglose del costo de modelo mensual en el caso base (1.32B tokens)

| Escenario | Input sin caché | Input con caché | Output | Otros (Monitor + 0.1 FTE + Ingeniería) | Contingencia 5% | **Costo Modelo Mensual** |
|---|---|---|---|---|---|---|
| Económico | $140 | $9 | $135 | $1,465 | $87 | **$1,836** |
| Balanceado | $554 | $24 | $1,188 | $1,465 | $162 | **$3,392** |
| Premium | $1,848 | $79 | $3,960 | $1,465 | $368 | **$7,719** |
| Razonamiento | $1,478 | $158 | $4,224 | $1,465 | $366 | **$7,692** |
| Open Weight serverless | $525 | $225 | $187 | $1,465 | $120 | **$2,522** |

> En volúmenes bajos, los **costos de gobierno y operación** pesan más que el costo de los tokens. Incluso en Foundry, el costo por token no es el único rubro.

## 3.2 Matriz de volumen: demanda y utilización

Dimensionamiento con el throughput sintético de la sección 1.3 y la utilización máxima de 70%.

### Perfil Laboral (22 días × 8 h, pico 2×)

| Escenario | Tokens/mes | Prompts/s prom. (pico) | Tokens/s prom. (pico) | Output tokens/s | Concurrencia pico | Utilización de 1 nodo 4×A100 | Riesgo con 1 nodo 4×A100 | Config óptima | Utilización óptima |
|---|---|---|---|---|---|---|---|---|---|
| XS | 100M | 0.013 (0.026) | 158 (316) | 32 | 2 | 12% | Baja | 1×(1×A100) | 46% Baja |
| S | 500M | 0.066 (0.132) | 789 (1,578) | 158 | 8 | 61% | Media | 1×(2×H100) | 38% Baja |
| M | 1B | 0.132 (0.263) | 1,578 (3,157) | 316 | 16 | 121% | **Crítica** | 3×(1×H100) | 63% Media |
| **Base** | **1.32B** | **0.174 (0.347)** | **2,083 (4,167)** | **417** | **22** | **160%** | **Crítica** | **2×(2×H100)** | **51% Media** |
| L | 5B | 0.658 (1.315) | 7,891 (15,783) | 1,578 | 82 | 607% | Crítica | 6×(2×H100) | 64% Media |
| XL | 10B | 1.315 (2.630) | 15,783 (31,566) | 3,157 | 163 | 1,214% | Crítica | 3×(8×MI300X) | 58% Media |
| XXL | 20B | 2.630 (5.261) | 31,566 (63,131) | 6,313 | 326 | 2,428% | Crítica | 6×(8×MI300X) | 58% Media |
| Enterprise | 50B | 6.576 (13.152) | 78,914 (157,828) | 15,783 | 815 | 6,070% | Crítica | 13×(8×MI300X) | 67% Media |

### Perfil 24x7 (730 h, pico 1.5×)

| Escenario | Tokens/mes | Tokens/s prom. (pico) | Concurrencia pico | Utilización de 1 nodo 4×A100 | Riesgo con 1 nodo 4×A100 | Config óptima | Utilización óptima |
|---|---|---|---|---|---|---|---|
| XS | 100M | 38 (57) | < 1 | 2% | Baja | 1×(1×A100) | 8% Baja |
| S | 500M | 190 (285) | 1 | 11% | Baja | 1×(1×A100) | 42% Baja |
| M | 1B | 381 (571) | 3 | 22% | Baja | 1×(1×H100) | 34% Baja |
| **Base** | **1.32B** | **502 (753)** | **4** | **29%** | **Baja** | **1×(1×H100)** | **45% Baja** |
| L | 5B | 1,903 (2,854) | 15 | 110% | Crítica | 1×(2×H100) | 70% Alta |
| XL | 10B | 3,805 (5,708) | 29 | 220% | Crítica | 2×(2×H100) | 70% Alta |
| XXL | 20B | 7,610 (11,416) | 59 | 439% | Crítica | 1×(8×MI300X) | 63% Media |
| Enterprise | 50B | 19,026 (28,539) | 147 | 1,098% | Crítica | 3×(8×MI300X) | 53% Media |

> **Hallazgo clave.** Con el perfil laboral, una GPU encendida 24x7 está ociosa alrededor del **76% del tiempo** (176 de 730 h). Además, el pico obliga a sobredimensionarla. Un solo nodo NC96ads A100 v4 **no** soporta el caso base en horario laboral (160% de utilización pico).

---

# 4. Resultados sintéticos

## 4.1 Tabla de costos mensuales (TCO) [D]

En la columna GPU se usa la configuración óptima (mínimo TCO) para cada volumen.

| Escenario | Foundry Económico | Foundry Balanceado | Foundry Premium | Foundry Razonamiento | Foundry OW serverless | Híbrido 80/20 * | GPU PAYG (demanda laboral) | GPU PAYG (demanda 24x7) | GPU RI 3 años (demanda 24x7) |
|---|---|---|---|---|---|---|---|---|---|
| XS (100M) | $1,546 | $1,664 | $1,992 | $1,990 | $1,598 | $1,635 | $11,364 | $11,364 | $9,231 |
| S (500M) | $1,641 | $2,231 | $3,870 | $3,859 | $1,901 | $2,087 | $20,869 | $11,369 | $9,236 |
| M (1B) | $1,760 | $2,939 | $6,217 | $6,196 | $2,280 | $2,651 | $29,951 | $14,429 | $11,816 |
| **Base (1.32B)** | **$1,836** | **$3,392** | **$7,719** | **$7,692** | **$2,522** | **$3,013** | **$35,086** | **$14,434** | **$11,820** |
| L (5B) | $2,710 | $8,605 | $24,995 | $24,890 | $5,308 | $7,167 | $91,959 | $20,926 | $15,125 |
| XL (10B) | $3,897 | $15,687 | $48,468 | $48,258 | $9,093 | $12,812 | $143,706 | $35,197 | $23,595 |
| XXL (20B) | $6,272 | $29,852 | $95,414 | $94,994 | $16,664 | $24,101 | $280,756 | $52,553 | $27,686 |
| Enterprise (50B) | $13,396 | $72,347 | $236,252 | $235,202 | $39,377 | $57,968 | $600,625 | $144,218 | $69,618 |

\* **Híbrido 80/20:** el 80% del tráfico se enruta al modelo Económico y el 20% al Premium. El *blended* resultante es de $1.064/M [D].

## 4.2 Tabla de costos anuales (TCO × 12) [D]

| Escenario | Foundry Económico | Foundry Balanceado | Foundry Premium | Foundry Razonamiento | Foundry OW serverless | Híbrido 80/20 | GPU PAYG (laboral) | GPU PAYG (24x7) | GPU RI 3 años (24x7) |
|---|---|---|---|---|---|---|---|---|---|
| XS | $18.6K | $20.0K | $23.9K | $23.9K | $19.2K | $19.6K | $136.4K | $136.4K | $110.8K |
| S | $19.7K | $26.8K | $46.4K | $46.3K | $22.8K | $25.0K | $250.4K | $136.4K | $110.8K |
| M | $21.1K | $35.3K | $74.6K | $74.4K | $27.4K | $31.8K | $359.4K | $173.2K | $141.8K |
| **Base** | **$22.0K** | **$40.7K** | **$92.6K** | **$92.3K** | **$30.3K** | **$36.2K** | **$421.0K** | **$173.2K** | **$141.8K** |
| L | $32.5K | $103.3K | $299.9K | $298.7K | $63.7K | $86.0K | $1,103.5K | $251.1K | $181.5K |
| XL | $46.8K | $188.2K | $581.6K | $579.1K | $109.1K | $153.7K | $1,724.5K | $422.4K | $283.1K |
| XXL | $75.3K | $358.2K | $1,145.0K | $1,139.9K | $200.0K | $289.2K | $3,369.1K | $630.6K | $332.2K |
| Enterprise | $160.8K | $868.2K | $2,835.0K | $2,822.4K | $472.5K | $695.6K | $7,207.5K | $1,730.6K | $835.4K |

## 4.3 Desglose del TCO GPU en el caso base (1.32B) [D]

| Rubro | Demanda laboral: 2× NC80adis H100 v5 (PAYG 24x7) | Demanda 24x7: 1× NC40ads H100 v5 (PAYG) |
|---|---|---|
| GPU | $20,382 | $5,095 |
| Storage | $509 | $273 |
| Monitor | $517 | $266 |
| Networking + Load Balancer | $337 | $318 |
| Backup | $77 | $39 |
| Disaster Recovery | $2,038 | $510 |
| Operación | $5,400 | $4,800 |
| Ingeniería | $1,250 | $1,250 |
| Contingencia 15% | $4,576 | $1,883 |
| **TCO mensual** | **$35,086** | **$14,434** |
| **TCO anual** | **$421,032** | **$173,208** |
| $/M efectivo | $26.58 | $10.93 |

## 4.4 Matriz comparativa con la estructura de ejemplo, conciliada con capacidad real

El ejemplo propuesto (Foundry = $1.50/M y GPU = $10,700 fijos) corresponde exactamente a **GPT-5.4 mini sin caché ($1.50/M [D])** frente a **1× NC96ads_A100_v4 ($10,725/mes [O])**. Sin embargo, supone implícitamente que **un solo nodo atiende hasta 20B tokens/mes**. Con el throughput sintético de 2,600 tokens/s del nodo 4×A100 para un modelo de 70B, su capacidad útil es de **≈3.2B tokens/mes con demanda 24x7** y **≈0.58B con demanda laboral**.

| Escenario | Tokens/mes | Foundry (solo tokens) | GPU directo, 4×A100 (24x7) | Nodos | TCO GPU (24x7) | TCO GPU (laboral) | Recomendación |
|---|---|---|---|---|---|---|---|
| XS | 100M | $150 | $10,725 | 1 | $21,540 | $21,540 | **Foundry** |
| S | 500M | $750 | $10,725 | 1 | $21,545 | $21,545 | **Foundry** |
| M | 1B | $1,500 | $10,725 | 1 | $21,551 | $36,434 (2 nodos) | **Foundry** |
| **Base** | **1.32B** | **$1,980** | **$10,725** | **1** | **$21,555** | **$51,320 (3 nodos)** | **Foundry** |
| L | 5B | $7,500 | $21,450 | 2 | $36,485 | $140,662 (9 nodos) | **Foundry** |
| XL | 10B | $15,000 | $42,901 | 4 | $66,314 | $274,668 (18 nodos) | **Foundry** (evaluar PTU) |
| XXL | 20B | $30,000 | $75,076 | 7 | $111,089 | $527,798 (35 nodos) | **Foundry / Híbrido** |
| Enterprise | 50B | $75,000 | $171,603 | 16 | $245,415 | $1,302,070 (87 nodos) | **Híbrido** (GPU solo con H200/MI300X y RI 3 años; ver sección 6) |

> **Conclusión de la conciliación.** Con A100 a precio de lista, la GPU **no alcanza el punto de equilibrio** frente a $1.50/M en ningún volumen. El ejemplo original solo es válido si un nodo tuviera capacidad ilimitada. La ventaja de la GPU aparece únicamente con hardware de mayor densidad (H200/MI300X), reserva a 3 años, demanda 24x7 sostenida y frente a modelos Premium.

---

# 5. Benchmark financiero

## 5.1 Costo efectivo por millón de tokens

TCO de 1 nodo dimensionado al 70% de utilización.

| Config | Capacidad útil/mes (laboral) | TCO $/M PAYG (laboral) | TCO $/M RI 3 años (laboral) | Capacidad útil/mes (24x7) | TCO $/M PAYG (24x7) | TCO $/M RI 3 años (24x7) |
|---|---|---|---|---|---|---|
| 1×A100 | 151M | $75.36 | $61.22 | 834M | $13.64 | $11.08 |
| 4×A100 | 577M | $37.37 | $22.57 | 3.19B | $6.77 | $4.09 |
| 8×A100 | 1.13B | $33.82 | — | 6.26B | $6.13 | — |
| 1×H100 | 370M | $38.94 | — | 2.05B | $7.05 | — |
| 2×H100 | 909M | $22.96 | $16.58 | 5.03B | $4.16 | $3.01 |
| 8×H100 | 4.21B | $23.45 | $11.36 | 23.3B | $4.25 | $2.07 |
| 8×H200 | 4.77B | $18.11 | $9.86 | 26.4B | $3.28 | $1.79 |
| **8×MI300X** | 3.99B | $13.11 | $6.88 | 22.1B | **$2.38** | **$1.26** |
| Managed Compute 2×H100 | 909M | $21.02 | — | 5.03B | $3.81 | — |
| Managed Compute 4×H100 | 2.13B | $15.85 | — | 11.8B | $2.88 | — |
| Managed Compute 8×H200 | 4.77B | $15.06 | — | 26.4B | $2.73 | — |
| Managed Compute 8×MI300 | 3.99B | $15.78 | — | 22.1B | $2.86 | — |
| **Foundry Económico** | Ilimitada | **$0.215 + overhead** | | | | |
| **Foundry OW serverless (Llama 70B)** | Ilimitada | **$0.71** | | | | |
| **Foundry Balanceado** | Ilimitada | **$1.338** | | | | |
| **Foundry Premium / Razonamiento** | Ilimitada | **$4.46 / $4.44** | | | | |
| **PTU GPT-5.x, reserva anual, 100% de utilización** [D] | Según PTU | **≈ $3.23** | | | | |

## 5.2 Sensibilidad 1: precio de Foundry (Bajo, Medio, Alto)

Costo mensual solo de tokens.

| Volumen | **Bajo** ($0.215/M, Económico) | **Medio** ($1.338/M, Balanceado) | **Alto** ($4.46/M, Premium) | Mejor GPU, RI 3 años (24x7) |
|---|---|---|---|---|
| XS | $22 | $134 | $446 | $9,231 |
| M | $215 | $1,338 | $4,460 | $11,816 |
| Base | $284 | $1,766 | $5,887 | $11,820 |
| L | $1,076 | $6,690 | $22,300 | $15,125 |
| XL | $2,151 | $13,380 | $44,600 | $23,595 |
| XXL | $4,302 | $26,760 | $89,200 | $27,686 |
| Enterprise | $10,756 | $66,900 | $223,000 | $69,618 |

**Lectura:**
- Con precio **Bajo**, la GPU nunca compite.
- Con precio **Medio**, la GPU empata solo a partir de ≈20B tokens/mes con demanda 24x7 y RI a 3 años.
- Con precio **Alto**, la GPU gana desde ≈2.5B tokens/mes (24x7 con RI a 3 años) o ≈4B (24x7 PAYG).

## 5.3 Sensibilidad 2: utilización de GPU

TCO $/M de 1 nodo con demanda 24x7, en formato PAYG / RI 3 años.

| Config | 20% | 40% | 60% | 80% | 90% |
|---|---|---|---|---|---|
| 4×A100 | $15.77 / $9.53 | $7.89 / $4.77 | $5.27 / $3.19 | $3.95 / $2.39 | $3.52 / $2.13 |
| 2×H100 | $9.69 / $7.00 | $4.85 / $3.51 | $3.24 / $2.34 | $2.43 / $1.76 | $2.16 / $1.57 |
| 8×H200 | $7.65 / $4.17 | $3.83 / $2.09 | $2.56 / $1.40 | $1.92 / $1.05 | $1.71 / $0.94 |
| 8×MI300X | $5.54 / $2.91 | $2.78 / $1.46 | $1.86 / $0.98 | $1.39 / $0.74 | **$1.24 / $0.66** |

**Lectura:**
- Ninguna configuración baja de **$0.66/M**, incluso al 90% de utilización con RI a 3 años. Por lo tanto, ningún escenario autohospedado iguala al Económico ($0.215) ni al Open Weight serverless ($0.71) en Foundry, salvo 8×MI300X con RI a 3 años y ≥90% de utilización sostenida frente al Open Weight serverless.
- Frente al Balanceado ($1.34), la GPU solo compite con RI a 3 años y utilización ≥60% (8×MI300X) o ≥80% (8×H200).
- Frente al Premium ($4.46), compite desde ≈40% de utilización en H100, H200 o MI300X.

## 5.4 Sensibilidad 3 y 4: horario de operación y alta disponibilidad

TCO mensual PAYG con la configuración óptima.

| Volumen | Demanda | Encendido de VM | HA = No | HA = Sí (N+1) | Δ por HA |
|---|---|---|---|---|---|
| Base | Laboral | 24x7 | $35,086 | $45,477 | +30% |
| Base | Laboral | Horario laboral (220 h) | $17,073 | $22,273 | +30% |
| Base | 24x7 | 24x7 | $14,434 | $20,794 | +44% |
| XL | Laboral | 24x7 | $143,706 | $177,262 | +23% |
| XL | Laboral | Horario laboral (220 h) | $50,804 | $65,478 | +29% |
| XL | 24x7 | 24x7 | $35,197 | $49,403 | +40% |
| Enterprise | Laboral | 24x7 | $600,625 | $646,266 | +8% |
| Enterprise | Laboral | Horario laboral (220 h) | $198,052 | $212,725 | +7% |
| Enterprise | 24x7 | 24x7 | $144,218 | $163,567 | +13% |

**Lectura:**
- Apagar la GPU fuera del horario reduce el TCO entre **51% y 67%**. Sin embargo, aumenta el riesgo de no obtener capacidad GPU al reencender. Se mitiga con *Capacity Reservation*, que se cobra aun con la VM apagada.
- La HA añade entre +30% y +44% en volúmenes bajos. **En Foundry, la HA ya está incluida en el precio.**

## 5.5 Provisioned Throughput (PTU) frente a Serverless (GPT-5.4)

| Volumen | Demanda | PTU requeridas [D] | PTU con reserva anual (equivalente/mes) | GPT-5.4 serverless (tokens) | Resultado |
|---|---|---|---|---|---|
| Base | Laboral | 160 | $35,360 | $5,887 | Serverless (6×) |
| Base | 24x7 | 30 | $6,630 | $5,887 | Serverless (+13%) |
| XL | 24x7 | 220 | $48,620 | $44,600 | Serverless (+9%) |
| XXL | 24x7 | 440 | $97,240 | $89,200 | Serverless (+9%) |
| Enterprise | 24x7 | 1,100 | $243,100 | $223,000 | Serverless (+9%) |

**Punto de equilibrio PTU [D].** La PTU con reserva anual cuesta ≈ **$3.23/M al 100% de utilización**. Por lo tanto, solo supera a GPT-5.4 serverless ($4.46/M) cuando la **utilización sostenida supera ~72%**. Su justificación principal es **latencia y throughput garantizados**, no el ahorro. Esto depende del supuesto S18.

## 5.6 Ranking de escenarios

### Caso base (1.32B tokens/mes, demanda laboral)

| Rank | Escenario | TCO mensual | TCO anual | $/M efectivo |
|---|---|---|---|---|
| 1 | Foundry Económico (DeepSeek V4 Flash) | $1,836 | $22.0K | $1.39 |
| 2 | Foundry Open Weight serverless (Llama 3.3 70B) | $2,522 | $30.3K | $1.91 |
| 3 | Híbrido 80/20 (Económico + Premium) | $3,013 | $36.2K | $2.28 |
| 4 | Foundry Balanceado (GPT-5.4 mini) | $3,392 | $40.7K | $2.57 |
| 5 | Foundry Razonamiento (o3) | $7,692 | $92.3K | $5.83 |
| 6 | Foundry Premium (GPT-5.4) | $7,719 | $92.6K | $5.85 |
| 7 | GPU dedicada con RI 3 años (2×(2×H100)) | $23,484 | $281.8K | $17.79 |
| 8 | GPU dedicada PAYG (2×(2×H100)) | $35,086 | $421.0K | $26.58 |
| 9 | PTU GPT-5.x (160 PTU, reserva anual) | ≈ $35,360 + overhead | ≈ $424K | ≈ $27 |

### Enterprise (50B tokens/mes, demanda 24x7)

| Rank | Escenario | TCO mensual | $/M efectivo |
|---|---|---|---|
| 1 | Foundry Económico | $13,396 | $0.27 |
| 2 | Foundry Open Weight serverless | $39,377 | $0.79 |
| 3 | Híbrido 80/20 | $57,968 | $1.16 |
| 4 | GPU dedicada 3×(8×MI300X) con RI 3 años | $69,618 | $1.39 |
| 5 | Foundry Balanceado | $72,347 | $1.45 |
| 6 | GPU dedicada 3×(8×MI300X) PAYG | $144,218 | $2.88 |
| 7 | Foundry Premium / Razonamiento | $236,252 / $235,202 | $4.73 |
| 8 | PTU GPT-5.x (1,100 PTU, reserva anual) | ≈ $243,100 + overhead | ≈ $4.9 |

## 5.7 Evaluación multicriterio

La recomendación no se basa solo en costo. Puntaje de 1 a 5; el costo varía según la banda de volumen.

| Criterio (peso) | Foundry Serverless | Foundry Provisioned (PTU) | Foundry Managed Compute | GPU Dedicada (IaaS) | Híbrido |
|---|---|---|---|---|---|
| Escalabilidad (20%) | 5 | 4 | 3 | 2 | 4 |
| Operación (15%) | 5 | 5 | 4 | 1 | 3 |
| Gobernanza (15%) | 5 | 5 | 4 | 2 | 4 |
| Seguridad (10%) | 4 | 5 | 4 | 3 | 4 |
| Flexibilidad (10%) | 3 | 2 | 4 | 5 | 5 |
| *Subtotal sin costo (70%)* | *3.20* | *3.00* | *2.60* | *1.65* | *2.75* |
| Costo, banda A: ≤ 1.32B (30%) | 5 | 1 | 2 | 1 | 3 |
| **Score banda A** | **4.70** | 3.30 | 3.20 | 1.95 | 3.65 |
| Costo, banda B: 5–10B (30%) | 4 | 2 | 3 | 3 | 4 |
| **Score banda B** | **4.40** | 3.60 | 3.50 | 2.55 | 3.95 |
| Costo, banda C: ≥ 20B 24x7 (30%) | 3 | 3 | 4 | 4 | 5 |
| **Score banda C** | 4.10 | 3.90 | 3.80 | 2.85 | **4.25** |

---

# 6. Punto de equilibrio

## 6.1 Punto de equilibrio de 1 nodo frente a cada escenario de Foundry

Demanda 24x7. **(×)** = el punto de equilibrio supera la capacidad útil del nodo, por lo que no existe con ese hardware. Al escalar nodos, el costo por millón se mantiene igual o sube.

| Config | Capacidad útil/mes | vs. Económico ($0.215) | vs. OW serverless ($0.71) | vs. Balanceado ($1.338) | vs. Premium ($4.46) | vs. Razonamiento ($4.44) |
|---|---|---|---|---|---|---|
| 4×A100 PAYG | 3.2B | 89B (×) | 27B (×) | 14B (×) | 4.3B (×) | 4.3B (×) |
| 4×A100 RI 3 años | 3.2B | 51B (×) | 15B (×) | 8.2B (×) | **2.5B ✔** | **2.5B ✔** |
| 2×H100 PAYG | 5.0B | 86B (×) | 26B (×) | 14B (×) | **4.1B ✔** | **4.1B ✔** |
| 2×H100 RI 3 años | 5.0B | 60B (×) | 18B (×) | 9.6B (×) | **2.9B ✔** | **2.9B ✔** |
| 8×H100 RI 3 años | 23.3B | 206B (×) | 62B (×) | 33B (×) | **9.9B ✔** | **9.9B ✔** |
| 8×H200 PAYG | 26.4B | 377B (×) | 114B (×) | 60B (×) | **18.1B ✔** | **18.2B ✔** |
| 8×H200 RI 3 años | 26.4B | 202B (×) | 61B (×) | 32B (×) | **9.7B ✔** | **9.7B ✔** |
| 8×MI300X PAYG | 22.1B | 226B (×) | 68B (×) | 36B (×) | **10.8B ✔** | **10.9B ✔** |
| **8×MI300X RI 3 años** | 22.1B | 115B (×) | 35B (×) | **18.5B ✔** | **5.5B ✔** | **5.6B ✔** |

## 6.2 Lectura ejecutiva del punto de equilibrio

| Comparador en Foundry | ¿Existe punto de equilibrio con GPU dedicada? | Volumen mínimo | Condiciones |
|---|---|---|---|
| Económico (DeepSeek V4 Flash) | **No** | — | El costo de la GPU nunca baja de $0.66/M |
| Open Weight serverless (mismo modelo) | **No, en la práctica** | — | Solo 8×MI300X con RI a 3 años y ≥90% de utilización sostenida |
| Balanceado (GPT-5.4 mini) | **Sí, con condiciones** | **≈ 18.5B tokens/mes** | Demanda 24x7, 8×MI300X, RI a 3 años |
| Premium / Razonamiento | **Sí** | **≈ 2.5B–5.5B (RI 3 años) · ≈ 4–11B (PAYG)** | Demanda 24x7 y aceptación de un modelo open weight como sustituto de calidad |
| Cualquier modelo con **demanda laboral** | **No**, con VMs 24x7 | — | El TCO mínimo es de $6.88/M. Se requiere encender las VMs solo en horario laboral (≈ $3.96/M en Enterprise) |

---

# 7. Recomendaciones ejecutivas

## 7.1 Veredicto para el caso base (10 usuarios, 1.32B tokens/mes)

**Recomendado: Azure AI Foundry Serverless.**

- El costo anual va de **$22K a $93K** según el modelo, frente a **$282K (RI 3 años) a $421K (PAYG)** de GPU dedicada con demanda laboral.
- La GPU dedicada es entre **3× y 19× más cara**, y con un solo nodo 4×A100 tendría **riesgo de saturación crítico** (160% en el pico laboral).
- Foundry obtiene el mejor score multicriterio (**4.70 / 5**).

## 7.2 Recomendación por modalidad

| Modalidad | Cuándo recomendarla | Volumen orientativo | Disparadores |
|---|---|---|---|
| **Recomendado Foundry (Serverless)** | Opción por defecto. Demanda variable, horario laboral, exploración y múltiples modelos | XS a XL con cualquier perfil; cualquier volumen con modelos Económicos u Open Weight | Pago por uso, HA y gobernanza incluidos, sin riesgo de capacidad GPU |
| **Recomendado Provisioned (PTU)** | Cargas productivas críticas con SLA de latencia sobre modelos GPT Premium | ≥ 10B/mes en 24x7 con utilización sostenida > 72% | Latencia predecible, aislamiento de capacidad. **No** se justifica solo por ahorro |
| **Recomendado Managed Compute** | Modelos open weight personalizados (*fine-tuned*, versiones o cuantizaciones no disponibles en serverless) sin equipo de IaaS | ≥ 5B/mes en 24x7 ($2.7–3.8/M) | Control del modelo con operación compartida con Microsoft |
| **Recomendado GPU Dedicada** | Soberanía o aislamiento total, pesos propietarios, *stack* de inferencia personalizado y equipo MLOps existente | ≥ 20B/mes en 24x7, H200/MI300X, RI a 3 años y utilización ≥ 60% | Requisitos regulatorios o técnicos. **Rara vez** se justifica solo por costo |
| **Recomendado Híbrido** | Escala Enterprise con mezcla de casos de uso | ≥ 20B/mes | Ruteo por complejidad: 80% a modelo Económico u Open Weight (serverless o Managed Compute) y 20% a Premium o Razonamiento. Ahorro de **≈ 75%** frente a 100% Premium |

## 7.3 Matriz de recomendación por volumen

| Escenario | Tokens/mes | Demanda laboral | Demanda 24x7 |
|---|---|---|---|
| XS | 100M | Foundry Serverless | Foundry Serverless |
| S | 500M | Foundry Serverless | Foundry Serverless |
| M | 1B | Foundry Serverless | Foundry Serverless |
| **Base** | **1.32B** | **Foundry Serverless** | **Foundry Serverless** |
| L | 5B | Foundry Serverless | Foundry Serverless (evaluar Managed Compute si el modelo es propio) |
| XL | 10B | Foundry Serverless + ruteo Híbrido | Híbrido / evaluar PTU para Premium |
| XXL | 20B | Híbrido | Híbrido / evaluar GPU MI300X/H200 con RI a 3 años si se reemplaza un modelo Balanceado o Premium |
| Enterprise | 50B | Híbrido (Foundry + Managed Compute) | Híbrido o GPU dedicada con RI a 3 años (empate frente a Balanceado; ahorro de 71% frente a Premium) |

## 7.4 Mensajes clave para el comité

1. **El token más barato de Azure se compra, no se fabrica.** Con los precios de lista, ningún escenario de GPU autohospedada iguala el costo de los modelos Económicos u Open Weight servidos por Foundry.
2. **La GPU dedicada solo gana contra modelos Premium**, con demanda 24x7 sostenida, hardware de alta densidad (H200/MI300X) y compromiso de 3 años. Además, implica aceptar un modelo open weight como sustituto de calidad.
3. **La utilización define la economía.** Una GPU al 20% cuesta 4× más por token que al 80%. Los equipos humanos que trabajan en horario laboral dejan la GPU ociosa alrededor del 76% del tiempo.
4. **La operación pesa más que el hardware en volúmenes bajos.** En el caso base, la operación, la ingeniería, el DR y la contingencia suman ~40% del TCO de la GPU.
5. **PTU es un instrumento de SLA, no de ahorro**, salvo que la utilización sostenida supere ~72%.
6. **La palanca de mayor impacto es el ruteo de modelos (Híbrido).** Enviar el 80% del tráfico a un modelo Económico reduce el costo ~75% frente a usar solo Premium, sin invertir en infraestructura.

## 7.5 Próximos pasos sugeridos

| # | Acción | Responsable | Resultado |
|---|---|---|---|
| 1 | Desplegar en Foundry Serverless y medir durante 30 días los tokens reales, el % de caché, el factor pico y la mezcla input/output | Plataforma IA + FinOps | Reemplazar S1–S5 por datos reales |
| 2 | Evaluar calidad con Foundry Evaluations: Económico, Balanceado y Premium sobre los casos de uso reales | Arquitectura + negocio | Definir qué % del tráfico puede ir a modelos Económicos |
| 3 | Ejecutar un benchmark de throughput de 1 semana en Managed Compute (H100/H200/MI300) con el modelo open weight candidato | Ingeniería | Reemplazar los supuestos de la sección 1.3 |
| 4 | Recalcular este modelo con precios EA/MACC y Savings Plan | FinOps | Punto de equilibrio con precio neto |
| 5 | Revisión trimestral: el precio por token baja con cada generación de modelos y el hardware GPU tiene ciclos de 3 años | CCoE | Evitar *lock-in* financiero en reservas a 3 años |

---

### Notas metodológicas

- Los precios **[O]** se obtuvieron de la Azure Retail Prices API el 7 de octubre de 2026 (East US 2, Linux, despliegue Global Standard para modelos) y pueden cambiar sin aviso. Antes de cualquier compromiso, deben confirmarse en la Azure Pricing Calculator y en la página de precios de Azure AI Foundry.
- No incluye: impuestos, soporte Microsoft, licencias de software de inferencia comercial (por ejemplo, NVIDIA AI Enterprise para NIM), costo de oportunidad del capital, *fine-tuning* ni almacenamiento de datasets de entrenamiento.
- Los resultados **[D]** son reproducibles al sustituir cualquier supuesto **[S]** en las fórmulas de la sección 2.
