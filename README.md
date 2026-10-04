# Ecosistema de Propuestas Comerciales con IA — Venta de Hierro para Construcción

**Trabajo Final Integrador — Automatización con IA**

Sistema en **n8n** que genera propuestas comerciales personalizadas para una distribuidora de hierro para construcción. Calcula los precios con la lista real, redacta la propuesta con IA y la envía por mail con una **aprobación humana (Approve / Decline)** antes de que salga al cliente.

> Los clientes, emails, teléfonos y precios que aparecen en el repositorio son datos de prueba.

---

## 1. Caso de uso

En el rubro, los clientes piden cantidades específicas de cada medida de hierro (por ejemplo, 2 TN de ADN 8 mm y 1 TN de ADN 10 mm). Armar la cotización a mano implica buscar precios, aplicar IVA según el tipo de facturación y redactar el mail. Este flujo automatiza todo eso y deja una persona como control final.

El pedido puede entrar por **dos caminos**:

| Rama | Disparo | Quién carga el pedido |
|------|---------|-----------------------|
| **A — Carga interna** | Notion Trigger (página nueva en la base *Pedidos*) | El equipo, directo en Notion |
| **B — Formulario público** | n8n Form Trigger ("Pedido de Materiales") | El cliente, por su cuenta |

La rama B termina creando el pedido en Notion, así que ambas ramas confluyen en el mismo pipeline de IA.

---

## 2. Stack tecnológico (4 pilares)

| Pilar | Tecnología | Rol |
|-------|-----------|-----|
| Orquestador | **n8n** | Coordina el flujo, validaciones y rutas de error |
| Base de datos | **Notion** | Bases *Clientes*, *Pedidos* y *Lista de Precios* |
| Inteligencia artificial | **Google Gemini** | Calcula precios con la lista real y redacta la propuesta |
| Canal de salida + HITL | **Gmail** (*Send and Wait for Response*) | Envía la propuesta y espera Approve / Decline |

Además se usa **Google Sheets** como registro de errores.

**Por qué Gemini:** el tier gratuito permite usar el flujo sin tarjeta de crédito, y la consigna habilita cualquier LLM siempre que se justifique la elección.

**Por qué solo Gmail como canal de salida:** se priorizó dejar un flujo completo y probado de punta a punta con un único canal, en vez de sumar canales sin validarlos.

---

## 3. Arquitectura

El diagrama completo está en [`docs/arquitectura.pdf`](docs/arquitectura.pdf).

### Rama B — Formulario público

```
On form submission
  → HTTP Request1 (busca el Cliente en Notion por Nombre, "Contains")
  → If2 (¿el cliente existe?)
       ├─ sí → Edit Fields1 (toma el cliente_id existente)
       └─ no → Create a database page (crea el Cliente) → Edit Fields (cliente_id nuevo)
  → Code in JavaScript1 (arma "detalle_pedido" desde los campos de cantidad)
  → Create a database page1 (crea el Pedido en Notion con Estado = Pendiente)
```

El formulario usa **un campo numérico por cada medida de hierro** (en vez de texto libre). El nodo Code convierte esas cantidades en un detalle con formato fijo (por ejemplo, `2TN HIERRO ADN X 08mm`), así Gemini no tiene que interpretar texto escrito libremente por el cliente. El total lo sigue calculando Gemini con la lista de precios.

### Rama A — Pipeline de procesamiento (Notion Trigger)

```
Notion Trigger (pedido nuevo en Pedidos)
  → HTTP Request (consulta la Lista de Precios)
  → Code in JavaScript (convierte el JSON a texto plano "Producto — $Precio / Unidad")
  → Get many database pages (Pedidos, filtro Estado = Pendiente)
  → Get a database page (datos del Cliente)
  → If (¿el detalle no está vacío y el Estado no es "Procesado por IA"?)
       ├─ true  → Message a model (Gemini)
       │            ├─ si Gemini falla → Google Sheets: Append row (registro)
       │            → Update a database page (guarda la propuesta, Estado = "Procesado por IA")
       │            → Gmail Send and Wait for Response (HITL: Approve / Decline al responsable)
       │            → If1 (¿el responsable aprobó?)
       │                 ├─ Approve → Gmail Send a message (envía la propuesta al email del cliente)
       │                 │              └─ si el envío falla → Google Sheets: Append row (registro)
       │                 └─ Decline → Google Sheets: Append row (registro)
       └─ false → (en paralelo)
                    ├─ Update a database page (Estado = CON ERROR, para no volver a procesarlo)
                    └─ Google Sheets: Append row (registro de errores)
```

---

## 4. Lógica de negocio

