# Comparativo de Costos: Azure AI Foundry vs Infraestructura Dedicada (GPU)
 
## Objetivo
 
Evaluar económicamente cuándo conviene:
 
1. Consumir modelos LLM mediante Azure AI Foundry (Serverless / AI Credits).
2. Hospedar un modelo Open Weight (Kimi, DeepSeek, Qwen, Llama, Mistral, etc.) en infraestructura propia de Azure.
 
---
 
# Escenario Base
 
## Supuestos
 
| Parámetro | Valor |
|------------|---------|
| Usuarios | 10 Developers |
| Días laborales por mes | 22 |
| Prompts por usuario por día | 500 |
| Tokens promedio por prompt | 12,000 |
| Total prompts diarios | 5,000 |
| Total tokens diarios | 60,000,000 |
| Total tokens mensuales | 1,320,000,000 |
 
---
 
# Opción A: Azure AI Foundry (Serverless)
 
## Modelo
 
Ejemplo:
 
- Kimi K2
- DeepSeek
- Qwen
- Llama
- GPT
 
Consumidos mediante:
 
```text
Azure AI Foundry
↓
Serverless Endpoint
↓
Pago por consumo
```
 
---
 
## Fórmula
 
```text
Costo Mensual =
(Tokens Mensuales / 1,000,000)
× Precio por millón de tokens
```
 
---
 
## Escenario 1
 
Supongamos:
 
```text
1 USD / millón tokens
```
 
Consumo:
 
```text
1,320 millones tokens
```
 
Resultado:
 
```text
$1,320 USD / mes
```
 
---
 
## Ventajas
 
- Sin administración de infraestructura.
- Sin gestión de GPUs.
- Escalamiento automático.
- Observabilidad incluida.
- Guardrails incluidos.
- Microsoft Entra ID.
- Alta disponibilidad administrada.
- Sin capacidad reservada.
 
---
 
## Desventajas
 
- Costo variable.
- Dependencia del consumo.
- Puede volverse costoso a muy alta escala.
 
---
 
# Opción B: GPU Dedicada (Self-Hosted)
 
## Arquitectura
 
```text
Azure VM GPU
↓
vLLM / TGI / NIM
↓
Kimi / DeepSeek / Llama
↓
Endpoint propio
```
 
---
 
## Ejemplo
 
VM seleccionada:
 
```text
NC96ads A100 v4
```
 
Características:
 
```text
96 vCPU
880 GB RAM
A100 GPU
```
 
Costo estimado:
 
```text
$14.692 USD/hora
```
 
Operación continua:
 
```text
730 horas / mes
```
 
Resultado:
 
```text
$10,725 USD / mes
```
 
Costo anual:
 
```text
≈ $128,700 USD / año
```
 
---
 
## Costos que normalmente NO están incluidos
 
Agregar:
 
- Azure Monitor
- Networking
- Storage
- Backups
- Alta disponibilidad
- Disaster Recovery
- Ingeniería de operación
 
---
 
## Ventajas
 
- Control total del modelo.
- Control de cuantización.
- Control del contexto.
- Fine-tuning avanzado.
- Posibilidad de usar modelos personalizados.
- Costos predecibles.
 
---
 
## Desventajas
 
- Pago fijo aun sin consumo.
- Administración de infraestructura.
- Escalamiento manual.
- Riesgo de capacidad GPU.
- Operación continua.
 
---
 
# Comparación Directa
 
| Concepto | Foundry Serverless | GPU Dedicada |
|-----------|-----------|-----------|
| Inversión inicial | $0 | $0 |
| Costo fijo mensual | $0 | $10,725 |
| Pago por uso | Sí | No |
| Escala para 10 usuarios | Excelente | Sobre dimensionado |
| Escala para 100 usuarios | Excelente | Evaluar |
| Escala para 1,000 usuarios | Evaluar | Posible ventaja |
| Operación | Microsoft | Cliente |
| Parches | Microsoft | Cliente |
| Observabilidad | Incluida | Implementar |
| Alta disponibilidad | Incluida | Diseñar |
| Gobernanza | Incluida | Implementar |
| Entra ID | Nativo | Implementar |
| Riesgo de capacidad GPU | Bajo | Alto |
 
---
 
# Punto de Equilibrio
 
## Fórmula
 
```text
Punto de Equilibrio (millones de tokens)
 
=
Costo Mensual GPU
/
Costo por Millón de Tokens
```
 
---
 
## Ejemplo
 
Si:
 
```text
GPU = $10,725 USD / mes
```
 
y
 
```text
Costo = $1 USD / millón tokens
```
 
Entonces:
 
```text
10,725 millones tokens / mes
```
 
sería aproximadamente el punto donde ambas opciones cuestan lo mismo.
 
