# Proyecto Final: Agente Comercial Autónomo con RAG en n8n — Estudio "Shaseki"

**Autor:** Ariel Hernán Contini  
**Arquitectura:** n8n + WhatsApp Business Cloud API + Google Drive + Groq Cloud (LPU) + Cohere Embeddings  

---

## 1. Definición del Caso de Negocio y Justificación de ROI

### Proceso Manual Automatizado
El estudio de diseño y dirección creativa **Shaseki** gestiona consultas diarias de clientes y prospectos divididos en tres segmentos comerciales (**Empresa - Cat. A**, **PyME - Cat. B** y **Particular - Cat. C**) a lo largo de un tarifario con más de 100 servicios distribuidos en 14 rubros (Identidad, Web, UX/UI, Audiovisual, Packaging, Ilustración, entre otros).

Previo a esta automatización, cada consulta entrante por WhatsApp requería:
1. Verificar manualmente el tarifario vigente actualizado por IPC.
2. Filtrar el arancel según la tipología de cliente (A, B o C) y calcular porcentajes de anticipo (30%), adaptaciones (50%) o descuentos por gremio (20%).
3. Redactar respuestas individuales o filtrar solicitudes fuera de alcance (como costos de imprenta física o plazos especiales de financiación).

### Cuantificación de Tiempo, Costo Actual y Ahorro Neto (ROI)
* **Tiempo manual invertido:** ~2,5 horas semanales (10 horas mensuales) buscando aranceles, verificando cláusulas operativas y redactando pre-cotizaciones.
* **Tiempo de mantenimiento del sistema:** ~15 minutos semanales destinados a actualizar el documento maestro en Google Drive y disparar la re-indexación vectorial cuando se ajustan los precios por inflación.
* **Ahorro Neto Efectivo:** **2 horas y 15 minutos semanales (9 horas mensuales)** liberadas íntegramente para horas facturables de diseño y producción. Tomando como referencia el valor de la *Hora de Trabajo* oficial del estudio ($30.500 ARS / ~20 USD), el sistema recupera **$274.500 ARS mensuales (~180 USD/mes)** en capacidad productiva frente a un costo operativo de API cercano a **$0 USD** (dentro de los *Free Tiers* de Groq y Cohere).

### Por qué escala mejor y Límites del ROI
* **Ventaja de escalabilidad:** Ante campañas publicitarias o lanzamientos donde las consultas crecen de 5 a 50 diarias, el proceso manual genera demoras de horas en la primera respuesta comercial (enfriando el *lead*). El agente RAG responde en menos de 2 segundos las 24 horas manteniendo precisión arancelaria.
* **Límites del ROI (¿Cuándo dejaría de tener sentido automatizarlo?):**
  1. **Proyectos 100% ad-hoc sin tarifario estandarizado:** Si el estudio abandonara los servicios tipificados y pasara a cotizar únicamente consultoría a medida donde cada presupuesto requiere reuniones previas de descubrimiento, el RAG perdería utilidad práctica.
  2. **Volumen ínfimo vs. alta volatilidad:** Si el estudio recibiera menos de 2 consultas mensuales pero cambiara sus precios todos los días manualmente, el tiempo de mantenimiento superaría el tiempo ahorrado.

---

## 2. Arquitectura del Sistema y Fuentes de Entrada

El workflow integra **dos fuentes de entrada diferenciadas** en flujos desacoplados que convergen en el almacenamiento vectorial (`vector_store_key`):

1. **Fuente 1 — Google Drive API (`Look into your book of rules`):** Actúa como fuente documental autoritativa descargando el archivo maestro `tarifario_shaseki.md` mediante autenticación OAuth2 para alimentar la base vectorial.
2. **Fuente 2 — WhatsApp Business Cloud Webhook (`Mensaje Entrante`):** Escucha eventos en tiempo real (`messages`) enviados por usuarios finales al número oficial de WhatsApp del estudio.