**Cálculo de precios:**

```
precio_final = precio_unitario × cantidad × (1 + IVA_aplicable)
```

| Tipo de pedido | IVA aplicado |
|----------------|--------------|
| Facturado | 21 % |
| Sin Factura | 10,5 % (el IVA se reparte a la mitad) |

**Ejemplos verificados a mano** (la lista de precios está expresada en TN, igual que los pedidos):

| Pedido | Cálculo | Total |
|--------|---------|-------|
| MINIG SA (Sin Factura): 1 TN de 8 mm, 2 TN de 10 mm, 3 TN de 16 mm y 2 TN de 25 mm | 254.207,39 × 1,105 | **$280.899,17** |
| WRASKO SA (Facturado): 2 TN de 6 mm, 1 TN de 8 mm y 2 TN de 16 mm | 72.665,51 × 1,21 | **$87.925,27** |

En los dos casos el total que generó Gemini coincide con precio de lista × cantidad × IVA.

**RAG básico de precios:** en lugar de que la IA invente valores, el flujo consulta la base *Lista de Precios* (7 productos) y se la pasa a Gemini dentro del prompt. El prompt referencia cada dato con `$('Nombre del nodo')` de forma explícita, porque `$json` sin nombre toma el nodo inmediato anterior de la cadena.

---

## 5. Manejo de errores

El flujo no se rompe ante datos incompletos o fallas: los desvía a una hoja de registro.

| Situación | Cómo se detecta | Qué pasa |
|-----------|-----------------|----------|
| Pedido sin detalle (todas las cantidades en 0) | `If` valida que el detalle no esté vacío | Se desvía a la hoja, **no llega a Gemini**, y el pedido pasa a Estado **CON ERROR** |
| Pedido ya procesado | `If` compara Estado con "Procesado por IA" | Se desvía a la hoja y no se reprocesa ni se reenvía el mail |
| Pedido sin Cliente asignado | *Continue Using Error Output* en `Get a database page` | Se desvía a la hoja |
| Falla de la API de Gemini | *Continue Using Error Output* en `Message a model` | Se desvía a la hoja |
| Propuesta rechazada (Decline) | `If1` evalúa la respuesta del responsable | Se registra en la hoja y **no se envía nada al cliente** |
| Falla al enviar el mail al cliente (por ejemplo, cliente sin email) | *Continue Using Error Output* en `Send a message` | Se registra en la hoja con el mensaje de error de Gmail |