---
 
# Escenarios Recomendados
 
## Escenario Conservador
 
```text
100M tokens / mes
```
 
Recomendación:
 
```text
Foundry Serverless
```
 
---
 
## Escenario Medio
 
```text
1B tokens / mes
```
 
Recomendación:
 
```text
Foundry Serverless
```
 
Evaluar Provisioned Throughput.
 
---
 
## Escenario Alto
 
```text
10B tokens / mes
```
 
Recomendación:
 
```text
Comparar Foundry vs GPU dedicada
```
 
---
 
## Escenario Muy Alto
 
```text
20B+ tokens / mes
```
 
Evaluar:
 
```text
Managed Compute
o
GPU dedicada
```
 
---
 
# Framework de Evaluación
 
## Caso de Uso
 
- Clasificación de tickets
- Generación de código
- RAG
- Agentes
- Document Intelligence
- Chatbots
 
---
 
## Métricas
 
### Calidad
 
- Exactitud
- Hallucination Rate
- Groundedness
 
### Rendimiento
 
- Latencia
- Throughput
- Concurrencia
 
### Costos
 
- Tokens
- GPU
- Storage
- Networking
 
### Operación
 
- SLA
- Observabilidad
- Seguridad
- Gobernanza
 
---
 
# Recomendación Inicial
 
Para clientes que todavía están explorando:
 
```text
Azure AI Foundry
+
Modelo Serverless
+
Benchmark de consumo real
```
 
Antes de invertir en infraestructura GPU dedicada.
 
La decisión de GPU dedicada debe basarse en:
 
- Consumo real validado.
- Necesidad de fine-tuning.
- Control de pesos.
- Restricciones regulatorias.
- Justificación económica demostrable.

Este documento te sirve como plantilla base para agregar después comparativos específicos de:

Kimi vs DeepSeek
Kimi vs GPT-5
Serverless vs Provisioned Throughput
A100 vs H100 vs H200 vs MI300
vLLM vs Foundry Managed Compute
Copilot Credits vs Azure AI Credits
RAG vs Fine-Tuning vs Distillation.

---

# Selección de modelos y arquitectura multimodal

Esta sección complementa el comparativo con criterios para una evaluación inicial. Los modelos indicados son **candidatos, no equivalentes ni ganadores garantizados**. La disponibilidad, versión, modalidad, API, región y estado (GA o preview) deben confirmarse en la oferta concreta antes de diseñar o cotizar. Los nombres y capacidades de los catálogos cambian.

## Candidatos por caso de uso

| Caso | Candidatos administrados | Alternativas de pesos disponibles | Criterio de evaluación |
|---|---|---|---|
| Clasificar tickets y correos de texto | GPT-4.1 mini; GPT-5 nano/mini; DeepSeek Flash | Qwen Instruct; Llama pequeño | Exactitud por categoría, abstenciones y costo por ticket correctamente resuelto. |
| RAG de políticas y procedimientos | GPT-4.1 mini para consultas habituales; GPT-4.1 o GPT-5 para casos complejos | Qwen Instruct; Llama 3.3 70B | Medir recuperación, groundedness y precisión de citas además de la respuesta. Embeddings (por ejemplo, text-embedding-3-small/large), indexación y recuperación se contabilizan aparte. |
| Revisión de contratos | GPT-4.1/GPT-5 con razonamiento; Claude Sonnet/Opus | Modelos compatibles de Qwen o Llama, sujetos a evaluación | Requiere revisión humana y criterios legales definidos; el modelo no sustituye asesoría ni aprobación jurídica. |
| Generación de código | GPT-4.1; Codex; Claude Sonnet | Qwen Coder | Medir pruebas aprobadas, seguridad y mantenibilidad en el repositorio objetivo; no usar solo preferencia subjetiva. |
| Debugging y razonamiento técnico | o3; GPT-5 con razonamiento; Claude Opus | DeepSeek; Qwen | Comparar resolución reproducible, corrección de pruebas y latencia en problemas representativos. |
| Interpretar capturas de pantalla | GPT-4.1/mini; GPT-5 con visión (según versión); Llama 4 Maverick | Qwen-VL; modelos Llama con visión | Probar OCR visual, comprensión de interfaz y exactitud con las resoluciones y formatos reales. |
| Facturas y documentos PDF | Azure AI Document Intelligence; Azure AI Content Understanding; Mistral Document AI | OCR más un modelo de visión compatible | Comparar extracción de campos, tablas, páginas difíciles y costo por documento correcto; considerar revisión humana. |
| Transcripción | Azure AI Speech; Whisper; modelos de transcripción disponibles en Foundry | Whisper de pesos abiertos | Evaluar WER, idiomas, ruido, diarización y latencia con audio real. |
| Voz conversacional | GPT Realtime; o Speech + LLM + TTS | ASR + LLM + TTS de pesos disponibles | Medir latencia extremo a extremo, calidad de turnos, interrupciones, idiomas y costo por minuto útil. |
| Análisis de video | Content Understanding o Video Indexer + LLM | Modelo de video compatible o pipeline de frames + transcripción | Para eventos temporales se necesitan timestamps y cobertura; frames aislados no equivalen al video completo. |
| Generación de imágenes | GPT Image; FLUX | Variantes de pesos disponibles, según licencia | Verificar derechos de uso, calidad, política de contenido y costo por activo aceptado. |
| Generación de video | Sora-2, según documentación y estado preview | Modelos de video de pesos disponibles, según licencia | Confirmar acceso, límites, derechos y costo en la oferta concreta; no asumir disponibilidad GA. |

