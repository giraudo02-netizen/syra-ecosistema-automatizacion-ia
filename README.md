# Ecosistema de Automatizacion IA - SYRA S.A.

Pipeline de contenido con control de calidad humano (**HITL - Human In The Loop**), construido sobre **Airtable** (base de control) y **Make** (motor de automatizacion), con un nodo de **IA generativa** (OpenAI) para la redaccion de contenido.

Entrega final del curso de Automatizacion con IA - Lisandro Giraudo.

---

## 1. Diagrama de arquitectura

Ver [diagrama_arquitectura.pdf](./diagrama_arquitectura.pdf) para el diagrama completo.

**Resumen del flujo:**

Paso 1, Airtable: fila nueva con Estado=Generando.

Paso 2, Escenario 1 en Make: Airtable Search Records, luego OpenAI redacta el post, luego Airtable Update Record guarda el Borrador y pasa Estado a En revision. Tiene Error Handler (Retry) en el nodo de IA.

Paso 3, Validacion humana (HITL): un humano revisa el borrador en Airtable y tilda Aprobado.

Paso 4, Escenario 2 en Make: Airtable Search Records con filtro Estado=En revision AND Aprobado=true, luego Airtable Update Record pasa Estado a Publicado mas la fecha. Tambien tiene Error Handler (Retry).

**Por que dos escenarios separados en vez de uno solo con una pausa intermedia:** Make no tiene un nodo nativo de "esperar aprobacion humana" (no hay estado persistente entre ejecuciones dentro de un mismo escenario). El patron correcto para plataformas de automatizacion sin estado persistente es dividir el flujo en dos escenarios independientes, donde el segundo usa un filtro de busqueda como compuerta: el registro es literalmente invisible para el Escenario 2 hasta que el campo Aprobado pasa a true. Esto es, en los hechos, el mecanismo de pausa HITL.

---

## 2. Estructuras de datos

### 2.1 Base de control (Airtable)

**Base:** Centro de Comando - Contenido
**Tabla:** Piezas de Contenido

| Campo | Tipo | Descripcion |
|---|---|---|
| Idea Semilla | Texto largo | Input inicial: el tema/idea que dispara la generacion |
| Borrador Generado | Texto largo | Output de la IA, el post redactado |
| Estado | Seleccion unica | Generando / En revision / Aprobado / Rechazado / Publicado |
| Aprobado | Checkbox | Campo de control humano (HITL). Solo un humano lo tilda |
| Fecha Ultima Ejecucion | Fecha y hora | Se actualiza en cada corrida de Make |
| Fecha de Publicacion | Fecha y hora | Se completa cuando Estado pasa a Publicado |
| Plataforma | Seleccion unica | LinkedIn / Instagram / Blog / Email / Facebook |

La maquina de estados de Estado es: Generando pasa a En revision cuando el Escenario 1 termina con exito; el registro queda invisible para el Escenario 2 hasta que un humano tilda Aprobado; recien ahi el Escenario 2 lo detecta y lo pasa a Publicado.

### 2.2 Logica del flujo (Make)

Los blueprints JSON exportados de cada escenario estan en la carpeta blueprints:

- Escenario 1: [escenario_1_generacion.json](./blueprints/escenario_1_generacion.json), con Trigger Airtable Search Records, IA OpenAI Generate Completion, Escritura Airtable Update Record y Error Handler.
- Escenario 2: [escenario_2_publicacion.json](./blueprints/escenario_2_publicacion.json), con Trigger de filtro compuesto Airtable Search Records, Escritura Airtable Update Record y Error Handler.
Cada blueprint es un JSON auto-descriptivo: incluye el modulo, la conexion (sin credenciales, Make las excluye del export), y el mapeo de campos exacto.

---

## 3. Matriz de decision de costos

Comparacion de los modelos de IA evaluados para el nodo de generacion de contenido, dado que la tarea es texto corto (80-130 palabras) en espanol, sin necesidad de razonamiento complejo:

| Modelo | Costo relativo | Cuando usarlo |
|---|---|---|
| GPT-5.6-luna (usado) | Muy bajo | Tarea actual: texto corto y repetitivo, sin razonamiento complejo. El mas rapido y economico de la familia, ideal para alto volumen |
| GPT-5.6-sol | Alto | Reservar para tareas que requieran razonamiento profundo u orquestacion de agentes (no es el caso de este pipeline) |
| GPT-5.6-terra | Medio | Punto medio: si en el futuro el prompt necesita mas contexto de marca o multiples plataformas a la vez |
| Claude Sonnet 5 | Medio-alto | Alternativa si se necesita mayor calidad de redaccion o control de tono mas fino; mas caro que el modelo economico de OpenAI |

Para este volumen (pocas piezas de contenido por semana), la diferencia de costo entre el modelo economico y el modelo flagship es irrelevante en terminos absolutos, pero la eleccion documenta el criterio: no pagar de mas por capacidad que la tarea no necesita.

**Decision tomada:** se uso el modelo mas economico de la familia (gpt-5.6-luna) porque la tarea (redactar un post corto siguiendo una plantilla de marca fija) no se beneficia de mayor capacidad de razonamiento. Si el pipeline creciera para incluir generacion de imagenes, analisis de sentimiento de comentarios, o redaccion de piezas mas largas y estrategicas, se reevaluaria subir a un modelo de gama media.

**Optimizaciones de costo aplicadas:** Max Output Tokens limitado para evitar que la IA divague; el prompt incluye las directrices de marca en el mismo mensaje para evitar una segunda llamada a la API; y el filtro de busqueda en Airtable evita que Make reprocese registros que ya tienen un borrador.

---

## 4. Seguridad y resiliencia

### 4.1 Minimizacion de datos

La conexion de Airtable y OpenAI en Make usa OAuth, sin tokens ni claves expuestas en el blueprint exportado (Make las excluye automaticamente del JSON). El prompt enviado a la IA solo incluye la Idea Semilla mas las directrices de marca ya fijas en el escenario; no se envian datos personales ni informacion sensible de la empresa. La base de Airtable no almacena datos de clientes ni informacion confidencial: solo contenido de marketing en borrador.

### 4.2 Gestion de errores (resiliencia)

Ambos escenarios de Make tienen un Error Handler adjunto al modulo critico (la llamada a la IA en el Escenario 1, la escritura final en el Escenario 2). Si el modulo falla, por ejemplo si la API de OpenAI no responde o Airtable devuelve un error de rate-limit, el error handler captura el fallo y no deja que el escenario se rompa silenciosamente. La ejecucion fallida se guarda como ejecucion incompleta en Make, disponible para revision manual en la pestana Ejecuciones incompletas de cada escenario. Esto evita el peor escenario posible: que un registro quede colgado en un estado intermedio sin que nadie se entere.

### 4.3 El filtro final como control de seguridad

El filtro de busqueda del Escenario 2 (Estado igual a En revision y Aprobado igual a true) es, ademas de logica de negocio, un control de seguridad: ningun contenido llega a la plataforma externa ni al estado Publicado sin que un humano lo haya validado explicitamente. No existe ningun camino en el flujo que evite este control.

---

## 5. Dashboard y evidencia

- Dashboard, enlace publico de solo lectura: vista de Airtable agrupada por Estado, que funciona como cuadro de mando en tiempo real. Enlace: https://airtable.com/appSN0Dz6g3fzaUDE/shrJAMXb9kCQ8t7hu
- Capturas de ejecuciones reales: ver la carpeta screenshots, con el diagrama de cada escenario mostrando historial de ejecuciones exitosas, y capturas de la base de Airtable mostrando el ciclo completo: un registro frenado en En revision (sin aprobar) junto a otro ya Publicado (aprobado).



## Stack utilizado

- Airtable: base de datos y centro de comando, ademas del dashboard
- Make: motor de automatizacion, con dos escenarios
- OpenAI (GPT-5.6-luna): generacion de contenido
- GitHub: documentacion y evidencia de la entrega
