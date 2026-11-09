# Cuadro de mando de medicamentos refrigerados de alto valor en Power BI

**Español** · [English](#english-summary)

**Power BI + SharePoint + Power Automate**

Cuadro de mando para controlar los medicamentos refrigerados de alto valor de una farmacia: agonistas GLP-1, insulinas y otros tratamientos en cadena de frío. Muestra cuánto dinero hay inmovilizado en la nevera, qué tratamientos se van a dispensar y a qué pacientes hay que avisar. Dos flujos de Power Automate preguntan al paciente si quiere su siguiente dispensación y le avisan cuando llega el pedido. Las capturas de este repositorio usan datos sintéticos.

![Cuadro de mando](docs/dashboard.png)

## El problema

Es un proyecto demo que parte de lo que viví en cinco años trabajando en farmacia. Los agonistas GLP-1 (Ozempic, Mounjaro, Wegovy…) reúnen cuatro problemas a la vez:

- **Precio alto por unidad**: cada pluma parada en la nevera es mucho dinero inmovilizado.
- **Desabastecimiento**: hay que pedir con antelación y reservar la caja al paciente.
- **Demanda creciente**: cada mes hay más pacientes en tratamiento.
- **Pacientes que no recogen**: se pide para un paciente que luego no viene.

El último es el más caro, porque **un refrigerado solo se puede devolver al mayorista en las 48 horas siguientes a recibirlo** (10 días en el resto de productos). Pasado ese plazo, la caja se queda en la nevera hasta que otro paciente la necesite, o hasta que caduca.

El programa de gestión de la farmacia no da una vista del stock refrigerado ni avisa al paciente. La idea del proyecto es **pedir solo lo que el paciente ha confirmado que va a recoger, y en el momento adecuado**.

## La solución

1. Cada pedido de un paciente se registra en una lista de **SharePoint**, con la fecha de la próxima dispensación.
2. Cuando llega esa fecha, **Power Automate** pregunta al paciente por correo si quiere el tratamiento. Si acepta, avisa a la farmacia y marca el pedido como `LISTO PARA PEDIR`. Si no acepta, lo marca como `PEDIDO DENEGADO` y no se inmoviliza dinero en la nevera.
3. Cuando cambia el estado del pedido (por ejemplo, porque ha llegado), un segundo flujo envía al paciente un correo con la fecha y hora de llegada.
4. **Power BI** lee las listas y calcula con **DAX** el stock en nevera, el dinero inmovilizado, el avance de cada tratamiento y la prioridad de cada pedido.

```mermaid
flowchart LR
    A["Nuevo pedido en SharePoint"] --> B["Espera hasta la fecha de próxima dispensación"]
    B --> C{"¿El paciente acepta?"}
    C -->|Sí| D["Aviso a la farmacia<br/>Estado = LISTO PARA PEDIR"]
    C -->|No| E["Estado = PEDIDO DENEGADO"]
    D --> F["Llega el pedido: cambia el ESTADO"]
    F --> G["Correo al paciente con fecha y hora de llegada"]
    A -.-> H["Power BI: modelo y medidas DAX"]
    D -.-> H
    E -.-> H
    H --> I["Cuadro de mando"]
```

### Lista de SharePoint

![Lista STOCK_CONTROL en SharePoint](docs/sharepoint_lista.png)

La lista `STOCK_CONTROL` es la única entrada de datos: la farmacia registra cada pedido y los flujos actualizan su estado.

| Columna | Uso |
| --- | --- |
| `Título` | Producto y dosis (p. ej. *SAXENDA 6 MG*). |
| `CODIGO NACIONAL` | Clave para relacionar con el catálogo de productos. |
| `TIPO DE PRODUCTO` | Agonistas GLP-1, insulinas, antidiabéticos orales u otros. |
| `ESTADO DEL PEDIDO` | `LISTO PARA PEDIR`, `PEDIDO DENEGADO`… Lo actualizan los flujos. |
| `FECHA PROXIMA DISPENSACION` | Cuándo se pregunta al paciente. El flujo 1 espera hasta esta fecha. |
| `FECHA REGISTRO` | Inicio del tratamiento. Base de `%progreso`. |
| `EMAIL` / `TELEFONO` | Contacto del paciente. En el informe solo se ve el correo enmascarado. |
| `FECHA Y HORA` | Fecha y hora de llegada que se comunican al paciente. |
| `IMPORTE` | Valor del pedido. Base del dinero en nevera e inmovilizado. |
| `FARMACIA` | Farmacia que registra el pedido. Permite que varias farmacias de un grupo compartan la solución y filtren por farmacia. |
| `ORIGEN` | `Stock` (servido en el momento) o `Pedido` (hubo que pedirlo). Base de la tasa de servicio inmediato. |
| `COMENTARIO` | Notas libres. |

En la captura, la columna de correo está pixelada.

### Flujo 1: confirmación del pedido

```mermaid
flowchart TD
    T(["1 · Cuando se crea un elemento<br/>SharePoint"]) --> R["2 · Redactar<br/>datos del pedido"]
    R --> W["3 · Retraso hasta<br/>fecha de próxima dispensación"]
    W --> M["4 · Enviar correo con opciones<br/>Acepto / No acepto"]
    M --> C{"5 · Condición<br/>¿Acepta?"}
    C -->|Sí| N["Correo de aviso a la farmacia"]
    N --> U1["Actualizar elemento<br/>Estado = LISTO PARA PEDIR"]
    C -->|No| U2["Actualizar elemento<br/>Estado = PEDIDO DENEGADO"]
```

<details>
<summary>Captura del flujo en Power Automate</summary>

![Flujo de confirmación del pedido](docs/powerautomate_confirmacion_pedido.png)

</details>

| Paso | Acción | Qué hace |
| --- | --- | --- |
| 1 | **Cuando se crea un elemento** (SharePoint) | Se dispara con cada pedido nuevo en la lista de pedidos. |
| 2 | **Redactar** | Prepara los datos del pedido para el correo. |
| 3 | **Retraso hasta** | Espera hasta la fecha de próxima dispensación. |
| 4 | **Enviar correo electrónico con opciones** (Outlook) | Pregunta al paciente: *Acepto / No acepto*. |
| 5 | **Condición** | Comprueba la respuesta del paciente. Si **acepta**, envía un aviso a la farmacia y actualiza el elemento con `Estado = LISTO PARA PEDIR`. Si **no acepta**, actualiza el elemento con `Estado = PEDIDO DENEGADO`. |

### Flujo 2: aviso de llegada

```mermaid
flowchart TD
    T(["1 · Cuando se crea o modifica un elemento<br/>SharePoint"]) --> G["2 · Obtener cambios del elemento<br/>solo propiedades"]
    G --> R["3 · Redactar<br/>¿ha cambiado ESTADO?"]
    R --> C{"4 · Condición"}
    C -->|Sí| M["Correo al paciente<br/>fecha y hora de llegada"]
    C -->|No| X["No hace nada"]
```

<details>
<summary>Captura del flujo en Power Automate</summary>

![Flujo de aviso de llegada](docs/powerautomate_aviso_llegada.png)

</details>

| Paso | Acción | Qué hace |
| --- | --- | --- |
| 1 | **Cuando se crea o modifica un elemento o un archivo** (SharePoint) | Se dispara con cualquier cambio en la lista de pedidos. |
| 2 | **Obtener cambios para un elemento o archivo (solo propiedades)** | Compara el elemento con su versión anterior. |
| 3 | **Redactar** | Comprueba si ha cambiado la columna `ESTADO`. |
| 4 | **Condición** | Si ha cambiado `ESTADO`, envía al paciente un correo con la fecha y hora de llegada. Si no, no hace nada. |

Con **Obtener cambios**, el correo solo sale cuando cambia el estado y no con cualquier otra edición del pedido.

## Qué muestra el cuadro de mando

| Visual | Qué responde |
| --- | --- |
| Inmediatez en insulinas | ¿Qué parte de los pedidos de insulina se sirve desde stock, sin tener que pedirlos? |
| Servicio inmediato GLP-1 frente al objetivo mensual | ¿Cumplo el objetivo que me he puesto? (El 20 % es un valor de ejemplo, configurable por farmacia.) |
| Total de dinero en nevera | ¿Cuánto stock refrigerado tengo hoy? |
| Pedidos por grupo y por producto | ¿Qué pesa más: GLP-1, insulinas, antidiabéticos orales u otros? |
| Dinero inmovilizado (importe frente a días de stock) | ¿Qué producto lleva más tiempo parado y cuánto vale? Las líneas verticales marcan los plazos de devolución: **48 h** para refrigerados y **10 días** para el resto. Todo lo que queda a la derecha ya no se puede devolver. |
| Dispensación futura con semáforo | ¿Qué pedidos hay que hacer o confirmar esta semana, y cuáles son prioritarios? |
| Top 10 de tratamiento consumido, con demora | ¿Qué pacientes están a punto de terminar su tratamiento y suelen tardar en recoger? |
| Pedidos mensuales | ¿Cómo evoluciona la demanda? |

## Stack

- **SharePoint**: listas como origen de datos.
- **Power BI**: Power Query, modelo en estrella y DAX. Publicado en el Power BI Service con actualización programada.
- **Power Automate**: correos con opciones, esperas programadas y actualización de estados.
- **Power Apps**: primer prototipo del formulario de registro de pedidos (ver [Qué haría distinto](#qué-haría-distinto)).
- **Outlook**: envío de los correos a pacientes y a la farmacia.

## Requisitos

### Licencias y servicios

| Requisito | Para qué | Notas |
| --- | --- | --- |
| **Microsoft 365** (plan empresarial con SharePoint Online y Exchange Online) | Lista `STOCK_CONTROL`, correos de Outlook y Power Automate. | Los conectores que usan los flujos (SharePoint, Outlook, Retraso, Condición) son estándar: no hace falta licencia *premium* de Power Automate. |
| **Power BI Desktop** | Construir el modelo, las medidas DAX y los visuales. | Gratuito. Solo para Windows. |
| **Power BI Pro** (o Premium por usuario / capacidad Fabric) | Publicar en el Power BI Service, compartir el informe y programar la actualización. | Con Pro, hasta 8 actualizaciones al día. SharePoint Online es un origen en la nube: no necesita *gateway*. |
| **Power Apps** (opcional) | Formulario de registro de pedidos. | Solo para el prototipo descrito en [Qué haría distinto](#qué-haría-distinto). |

### Permisos

- Crear y editar listas en un sitio de SharePoint.
- Crear flujos de Power Automate con conexiones a SharePoint y Outlook. La cuenta del flujo es la que envía los correos.
- Un área de trabajo en el Power BI Service donde publicar el informe.

### Configuración

- **Zona horaria común** en el sitio de SharePoint, en los flujos y en Power BI (ver [Problemas encontrados](#problemas-encontrados)).
- **Fechas de próxima dispensación a menos de 30 días** del registro: es el máximo que una ejecución de Power Automate puede esperar con «Retraso hasta».
- Correo electrónico del paciente en cada pedido, y su consentimiento para recibir los avisos (RGPD).

### Conocimientos

- Power Query y modelado en estrella en Power BI.
- DAX intermedio: `CALCULATE`, `USERELATIONSHIP`, variables y `SWITCH ( TRUE () )`.
- Power Automate: desencadenadores de SharePoint, correos con opciones y condiciones.

## Modelo de datos y medidas clave

```mermaid
erDiagram
    PRODUCTS ||--o{ STOCK_CONTROL : "CODIGO NACIONAL"
    TIME_TABLE ||--o{ STOCK_CONTROL : "FECHA REGISTRO (activa)"
    TIME_TABLE ||..o{ STOCK_CONTROL : "FECHA DISPENSACION FUTURA (inactiva)"
    TIME_TABLE ||..o{ STOCK_CONTROL : "FECHA Y HORA de llegada (inactiva)"
```

Línea continua: relación activa. Línea discontinua: relación inactiva, que se activa con `USERELATIONSHIP`.

<details>
<summary>Captura de la vista de modelo en Power BI</summary>

![Modelo de datos](docs/modelo_datos.png)

</details>

Esquema en estrella con una tabla de hechos, dos dimensiones y una tabla aparte para las medidas:

| Tabla | Tipo | Contenido |
| --- | --- | --- |
| **`STOCK_CONTROL`** | Hechos | Un registro por pedido de paciente: `CODIGO NACIONAL`, `EMAIL`, `FARMACIA`, `ESTADO DEL PEDIDO`, `ORIGEN`, `FECHA REGISTRO`, `FECHA DISPENSACION FUTURA` y `COMENTARIO`. Tiene columnas calculadas como `emailmasked` (correo enmascarado para mostrarlo en el informe), `EnlaNevera` y `Chivatorecogida`. |
| **`PRODUCTS`** | Dimensión | Catálogo por `CODIGO NACIONAL`, con `TIPO DE PRODUCTO` (Agonistas GLP-1, insulinas, antidiabéticos orales y otros) y la columna calculada `Criterio_Stock`. |
| **`TIME_TABLE`** | Dimensión de fechas | Calendario (`Date`). |
| **`Medidas`** | Tabla de medidas | Agrupa las medidas DAX: `%progreso`, `Dias en STOCK (Real)`, `Etiqueta%ypereza`, `Filtroproximos7dias`, etc. |

Relaciones:

- `PRODUCTS` 1 → * `STOCK_CONTROL` por `CODIGO NACIONAL`.
- `TIME_TABLE` 1 → * `STOCK_CONTROL` con tres relaciones de fecha: la activa es `FECHA REGISTRO`, que se usa en los pedidos mensuales. Las inactivas son la fecha de dispensación futura y la de llegada, y se activan con `USERELATIONSHIP` en las medidas que las necesitan. Así hay una sola tabla calendario para analizar las tres fechas.

Transformaciones en Power Query:

- Conversión de tipos: fechas, importes y códigos nacionales que SharePoint devuelve como texto.
- Eliminación de las columnas internas de SharePoint (Id, GUID, autor, versiones…) para aligerar el modelo.
- Construcción de `PRODUCTS` a partir de la lista: códigos nacionales únicos con su tipo de producto.

### `Tasa Servicio Inmediato`: qué parte de los pedidos se sirve en el momento

Un pedido es inmediato cuando se sirve desde el stock de la farmacia, sin tener que pedirlo. El `+ 0` hace que la tarjeta muestre 0 % en vez de quedarse en blanco cuando no hay pedidos.

```dax
Tasa Servicio Inmediato =
VAR Total = COUNTROWS ( STOCK_CONTROL )
VAR Inmediatos =
    CALCULATE ( COUNTROWS ( STOCK_CONTROL ), STOCK_CONTROL[ORIGEN] = "Stock" )
RETURN
    DIVIDE ( Inmediatos, Total ) + 0
```

### `%progreso`: cuánto tratamiento ha consumido el paciente

Calcula qué parte del tiempo entre el registro del pedido y la próxima dispensación ya ha pasado. Alimenta el visual *Top 10 TTO consumido*: un paciente al 97 % está a punto de necesitar su siguiente caja.

Se da un **margen de tolerancia del 20 %**: un paciente puede ir algo tarde con su tratamiento. Por encima del 120 % se considera que ya no lo sigue y sale del ranking.

```dax
%progreso =
VAR fechainicio   = SELECTEDVALUE ( STOCK_CONTROL[FECHA REGISTRO] )
VAR FECHAFINAL    = SELECTEDVALUE ( STOCK_CONTROL[FECHA DISPENSACION FUTURA] )
VAR FECHAHOY      = TODAY ()

VAR DURACIONTOTAL = DATEDIFF ( fechainicio, FECHAFINAL, DAY )
VAR DIASPASADOS   = DATEDIFF ( fechainicio, FECHAHOY, DAY )
VAR RESULTADO     = DIVIDE ( DIASPASADOS, DURACIONTOTAL )

RETURN
    SWITCH (
        TRUE (),
        FECHAHOY > FECHAFINAL, BLANK (),
        ISBLANK ( fechainicio ) || ISBLANK ( FECHAFINAL ), BLANK (),
        fechainicio = FECHAFINAL, 1,
        RESULTADO <= 0, BLANK (),
        RESULTADO > 1.2, BLANK (),
        RESULTADO
    )
```

### `Etiqueta%ypereza`: etiqueta del Top 10

Junta el avance del tratamiento y la demora habitual del paciente en una sola etiqueta de datos, por ejemplo `97% (+1d demora)`.

```dax
Etiqueta%ypereza =
VAR Porcentaje = [%progreso]
VAR Pereza     = [TASAPEREZA]
RETURN
    FORMAT ( Porcentaje, "0%" ) & " (+"
        & IF ( ISBLANK ( Pereza ), "0", FORMAT ( Pereza, "0" ) )
        & "d demora)"
```

### `Semáforo Acción Pedido`: prioridad de cada pedido

Combina lo caro que es el producto con lo fiable que es el paciente. `[TASAPEREZA]` mide los días que el paciente suele tardar en recoger. Si son 3 o menos, se le considera cliente fiel.

El umbral de 3 días está ligado al plazo de devolución de 48 h. Un paciente que suele tardar más deja la caja fuera de plazo antes de que se sepa si va a venir. Si al final no viene, la farmacia ya no la puede devolver.

| Valor | Icono | Caso |
| --- | --- | --- |
| 3 | 🔴 | GLP-1 y paciente que suele tardar: confirmar antes de pedir, porque si no viene la caja no se puede devolver. |
| 2 | 🟡 | GLP-1 y paciente fiel: pedir, con seguimiento. |
| 1 | 🟢 | No es GLP-1: riesgo bajo. |

```dax
Semáforo Acción Pedido =
VAR EsCritico =
    IF ( SELECTEDVALUE ( PRODUCTS[TIPO DE PRODUCTO] ) IN { "Agonistas GLP-1" }, TRUE (), FALSE () )

VAR DiasPereza    = [TASAPEREZA]
VAR EsClienteFiel = IF ( DiasPereza <= 3, TRUE (), FALSE () )

RETURN
    SWITCH (
        TRUE (),
        EsCritico = TRUE () && EsClienteFiel = FALSE (), 3,   -- 🔴 caso 1
        EsCritico = TRUE () && EsClienteFiel = TRUE (),  2,   -- 🟡 caso 2
        EsCritico = FALSE (), 1                               -- 🟢 caso 3
    )
```

En el visual, el valor se muestra como icono con formato condicional.

## Datos y privacidad (RGPD)

- Es un proyecto demo: este repositorio **no contiene datos reales** ni el archivo `.pbix`, solo capturas hechas con un conjunto de datos sintético.
- **Minimización de datos**: el informe nunca muestra el correo completo del paciente. La columna calculada `emailmasked` deja solo las primeras letras (`luc*****@gmail.com`), lo justo para identificarlo. Esto permite además enseñar el informe en demos y capturas sin exponer datos.
- En la captura de SharePoint, la columna de correo está pixelada.

## Estructura del repositorio

```text
.
├── README.md
├── LICENSE
└── docs/
    ├── dashboard.png                          # cuadro de mando
    ├── sharepoint_lista.png                   # lista STOCK_CONTROL
    ├── modelo_datos.png                       # vista de modelo en Power BI
    ├── powerautomate_confirmacion_pedido.png  # flujo 1
    ├── powerautomate_aviso_llegada.png        # flujo 2
    ├── dax_tasa_servicio_inmediato.png        # capturas de las medidas DAX
    ├── dax_progreso.png
    ├── dax_etiqueta_progreso_pereza.png
    └── dax_semaforo.png
```

## Cómo recrearlo

Antes de empezar, revisa los [requisitos](#requisitos).

1. Crea en SharePoint la lista de pedidos con las columnas de `STOCK_CONTROL` (`CODIGO NACIONAL`, `EMAIL`, `ESTADO DEL PEDIDO`, `ORIGEN`, `FECHA REGISTRO`, `FECHA DISPENSACION FUTURA`…) y una lista de productos con `CODIGO NACIONAL` y `TIPO DE PRODUCTO`.
2. Monta los dos flujos de Power Automate siguiendo las tablas de pasos de arriba.
3. En Power BI Desktop, conéctate a las listas con el conector **Lista de SharePoint Online**, añade una tabla calendario, crea las relaciones del modelo y las medidas DAX de este README.
4. Monta los visuales tomando como referencia la captura del cuadro de mando.

## Qué se ve con los datos de ejemplo

Las cifras son del conjunto de datos sintético. No miden el impacto en la farmacia real; enseñan qué preguntas responde el cuadro de mando de un vistazo:

- **48.270 €** en stock refrigerado hoy, con productos de más de 100 € que llevan más de 300 días parados, muy lejos del plazo de devolución.
- En insulinas, el **87,85 %** de los pedidos se sirve desde stock. En GLP-1 solo el **6,06 %**, por debajo del objetivo del 20 %: es el reflejo del desabastecimiento.
- **416 pedidos de GLP-1 frente a 107 de insulinas**: el grupo más caro es también el de más demanda.
- Lista diaria de pacientes que están a punto de terminar su tratamiento, con su demora habitual en recoger y un semáforo que dice qué pedidos confirmar antes de pedir.

## Decisiones y aprendizajes

### Decisiones de negocio

| Decisión | Por qué |
| --- | --- |
| Foco en GLP-1 | Es el grupo donde coinciden precio alto, desabastecimiento, demanda creciente y pacientes que no recogen. |
| Confirmar con el paciente **antes** de pedir | Un refrigerado solo se puede devolver en 48 h. Si el paciente no viene, el coste lo asume la farmacia. |
| Semáforo con umbral de 3 días | Separa a los pacientes que recogen dentro del plazo de devolución de los que dejan la caja fuera de plazo. |
| Correo y no WhatsApp | Coste cero: Outlook viene con Microsoft 365. Muchas farmacias no tienen móvil de empresa para escribir a los pacientes por WhatsApp. |
| Columna `FARMACIA` desde el inicio | Una misma solución sirve para varias farmacias de un grupo. |
| Objetivos configurables | El 20 % de servicio inmediato en GLP-1 es un valor de ejemplo. Cada farmacia pondría el suyo. |

### Decisiones técnicas

| Decisión | Por qué |
| --- | --- |
| Listas de SharePoint como origen | Vienen incluidas en Microsoft 365, sin servidor ni coste extra, y tienen desencadenadores nativos en Power Automate. |
| «Obtener cambios» en el flujo de aviso | Sin este paso, el correo saltaría con cualquier edición del pedido. Con él, solo cuando cambia el estado. |
| Modelo en estrella con tabla de medidas | Separa hechos, dimensiones y lógica. Las medidas quedan agrupadas y fáciles de mantener. |
| Una tabla calendario con relaciones inactivas | Una sola `TIME_TABLE` sirve para analizar registro, dispensación y llegada, activando cada relación con `USERELATIONSHIP`. |
| Enmascarar correos en el modelo | Aplica la minimización de datos del RGPD y permite hacer demos sin exponer datos. |
| Publicado con actualización programada | El cuadro de mando se actualiza solo, sin que nadie tenga que abrir Power BI Desktop. |

### Problemas encontrados

- **Zonas horarias**: SharePoint guarda las fechas en UTC y aparecían desplazadas al pasar a Power BI y al correo. Hay que fijar un criterio de zona horaria común en SharePoint, Power Automate y Power BI.
- **Límite de espera de Power Automate**: una ejecución no puede durar más de 30 días, así que «Retraso hasta» no sirve para fechas más lejanas. Aquí no afecta porque las dispensaciones son mensuales, pero lo tuve en cuenta como límite del diseño.

### Qué haría distinto

- **Separar las listas desde el inicio**: pacientes, productos y pedidos en listas distintas, en vez de una sola lista con todo. El `v2.1` del nombre de la lista viene de esas iteraciones.
- **Facilitar el registro**: los visuales responden a lo que necesita la farmacia. El punto débil es la entrada de datos. Primero probé a que el personal registrara cada pedido desde una app de **Power Apps**, pero en mitad de la jornada nadie tiene tiempo de rellenar un formulario. La adopción depende más de lo fácil que sea registrar un pedido que del cuadro de mando.

## Próximos pasos

- **Registro por foto del ticket**: una app que lea el ticket del pedido (OCR) y rellene la lista sola, para que registrar un pedido cueste segundos.
- **Previsión de demanda**: un modelo de machine learning que anticipe los pedidos de GLP-1 por mes y por paciente, para pedir con antelación en plena época de desabastecimiento.
- **WhatsApp como canal opcional**: solo si la farmacia lo pide y asume el coste del servicio.

## Autor

**Miguel Ángel Leo Acedo**, analista de datos / BI. Cinco años en farmacia.

- LinkedIn: https://www.linkedin.com/in/miguel-leo-acedo
- Otro proyecto: SmartPharma, https://smartpharmai.github.io/

---

## English summary

[Español](#cuadro-de-mando-de-medicamentos-refrigerados-de-alto-valor-en-power-bi) · **English**

**High-value refrigerated medicines dashboard in Power BI** · Power BI + SharePoint + Power Automate

A dashboard to control a pharmacy's high-value cold-chain medicines: GLP-1 agonists, insulins and other refrigerated treatments. It shows how much money is tied up in the fridge, which treatments are about to be dispensed and which patients need to be contacted. Two Power Automate flows ask patients whether they want their next dispensation and notify them when the order arrives. All screenshots use synthetic data. Database columns, measures and screenshots are in Spanish, as in the original solution.

### The problem

A demo project based on five years of working in community pharmacy. GLP-1 agonists (Ozempic, Mounjaro, Wegovy…) combine four problems:

- **High unit price**: every pen sitting in the fridge is a lot of tied-up money.
- **Shortages**: they must be ordered in advance and reserved for the patient.
- **Growing demand**: more patients start treatment every month.
- **Patients who never pick up**: the pharmacy orders for a patient who then doesn't come.

The last one is the most expensive, because **a refrigerated product can only be returned to the wholesaler within 48 hours of delivery** (10 days for other products). After that, the box stays in the fridge until another patient needs it, or until it expires. The pharmacy management software gives no view of refrigerated stock and doesn't notify patients. The goal is to **order only what the patient has confirmed they will pick up, at the right time**.

### The solution

1. Each patient order is logged in a **SharePoint** list (`STOCK_CONTROL`) with the next dispensation date.
2. On that date, **Power Automate** emails the patient asking whether they want the treatment. If they accept, the pharmacy is notified and the order is set to `LISTO PARA PEDIR` (*ready to order*). If not, it is set to `PEDIDO DENEGADO` (*order declined*) and no money is tied up.
3. When the order status changes (for example, when it arrives), a second flow emails the patient with the arrival date and time.
4. **Power BI** reads the list and uses **DAX** to calculate fridge stock, tied-up money, treatment progress and order priority.

```mermaid
flowchart LR
    A["New order in SharePoint"] --> B["Wait until next dispensation date"]
    B --> C{"Does the patient accept?"}
    C -->|Yes| D["Notify the pharmacy<br/>Status = LISTO PARA PEDIR"]
    C -->|No| E["Status = PEDIDO DENEGADO"]
    D --> F["Order arrives: STATUS changes"]
    F --> G["Email the patient with arrival date and time"]
    A -.-> H["Power BI: model and DAX measures"]
    D -.-> H
    E -.-> H
    H --> I["Dashboard"]
```

**Flow 1, order confirmation**: SharePoint *When an item is created* → *Compose* (order data) → *Delay until* the next dispensation date → Outlook *Send email with options* (*Accept / Decline*) → *Condition*: if accepted, email the pharmacy and set `Status = LISTO PARA PEDIR`; if declined, set `Status = PEDIDO DENEGADO`.

**Flow 2, arrival notice**: SharePoint *When an item or file is created or modified* → *Get changes for an item or a file (properties only)* → *Compose* (did `ESTADO` change?) → *Condition*: if the status changed, email the patient with the arrival date and time. Using *Get changes* means the email only goes out on a status change, not on any edit.

### Main list columns

| Column | Purpose |
| --- | --- |
| `Título` | Product and dose (e.g. *SAXENDA 6 MG*). |
| `CODIGO NACIONAL` | National product code; key to the product catalogue. |
| `TIPO DE PRODUCTO` | GLP-1 agonists, insulins, oral antidiabetics or other. |
| `ESTADO DEL PEDIDO` | Order status, updated by the flows. |
| `FECHA PROXIMA DISPENSACION` | When the patient is asked; Flow 1 waits until this date. |
| `FECHA REGISTRO` | Treatment start; base for `%progreso`. |
| `EMAIL` / `TELEFONO` | Patient contact. The report only shows a masked email. |
| `FECHA Y HORA` | Arrival date and time sent to the patient. |
| `IMPORTE` | Order value; base for fridge stock and tied-up money. |
| `FARMACIA` | Pharmacy that logged the order, so several pharmacies in a group can share the solution. |
| `ORIGEN` | `Stock` (served on the spot) or `Pedido` (had to be ordered); base for the immediate service rate. |
| `COMENTARIO` | Free-text notes. |

### What the dashboard shows

| Visual | Question it answers |
| --- | --- |
| Insulin immediacy | What share of insulin orders is served from stock? |
| GLP-1 immediate service vs. monthly target | Am I meeting my target? (20 % is an example, configurable per pharmacy.) |
| Total money in the fridge | How much refrigerated stock do I have today? |
| Orders by group and product | Which group weighs most: GLP-1, insulins, oral antidiabetics or other? |
| Tied-up money (value vs. days in stock) | Which product has been sitting longest and what is it worth? Vertical lines mark the return windows: **48 h** for refrigerated products and **10 days** for the rest. |
| Upcoming dispensations with traffic light | Which orders must be placed or confirmed this week, and which are priority? |
| Top 10 treatment consumed, with delay | Which patients are about to finish their treatment and usually pick up late? |
| Monthly orders | How is demand evolving? |

### Data model and key measures

Star schema: fact table `STOCK_CONTROL` (one row per patient order, with calculated columns such as `emailmasked`), dimensions `PRODUCTS` (by `CODIGO NACIONAL`, with `TIPO DE PRODUCTO`) and `TIME_TABLE` (calendar), plus a `Medidas` table holding the DAX measures. `TIME_TABLE` has three relationships to the fact table: `FECHA REGISTRO` is active; the future dispensation date and the arrival date are inactive and enabled with `USERELATIONSHIP`, so one calendar serves all three dates. Power Query converts types (SharePoint returns dates, amounts and codes as text), drops SharePoint's internal columns and builds `PRODUCTS` from the list.

| Measure | What it does |
| --- | --- |
| `Tasa Servicio Inmediato` | Share of orders served from stock (`ORIGEN = "Stock"`). `+ 0` shows 0 % instead of blank. |
| `%progreso` | Share of time elapsed between order registration and next dispensation. Allows a 20 % tolerance; above 120 % the patient is considered off treatment and drops out of the ranking. |
| `Etiqueta%ypereza` | Data label combining progress and the patient's usual pickup delay, e.g. `97% (+1d demora)`. |
| `Semáforo Acción Pedido` | Order priority: 🔴 (3) GLP-1 and a patient who usually takes more than 3 days to pick up, so confirm before ordering; 🟡 (2) GLP-1 and a reliable patient, so order and follow up; 🟢 (1) not GLP-1, low risk. The 3-day threshold is tied to the 48-hour return window. |

The full DAX code is in the Spanish sections above.

### Requirements

- **Microsoft 365** business plan with SharePoint Online and Exchange Online. The flows only use standard connectors, so no Power Automate premium licence is needed.
- **Power BI Desktop** (free, Windows only) to build the model and report.
- **Power BI Pro** (or Premium Per User / Fabric capacity) to publish, share and schedule refreshes. SharePoint Online is a cloud source, so no gateway is required.
- **Power Apps** (optional), only for the order-entry prototype.
- Permissions to create SharePoint lists, Power Automate flows with SharePoint and Outlook connections, and a Power BI workspace.
- A common time zone across SharePoint, Power Automate and Power BI, and next dispensation dates within 30 days (the maximum a Power Automate run can wait).
- Patient email and consent to receive notifications (GDPR).
- Skills: Power Query, star-schema modelling, intermediate DAX and Power Automate basics.

### How to recreate it

1. Create the SharePoint order list with the `STOCK_CONTROL` columns, and a product list with `CODIGO NACIONAL` and `TIPO DE PRODUCTO`.
2. Build the two Power Automate flows following the steps above.
3. In Power BI Desktop, connect with the **SharePoint Online List** connector, add a calendar table, create the relationships and the DAX measures.
4. Build the visuals using the dashboard screenshot as reference.

### What the sample data shows

Figures come from the synthetic dataset. They don't measure real impact; they show what the dashboard answers at a glance:

- **€48,270** in refrigerated stock today, with products worth over €100 sitting for more than 300 days, far past the return window.
- **87.85 %** of insulin orders are served from stock, but only **6.06 %** of GLP-1 orders, below the 20 % target: a reflection of the shortage.
- **416 GLP-1 orders vs. 107 insulin orders**: the most expensive group also has the highest demand.
- A daily list of patients about to finish their treatment, with their usual pickup delay and a traffic light telling which orders to confirm before ordering.

### Data and privacy (GDPR)

- Demo project: this repository contains **no real data** and no `.pbix` file, only screenshots built on a synthetic dataset.
- **Data minimisation**: the report never shows the patient's full email. The `emailmasked` column keeps only the first letters (`luc*****@gmail.com`). In the SharePoint screenshot, the email column is pixelated.

### Key decisions and lessons learned

- **Confirm with the patient before ordering**: a refrigerated product can only be returned within 48 hours.
- **Email instead of WhatsApp**: zero cost with Microsoft 365, and many pharmacies have no business phone.
- **`FARMACIA` column from day one**: one solution for several pharmacies in a group.
- **SharePoint lists as the source**: included in Microsoft 365, no server, native Power Automate triggers.
- **Problems found**: SharePoint stores dates in UTC, which shifted them in Power BI and emails; Power Automate runs can't last more than 30 days.
- **What I'd do differently**: split patients, products and orders into separate lists from the start, and make order entry easier. A Power Apps form didn't work because staff have no time to fill it in mid-shift; adoption depends more on how easy it is to log an order than on the dashboard.

### Next steps

- **Order logging from a receipt photo** (OCR) so logging takes seconds.
- **Demand forecasting** with machine learning to anticipate GLP-1 orders per month and per patient.
- **WhatsApp as an optional channel**, only if the pharmacy asks for it and covers the cost.

### Author

**Miguel Ángel Leo Acedo**, data / BI analyst with five years in pharmacy. [LinkedIn](https://www.linkedin.com/in/miguel-leo-acedo) · Other project: [SmartPharma](https://smartpharmai.github.io/)

## Licencia / License

[MIT](LICENSE).