**Registro de errores** (Google Sheets, hoja *TFI - Registro de Errores*): [ver en modo lectura](https://docs.google.com/spreadsheets/d/1juGh5yOzH_nWdYWIn6kJjAxvmzYNs_uwDGPT99UCk0s/edit?usp=sharing)

| Columna | Contenido |
|---------|-----------|
| Fecha | Momento del error |
| Pedido_ID | Nombre del pedido en texto legible |
| Nodo con error | Nodo por el que entró el error, con nombre legible (ver abajo) |
| Mensaje error | Descripción del problema (distingue "sin detalle" de "ya procesado") |

**Cómo se completa "Nodo con error":** el nodo Append row usa `$prevNode.name`, que devuelve el nombre del nodo que le mandó el dato, y lo traduce a un nombre legible:

| Ruta | Valor en la hoja |
|------|------------------|
| Pedido sin detalle o ya procesado | Validación de datos (If) |
| Falla de la API de Gemini | Generador de Propuesta (Gemini) |
| Pedido sin Cliente | Búsqueda de Cliente (Notion) |
| Propuesta rechazada (Decline) | Aprobación humana (HITL) |
| Falla al enviar el mail al cliente | Send a message |

**Ejemplo de registros reales** (de las pruebas 4, 5 y 6):

| Pedido_ID | Nodo con error | Mensaje error |
|-----------|----------------|---------------|
| PEPO SA - Nuevo pedido | Validación de datos (If) | Pedido sin detalle - no se envió a la IA (validación de datos) |
| VERTICAL - Nuevo pedido | Send a message | Invalid email address (item 0) |
| PEPO SA - Nuevo pedido | Aprobación humana (HITL) | Propuesta rechazada por el responsable (Decline) - no se envió al cliente |

**Por qué el estado CON ERROR:** el flujo procesa solo pedidos en Estado = Pendiente. Un pedido sin detalle que se quedara en Pendiente volvería a entrar y a registrarse en cada ejecución. Al marcarlo como CON ERROR se registra **una sola vez**, aunque se envíen varios pedidos vacíos seguidos.

**Caso real documentado — límite de requests:** durante las pruebas apareció el error *"The service is receiving too many requests from you"*, por el límite del tier gratuito de Gemini. Se resolvió generando una API Key nueva desde otro proyecto en Google AI Studio.

---

## 6. Punto de validación humana (HITL)

El nodo **Gmail — Send and Wait for Response** envía la propuesta al responsable interno con los botones **Approve / Decline**. El flujo queda en pausa hasta que alguien decide:

- **Approve:** el nodo `Send a message` envía la propuesta al email del cliente.
- **Decline:** no se envía nada al cliente y el rechazo queda registrado en la hoja de errores.

Nada sale al cliente sin esa aprobación.

---

## 7. Bases de datos (modo lectura)

- **Pedidos:** https://harmonious-tricorne-e27.notion.site/44f4be0598d44e8dbd6aa1a068b5ff07
- **Clientes:** https://harmonious-tricorne-e27.notion.site/8f4627917d554fbab009e2652849118a

| Base | Campos principales |
|------|--------------------|
| **Clientes** | Nombre, Email, Teléfono |
| **Pedidos** | Nombre, Detalle del pedido, Estado, Facturación, Fecha de creación, Propuesta generada, relación con Cliente |
| **Lista de Precios** | Producto, Precio por tonelada, Unidad = TN (7 productos) |

**Estados que asigna el flujo:** Pendiente (al crearse) → Procesado por IA (cuando Gemini genera la propuesta, **antes** de la aprobación humana). Los pedidos sin detalle pasan a **CON ERROR**. La propiedad Estado también tiene definidas las opciones Esperando aprobación, Aprobado y Enviado, que el flujo actual no asigna.

> Los pedidos que llegan por el formulario nacen con **Estado = Pendiente**. Los que se cargan a mano en Notion también deben llevar ese estado, porque el flujo filtra por él y ignora el resto.

---

## 8. Pruebas de estrés (6 corridas)

| # | Escenario | Resultado |
|---|-----------|-----------|
| 1 | Pedido completo cargado en Notion (MINIG SA, Sin Factura, 4 ítems) | Propuesta generada con la lista de precios, mail de aprobación recibido y, tras Approve, la ejecución llega hasta el envío al cliente. Total $280.899,17 |
| 2 | Formulario con **cliente existente** (WRASKO SA, Facturado) | Pedido vinculado al cliente existente (If2 por *true*), propuesta de $87.925,27 |
| 3 | Formulario con **cliente nuevo** (PEPO SA, Facturado) | Cliente creado en Notion con email y teléfono (If2 por *false*), pedido vinculado, propuesta de $179.103,78 |
| 4 | **Camino infeliz:** todas las cantidades en 0 | Se crea el pedido, el `If` lo desvía a la hoja sin llegar a Gemini y pasa a CON ERROR |
| 5 | **Camino infeliz:** formulario sin email (VERTICAL, Sin Factura) | La propuesta de $209.734,51 se genera y se aprueba; el envío al cliente falla (*Invalid email address*) y el error queda registrado en la hoja. Ver limitaciones |
| 6 | **Decline** de la propuesta (PEPO SA, Sin Factura) | `If1` sale por *false*, el rechazo se registra en la hoja y no se envía nada al cliente. Total de la propuesta: $51.119,09 |

Los totales de las cinco propuestas generadas coinciden con precio de lista × cantidad × IVA (21 % si es Facturado, 10,5 % si es Sin Factura).

Las capturas de cada corrida están en [`docs/screenshots/`](docs/screenshots/) y explicadas una por una en [`docs/evidencia-pruebas.pdf`](docs/evidencia-pruebas.pdf).

### Hallazgos durante las pruebas

Las pruebas encontraron dos problemas que se corrigieron antes de cerrar:

1. **El teléfono no se guardaba al crear un cliente desde el formulario.** La expresión del campo Teléfono en el nodo que crea el cliente buscaba `'Teléfono'` (con tilde), pero el campo del formulario se llama `Telefono`. Se corrigió la referencia y se volvió a publicar el workflow.
2. **El error de envío no se registraba.** El nodo `Send a message` no tenía salida de error conectada, así que un fallo al enviar el mail no llegaba a la hoja. Se activó *Continue Using Error Output* en ese nodo y se conectó la salida de error a `Append row in sheet`. La Prueba 5 ya muestra el registro en la hoja.

---

## 9. Limitaciones conocidas

- **Cliente sin email:** el formulario no exige el email y el flujo no lo valida al ingresar el pedido. La propuesta se genera y se manda a aprobación, y el error aparece recién al enviar el mail al cliente (*Invalid email address*). Queda registrado en la hoja de errores, pero la ejecución termina en *Succeeded* porque el error está manejado. Una mejora sería validar el email en el `If` inicial o hacerlo obligatorio en el formulario.
- **Estado "Procesado por IA" antes de la aprobación:** el pedido pasa a ese estado al guardarse la propuesta, antes del Approve / Decline. Si el responsable rechaza o el envío falla, el hecho queda registrado en la hoja de errores pero el pedido sigue en "Procesado por IA". Una mejora sería conectar esas dos salidas al nodo que asigna CON ERROR.
- **Búsqueda de cliente por nombre:** el formulario busca con *Contains* y toma el primer resultado. Un nombre contenido en el de otro cliente (por ejemplo "VERTICAL" y "VERTICAL SA") se vincula a ese cliente existente en lugar de crear uno nuevo. Una mejora sería normalizar el nombre o buscar por email.
- **Gmail *Send and Wait*:** procesa solo el primer ítem de un lote, por lo que un lote de varios pedidos genera un único mail de aprobación.
- **Emparejamiento por posición:** `Get a database page` (Clientes) y `Get many database pages` (Pedidos) se emparejan por orden. Un pedido sin Cliente vinculado puede perderse del emparejamiento. Los pedidos que entran por el formulario ya llevan Cliente vinculado.
- **Búsqueda de cliente por nombre:** usa *Contains* y toma el primer resultado (`results[0]`), por lo que un nombre parcial podría vincular el pedido con otra empresa cuyo nombre lo contenga.
- **Fallas temporales de Gemini:** el pedido queda en Pendiente y se reintenta en la próxima ejecución (por ejemplo, ante el límite de requests del tier gratuito). Solo el pedido sin detalle se marca como CON ERROR.
- **Tier gratuito de Gemini:** tiene límite de requests por ventana de tiempo.

---

## 10. Decisiones técnicas

**HTTP Request en lugar del nodo nativo de Notion (Lista de Precios y Clientes).** El nodo nativo autocompletaba en modo *By ID* un Data Source ID que no correspondía a la base real, y fallaba con `Could not extract page id from url: undefined`. Se reemplazó por un nodo HTTP Request contra la API de Notion (`POST /v1/databases/{id}/query`) con Header Auth y el header `Notion-Version: 2022-06-28`.

**Búsqueda de cliente por nombre de empresa.** Se filtra por Nombre con *Contains* y no por Email exacto, porque distintas personas de una misma empresa pueden hacer pedidos con emails distintos.

**Filtro por Estado = Pendiente.** Sin ese filtro, cada ejecución revisaba todos los pedidos de la base y registraba errores de pedidos viejos.

**Estado CON ERROR.** Un pedido sin detalle se marca como CON ERROR apenas se registra, para que el filtro lo ignore en las corridas siguientes y la hoja no se llene de filas repetidas.

---

## 11. Cómo usarlo

**Requisitos:** una instancia de n8n (cloud o self-hosted) y credenciales para Notion, Google Gemini, Gmail y Google Sheets.

1. Importar `workflow.json` en n8n (*Workflows → Import from file*).
2. Crear las credenciales y asignarlas a cada nodo:
   - **Notion:** integración interna, compartida con las bases *Clientes*, *Pedidos* y *Lista de Precios*.
   - **Header Auth** (nodos HTTP Request): header `Authorization: Bearer <token de la integración>`.
   - **Google Gemini:** API Key de Google AI Studio.
   - **Gmail** y **Google Sheets:** OAuth2.
3. Reemplazar en los nodos HTTP Request los IDs de las bases por los de tu propio Notion, cambiar el mail del responsable en el nodo *Send message and wait for response*, elegir tu propia hoja en *Append row in sheet*, y verificar que la propiedad **Estado** de *Pedidos* tenga las opciones Pendiente, Procesado por IA, Esperando aprobación, Aprobado, Enviado y **CON ERROR**.
4. Publicar el workflow (*Publish*). El formulario y el Notion Trigger corren con la versión **publicada**, así que cualquier cambio posterior en el editor requiere volver a publicar para que se aplique.
5. Probar cargando un pedido desde el formulario público o creando una página en la base *Pedidos* con Estado = Pendiente.

> Las credenciales no se incluyen en el JSON exportado: hay que configurarlas en cada instancia.

---

## 12. Estructura del repositorio

```
├── README.md
├── workflow.json              # Flujo de n8n exportado
└── docs/
    ├── arquitectura.pdf       # Diagrama de arquitectura (2 páginas)
    ├── evidencia-pruebas.pdf  # Las 6 pruebas con cada captura explicada
    └── screenshots/           # Capturas originales de las pruebas y del canvas
```