## Modalidades de entrada y salida: validar por versión

La modalidad describe las entradas y salidas que admite una versión y un endpoint específicos; no implica que todos los despliegues ofrezcan idénticas capacidades. En particular, audio o video pueden requerir servicios de preprocesamiento.

| Familia u oferta | Modalidades documentadas/esperadas para evaluar | Límite importante |
|---|---|---|
| GPT-4.1 y GPT-4.1 mini | Texto e imagen de entrada → texto de salida | Para audio o video, preprocesar y enviar transcripción, texto o imágenes al endpoint compatible; no asumir entrada nativa de audio/video. |
| GPT-5 con visión, o3 y o4-mini | Texto e imagen según versión → texto | Confirmar la ficha de la versión; no inferir soporte de audio o video por ser multimodal en otro sentido. |
| Claude Sonnet y Opus | Texto e imagen de entrada → texto | Audio y video requieren preprocesamiento salvo que la oferta y API específicas indiquen lo contrario. |
| DeepSeek V3.2/V4 Flash/Pro en Azure | Texto → texto | No asumir visión, audio ni video para estas ofertas. |
| Kimi K2.5/K2.6/K2.7 Code | Texto e imagen según la oferta consultada → texto | Ofertas consultadas en preview; confirmar runtime, versión y modalidad. No asumir audio ni video. |
| Llama 3.3 70B | Texto → texto | No tratarlo como modelo de visión. |
| Llama 4 Maverick | Texto e imagen → texto | Confirmar modelo, versión y endpoint ofertados. |
| Mistral Large 3 y Medium 3.5 | Texto e imagen según ofertas consultadas | Ofertas consultadas en preview; validar estado y modalidad exactos. |
| Mistral Document AI | Imagen/PDF → texto o Markdown | Es una oferta orientada a documentos, no un modelo general de video. |
| Qwen Instruct, Coder, VL y Omni | Depende de la variante | Son familias/ofertas distintas. Confirmar modelo, modalidad, runtime y disponibilidad exactos; no afirmar que Qwen-VL está disponible en serverless sin evidencia de la oferta. |
| GPT Audio/Realtime | Audio y texto según la versión → audio y/o texto | Revisar las modalidades exactas de la versión, API y endpoint seleccionados. |
| Whisper | Audio → transcripción de texto | Transcribe; por sí solo no razona sobre escenas ni interpreta el video. |
| GPT Image y FLUX | Generación y/o edición de imágenes según versión | No son modelos LLM de visión general; revisar entradas, salidas y límites de cada servicio. |

### Ejemplo: capacitación grabada en video

| Necesidad | Enfoque inicial |
|---|---|
| Resumir lo que explica el instructor | Transcribir el audio y enviar la transcripción a un LLM. |
| Describir diagramas o contenido en pantalla | Extraer frames representativos y analizarlos con un modelo de visión. |
| Encontrar cuándo ocurre un evento o cómo cambia una escena | Usar timestamps y análisis temporal con Content Understanding, Video Indexer o un modelo de video compatible. |

La extracción de algunos frames es una aproximación de muestreo: **no equivale al análisis completo del video** y puede omitir eventos entre frames. Para resultados temporales, validar cobertura, frecuencia de muestreo y sincronización con el audio.

## Modalidad, alojamiento y operación son decisiones distintas

Una necesidad multimodal **no exige por sí sola una VM**. Separar la selección de modalidad de la forma de consumir o alojar el modelo:

| Opción | Cuándo evaluarla | Consideraciones |
|---|---|---|
| API de Azure AI Foundry: serverless/API, Standard o Provisioned según oferta | Tareas estándar, prototipos y servicios/modelos ya disponibles mediante una API administrada | Confirmar modelo, SKU, región, cuotas, límites de tasa, modalidad, unidad de cobro y estado GA/preview. Las opciones no son intercambiables ni están disponibles para todos los modelos. |
| Foundry/Azure Machine Learning Managed Compute | Modelos o personalizaciones compatibles que requieren despliegue administrado en cómputo propio | Verificar soporte del modelo, SKU, runtime y requisitos de despliegue. El fine-tuning soportado no es exclusivo de una VM autogestionada. |
| VM con GPU | Excepción justificada por runtime, kernels, drivers, dependencias, hardware o procesamiento de video específico que no admita una opción administrada | Incluir operación, parches, capacidad, observabilidad, red, almacenamiento, resiliencia y calidad en el TCO; no inferir ventajas económicas sin benchmark representativo. |

Los pesos de GPT y Claude cerrados no se pueden autohospedar. **gpt-oss es una oferta distinta**, no una versión equivalente ni los pesos de los modelos GPT cerrados. Un modelo propio que no tenga una opción serverless puede ser desplegable en Managed Compute si es compatible; una VM debe responder a una necesidad técnica concreta.

Conectividad privada no garantiza soberanía de datos ni operación offline. Una VM en Azure no equivale a infraestructura local desconectada. “Open weight” tampoco significa licencia irrestricta: revisar licencia, versión y permiso de uso comercial. Cuotas, región, ciclo de vida, estado GA/preview y API concreta condicionan el soporte real.

## FinOps para cargas multimodales

El supuesto de **12.000 tokens por prompt** del escenario anterior representa texto: no se debe extrapolar directamente a visión, audio, video ni generación. Medir cada modalidad con tareas y criterios de aceptación representativos.

| Carga | Unidades y componentes a medir | Métrica de decisión |
|---|---|---|
| Texto y RAG | Tokens de entrada, tokens cacheados, salida y razonamiento; embeddings, recuperación, reintentos y herramientas | Costo por tarea correcta, junto con calidad y latencia. |
| Documentos | Páginas, tokens, OCR, extracción, reintentos y revisión humana | Costo por documento que cumple los criterios de extracción. |
| Imágenes de entrada | Tokens visuales o unidad de servicio, resolución, crops y cantidad de imágenes | Costo por imagen/caso aceptado, con calidad a la resolución objetivo. |
| Audio y voz | Minutos/segundos o tokens de audio; ASR, diarización, LLM y TTS | Costo por minuto útil, WER y latencia extremo a extremo. |
| Video | Minutos, frames, audio, decodificación, almacenamiento y transferencia/egress | Costo por hora analizada, junto con cobertura de eventos y precisión temporal. |
| Generación de imagen/video | Unidad facturable del servicio (imagen, clip, segundo o tokens, según corresponda) y regeneraciones | Costo por activo aceptado, no solo por intento generado. |

**Costo por resultado válido = TCO del procesamiento / número de resultados que cumplen los criterios de aceptación.** Un benchmark de GPU para texto no representa el rendimiento ni el costo de visión, audio, video o generación.

## Recomendación para la evaluación

1. Empezar en Foundry con tareas estándar y medir modelos representativos; usar enrutamiento entre modelos pequeños para solicitudes simples y modelos de razonamiento para las complejas cuando las pruebas justifiquen la diferencia.
2. Evaluar Managed Compute para pesos propios o personalización cuando modelo y runtime sean compatibles.
3. Considerar una VM solo ante una necesidad técnica demostrada o una comparación de TCO favorable que incluya operación, capacidad y calidad.
4. Registrar por escenario: modelo y versión, modalidad de entrada/salida, región, API y runtime, estado GA/preview, cuota, unidad de precio y métricas de calidad/aceptación.

La tabla de niveles Basic/Standard/Premium de la captura original **no es una tabla de tarifas de Foundry**: corresponde a otro contexto de producto/API. No usarla para inferir precios de Foundry. Tampoco tratar las ventanas de contexto que aparecen en una captura como límites de Azure; confirmar límites en la ficha y API actuales del modelo desplegado.

Para el análisis económico previo, consulta también [Modelo Financiero: FinOps Foundry vs GPU](Modelo-Financiero-FinOps-Foundry-vs-GPU.md); ese documento se enlaza como referencia y no se modifica aquí.

## Fuentes oficiales

Referencias consultadas el **8 de octubre de 2026**. La disponibilidad, las regiones, los estados GA/preview, las modalidades y los precios pueden cambiar; confirmar la ficha vigente antes de tomar decisiones.

- [Modelos vendidos directamente por Azure (Azure AI Foundry)](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Modelos Claude en Azure AI Foundry](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/claude-models)
- [Azure AI Content Understanding: descripción general](https://learn.microsoft.com/en-us/azure/ai-services/content-understanding/overview)
- [Desplegar modelos de Foundry](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/how-to/deploy-foundry-models)