### ¿Por qué este proceso requiere un AI Agent con RAG y no un nodo de IA simple?
Un nodo de IA simple (LLM estático con el tarifario pegado en el prompt) resultaría inviable e ineficiente por tres razones:
1. **Eficiencia de contexto y costos:** Enviar los más de 100 servicios en cada mensaje consumiría miles de tokens innecesarios por turno. Con **RAG (`retrieve-as-tool`)**, el agente vectoriza la consulta y recupera únicamente los fragmentos (`topK`) pertinentes al rubro consultado.
2. **Razonamiento condicional y uso de herramientas:** El **AI Agent** evalúa primero el mensaje, mantiene memoria de la conversación (`Don't you forget about me`), decide autónomamente cuándo invocar la herramienta `RAG`, discrimina qué tarifa aplicar según si el usuario declaró ser *Empresa*, *PyME* o *Particular* (o repregunta si es ambiguo) y decide cuándo activar el **Protocolo de Fallback** para derivar el caso a la dirección humana del estudio.
3. **Desacoplamiento dato-lógica:** Permite actualizar precios en Google Drive y re-indexar vectores sin tocar una sola línea del prompt ni la lógica del flujo en n8n.

---

## 3. Decisión de Modelo (Local vs. Cloud), Privacidad y Ciclo del Vector Store

### Elección: Arquitectura 100% Cloud (Groq LPU + Cohere Multilingual)
Se seleccionó **`openai/gpt-oss-20b`** corriendo en **Groq Cloud** (`Temperature: 0.2`) para el agente principal y **`embed-multilingual-v3.0`** de **Cohere** (1024 dimensiones) para los embeddings:

* **Costo según volumen e Independencia de Proveedores:** Para el volumen operativo del estudio (15–40 consultas diarias), el costo en la nube es marginal/gratuito. Correr un modelo con capacidad confiable de *Tool Calling* localmente en Ollama consumiría recursos críticos de GPU/RAM en la estación de trabajo de diseño (usada simultáneamente para renderizado, Adobe CC y Figma). Además, al utilizar los nodos modulares de LangChain en n8n, el sistema mantiene **independencia de proveedor**: se puede conmutar el sub-nodo `in the membrane` de Groq a OpenAI, Anthropic u Ollama local en un clic sin reconstruir el agente.
* **Políticas de Privacidad, Retención, Acceso y Tratamiento de Datos:**
  * **Minimización de datos (Tratamiento):** Al modelo LLM solo se le inyecta el cuerpo de texto de la consulta (`$json.messages[0].text.body`), evitando exponer metadatos sensibles en el prompt.
  * **Retención (Data Retention):** Se opera bajo políticas *Zero Data Retention* de API y una memoria volátil acotada a una ventana de 12 interacciones (`contextWindowLength: 12`) identificada por sesión (`sessionKey`), evitando el almacenamiento permanente de conversaciones privadas.
  * **Control de Acceso:** Las credenciales OAuth2 de Google Drive y los tokens de Meta WhatsApp Cloud se almacenan encriptados en el gestor de credenciales de n8n bajo permisos de mínimo privilegio.

### Separación del Flujo de Ingesta y Actualización del Vector Store
El flujo de ingesta (`GO!` -> `Look into your book of rules` -> `Tell me what you see` -> `DATALOADER` + `Cohere it!` -> `Vectores`) se conserva separado del flujo transaccional de WhatsApp. 
* **Mecanismo de actualización:** Cuando se actualizan los aranceles en Google Drive, se ejecuta el disparador `GO!`. El nodo `Vectores` está configurado en modo `Insert` con el parámetro **`clearStore: true`**: esto purga automáticamente los vectores antiguos antes de insertar los nuevos fragmentos, garantizando lecturas limpias sin duplicados ni precios viejos.

---

## 4. Manejo de Errores y Casos Límite (Resiliencia)

El diseño contempla explícitamente **dos puntos críticos de control de fallos**:

1. **Punto de Control #1 — Filtro de Payloads No Textuales (`valido si es texto`):**
   * *Riesgo:* Los usuarios en WhatsApp suelen enviar audios, imágenes, GIFs o stickers, o bien la API de Meta envía webhooks de estado (`delivered`/`read`) sin cuerpo de texto, lo que provocaría un fallo por input nulo en el AI Agent.
   * *Mitigación:* El nodo condicional `If` evalúa estrictamente que `={{ $json.messages[0].text.body }}` exista (`string exists`). Si la condición es **False**, desvía el flujo hacia el nodo `Si nos envían un sticker, GIF o Imagen`, respondiendo cortésmente al usuario que solo procesa texto y protegiendo al LLM.
2. **Punto de Control #2 — Límites de Alucinación y Fallback por Fuera de Alcance (Guardrail en System Prompt):**
   * *Riesgo:* Que el cliente consulte sobre servicios no ofrecidos (ej. costos de impresión física en imprenta) o condiciones financieras especiales (ej. pausas de proyecto por 20 días o pagos en criptomonedas) y el LLM invente una respuesta comprometiendo comercialmente al estudio.
   * *Mitigación:* Se fijó `Temperature: 0.2`, se obligó a invocar la herramienta `RAG` antes de responder y se programó un **Protocolo de Fallback determinista** que deriva automáticamente la consulta a la dirección de Shaseki cuando el dato no existe en los fragmentos recuperados.

---

## 5. Ingeniería de Prompt (System Prompt Completo)

```text
Rol: Eres el Asistente Comercial y Ejecutivo de Cuentas del estudio de diseño "Shaseki". Atiendes a clientes y prospectos a través de WhatsApp Business.

REGLA CRÍTICA DE EJECUCIÓN (OBLIGATORIA):
Antes de redactar cualquier respuesta sobre precios, aranceles, servicios, anticipos o condiciones de trabajo, DEBES INVOCAR SIEMPRE la herramienta "RAG" enviando como query el nombre del servicio consultado por el usuario. NUNCA respondas de memoria ni asumas que no tienes la información sin haber ejecutado primero la herramienta "RAG".

Tono y Estilo:
- Mantén un tono estrictamente profesional, cordial, consultivo y ejecutivo, adaptado para el cliente final de WhatsApp.
- Sé breve, claro y directo.
- Formato nativo de WhatsApp: Para hacer negritas, usa ESTRICTAMENTE un solo asterisco por lado (ejemplo correcto: *Presupuesto* / ejemplo prohibido: **Presupuesto**). Para listas, usa únicamente guiones (-) sin asteriscos y saltos de línea limpios.

Lógica de Cotización según Tipo de Cliente (Empresa, PyME, Particular):
El tarifario de Shaseki estructura los aranceles en tres categorías de cliente: *Empresa* (Categoría A), *PyME* (Categoría B) y *Particular* (Categoría C), además de valores generales como *Hora de Trabajo*, *Adaptaciones* (50%), *Gremio* (20%) y *Anticipo* (30%).
1. Si el usuario especifica su tipo de perfil en la consulta (ej. "Soy una PyME", "Es para mi empresa" o "Soy un cliente particular"), infórmale directamente el precio exacto correspondiente a su categoría.
2. Si el usuario consulta el precio de un servicio pero NO aclara qué tipo de cliente es, preséntale de forma ordenada los 3 valores disponibles (*Empresa*, *PyME* y *Particular*) para ese trabajo y pregúntale cordialmente al final en cuál de las categorías se encuadra su proyecto para afinar el presupuesto.

Restricciones y Reglas de Negocio:
1. Responde basándote ESTRICTAMENTE en los fragmentos que te devuelva la herramienta "RAG".
2. NO inventes ni recalcules tarifas ni condiciones fuera de lo estipulado en el tarifario.
3. Protocolo de Fallback (Fuera de alcance): Si tras ejecutar la herramienta "RAG" el servicio o dato solicitado no figura en los fragmentos devueltos, responde EXACTAMENTE con este mensaje profesional:
"Para brindarte una respuesta precisa sobre ese requerimiento particular, voy a derivar tu consulta directamente con la dirección de Shaseki para que evalúen el caso y te contacten a la brevedad."
